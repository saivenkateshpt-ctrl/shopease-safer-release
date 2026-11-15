// LAB 4: + automatic rollback (final pipeline)
def deploy(String name, int port, String image, boolean bad) {
    def version = image.tokenize(':')[1]
    sh "docker rm -f ${name} || true"
    sh "docker run -d --name ${name} --network shopease-net -p ${port}:5000 --memory 256m --cap-drop ALL --security-opt no-new-privileges -e APP_VERSION=${version} -e BREAK_HEALTH=${bad} ${image}"
}

def smoke(String host) {
    withEnv(["HOST=${host}"]) {
        sh '''
            for i in 1 2 3 4 5 6 7 8 9 10; do
                curl -fsS http://$HOST:5000/health && echo && exit 0
                echo "waiting for $HOST ($i/10)..."; sleep 3
            done
            echo "health check FAILED for $HOST"; exit 1
        '''
    }
}

def notifyTeam(String status) {
    def msg = "ShopEase order-service | ${status} | build #${env.BUILD_NUMBER} | ${env.BUILD_URL}"
    writeFile file: 'notify.json', text: '{"text": "' + msg + '"}'
    withCredentials([string(credentialsId: 'notify-token', variable: 'NOTIFY_TOKEN')]) {
        sh 'curl -sS -X POST -H "Content-Type: application/json" -H "Authorization: Bearer $NOTIFY_TOKEN" --data @notify.json http://notify-receiver:9000/ || true'
    }
}

pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 20, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    parameters {
        booleanParam(name: 'SIMULATE_BAD_RELEASE', defaultValue: false, description: 'Tick to simulate a release that passes all tests but is unhealthy in production')
    }

    environment {
        IMAGE = "shopease/order-service:${BUILD_NUMBER}"
    }

    stages {
        stage('Quality Gate') {
            steps { sh 'docker build --progress=plain --target test .' }
        }
        stage('Build Image') {
            steps { sh 'docker build --target runtime -t $IMAGE .' }
        }
        stage('Deploy to Staging') {
            steps { script { deploy('shopease-staging', 8081, env.IMAGE, false) } }
        }
        stage('Smoke Test Staging') {
            steps { script { smoke('shopease-staging') } }
        }
        stage('Approval') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    input message: 'Deploy build #' + env.BUILD_NUMBER + ' to PRODUCTION?', ok: 'Deploy', submitter: 'admin'
                }
            }
        }
        stage('Deploy to Production') {
            steps {
                script {
                    def previous = sh(script: "docker inspect --format '{{.Config.Image}}' shopease-prod 2>/dev/null || true", returnStdout: true).trim()
                    env.PREVIOUS_IMAGE = previous ?: 'none'
                    echo 'Production currently runs: ' + env.PREVIOUS_IMAGE
                    deploy('shopease-prod', 8082, env.IMAGE, params.SIMULATE_BAD_RELEASE == true)
                }
            }
        }
        stage('Verify Production') {
            steps {
                script {
                    try {
                        smoke('shopease-prod')
                    } catch (err) {
                        echo 'Production is UNHEALTHY - rolling back automatically'
                        if (env.PREVIOUS_IMAGE == 'none') {
                            error('Nothing to roll back to. Escalate to the on-call engineer now.')
                        }
                        deploy('shopease-prod', 8082, env.PREVIOUS_IMAGE, false)
                        smoke('shopease-prod')
                        env.ROLLED_BACK = 'true'
                        error('Release failed. Production rolled back to ' + env.PREVIOUS_IMAGE)
                    }
                }
            }
        }
    }

    post {
        always {
            sh 'echo "build=$BUILD_NUMBER image=$IMAGE commit=$(git rev-parse --short HEAD)" > build-report.txt'
            archiveArtifacts artifacts: 'build-report.txt', allowEmptyArchive: true
            sh 'docker image prune -f || true'
        }
        success { script { notifyTeam('SUCCESS') } }
        failure {
            script {
                notifyTeam(env.ROLLED_BACK == 'true' ? 'FAILED - production rolled back to previous version' : 'FAILED')
            }
        }
        aborted { script { notifyTeam('ABORTED - not approved') } }
        cleanup { cleanWs() }
    }
}