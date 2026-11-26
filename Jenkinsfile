def runOnDeployServer(String command) {
    withCredentials([sshUserPrivateKey(
        credentialsId: env.DEPLOYMENT_SSH_CREDENTIAL_ID,
        keyFileVariable: 'SSH_KEY',
        usernameVariable: 'SSH_USER'
    )]) {
        sh "ssh -i \$SSH_KEY -o StrictHostKeyChecking=accept-new " +
           "\$SSH_USER@${env.DEPLOYMENT_SERVER_IP} '${command}'"
    }
}

pipeline {
    agent none

    options {
        skipDefaultCheckout(true)
    }

    environment {
        DOCKERHUB_USERNAME = 'amayaajayan'
        DOCKERHUB_REPOSITORY = 'company-website'
        DOCKERHUB_CREDENTIAL_ID = 'dockerhub-login'

        DEPLOYMENT_SERVER_IP = '13.203.217.14'
        DEPLOYMENT_SSH_CREDENTIAL_ID = 'deployment-server-ssh'

        GITHUB_REPOSITORY_URL = 'https://github.com/amayaajayan23-stack/company-website.git'

        IMAGE_NAME = "${DOCKERHUB_USERNAME}/${DOCKERHUB_REPOSITORY}"
        CONTAINER_NAME = 'company-website'
    }

    stages {

        stage('Build Server Tasks') {
            agent {
                label 'build-server'
            }

            stages {

                stage('Checkout') {
                    steps {
                        git branch: 'main',
                            url: "${GITHUB_REPOSITORY_URL}"
                    }
                }

                stage('Docker Image Build') {
                    steps {
                        sh 'docker build -t $IMAGE_NAME:$BUILD_NUMBER -t $IMAGE_NAME:latest .'
                    }
                }

                stage('Docker Hub Login') {
                    steps {
                        withCredentials([usernamePassword(
                            credentialsId: env.DOCKERHUB_CREDENTIAL_ID,
                            usernameVariable: 'DH_USER',
                            passwordVariable: 'DH_PASS'
                        )]) {
                            sh 'echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin'
                        }
                    }
                }

                stage('Docker Image Push') {
                    steps {
                        sh 'docker push $IMAGE_NAME:$BUILD_NUMBER'
                        sh 'docker push $IMAGE_NAME:latest'
                    }

                    post {
                        always {
                            sh 'docker logout || true'
                        }
                    }
                }
            }
        }

        stage('Deployment Server Tasks') {
            agent {
                label 'built-in'
            }

            stages {

                stage('Connect to Deployment Server') {
                    steps {
                        script {
                            runOnDeployServer(
                                'hostname && docker --version'
                            )
                        }
                    }
                }

                stage('Pull Latest Image') {
                    steps {
                        script {
                            runOnDeployServer(
                                "docker pull ${IMAGE_NAME}:latest"
                            )
                        }
                    }
                }

                stage('Stop Old Container') {
                    steps {
                        script {
                            runOnDeployServer(
                                "docker stop ${CONTAINER_NAME} || true"
                            )

                            runOnDeployServer(
                                "docker rm ${CONTAINER_NAME} || true"
                            )
                        }
                    }
                }

                stage('Run New Container') {
                    steps {
                        script {
                            runOnDeployServer(
                                "docker run -d --name ${CONTAINER_NAME} " +
                                "--restart unless-stopped -p 9090:80 ${IMAGE_NAME}:latest"
                            )
                        }
                    }
                }

                stage('Verify Deployment') {
                    steps {
                        script {
                            runOnDeployServer(
                                "docker ps"
                            )
                        }
                    }
                }
            }
        }
    }
}
