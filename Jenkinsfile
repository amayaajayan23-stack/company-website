
pipeline {
    agent none

    options {
        timestamps()
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
    }

    environment {
        // Docker Hub
        DOCKERHUB_USERNAME = 'amayaajayan'
        DOCKERHUB_CREDENTIAL_ID = 'jenkins'
        IMAGE_NAME = 'amayaajayan/company-website'

        // GitHub
        GITHUB_REPOSITORY_URL = 'https://github.com/minhajcp2003-svg/company-website.git'
        GITHUB_BRANCH = 'main'

        // Application
        APP_NAME = 'company-website'
        HOST_PORT = '9090'
        CONTAINER_PORT = '80'
    }

    stages {

        // STAGE 1: CHECKOUT
        stage('Checkout') {
            agent {
                label 'build-server'
            }

            steps {
                echo 'Checking out source code...'

                git branch: "${GITHUB_BRANCH}",
                    url: "${GITHUB_REPOSITORY_URL}"
            }
        }

        // STAGE 2: BUILD DOCKER IMAGE
        stage('Build Docker Image') {
            agent {
                label 'build-server'
            }

            steps {
                script {
                    env.DEPLOY_IMAGE =
                        "${env.IMAGE_NAME}:${env.BUILD_NUMBER}"
                }

                echo "Building image: ${env.DEPLOY_IMAGE}"

                sh '''
                    set -eu

                    docker build --pull \
                        -t "$DEPLOY_IMAGE" .

                    docker tag "$DEPLOY_IMAGE" \
                        "$IMAGE_NAME:latest"
                '''
            }
        }

        // STAGE 3: PUSH TO DOCKER HUB
        stage('Push to Docker Hub') {
            agent {
                label 'build-server'
            }

            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${DOCKERHUB_CREDENTIAL_ID}",
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        set +x

                        printf '%s' "$DOCKER_PASSWORD" |
                            docker login \
                            --username "$DOCKER_USER" \
                            --password-stdin

                        docker push "$DEPLOY_IMAGE"

                        docker push "$IMAGE_NAME:latest"

                        docker logout
                    '''
                }

                echo 'Docker image pushed successfully!'
            }
        }

        // STAGE 4: DEPLOY
        stage('Deploy to Deployment Server') {
            agent {
                label 'deploy-server'
            }

            steps {
                echo "Deploying ${env.DEPLOY_IMAGE}..."

                sh '''
                    set -eu

                    echo "Pulling Docker image..."

                    docker pull "$IMAGE_NAME:$BUILD_NUMBER"

                    echo "Removing old container..."

                    docker rm -f "$APP_NAME" || true

                    echo "Starting new container..."

                    docker run -d \
                        --name "$APP_NAME" \
                        --restart unless-stopped \
                        -p "$HOST_PORT:$CONTAINER_PORT" \
                        "$IMAGE_NAME:$BUILD_NUMBER"

                    echo "Checking container status..."

                    sleep 5

                    docker ps --filter "name=$APP_NAME"

                    if [ "$(docker inspect -f '{{.State.Running}}' "$APP_NAME")" != "true" ]; then
                        echo "Container failed to start!"
                        docker logs "$APP_NAME" || true
                        exit 1
                    fi

                    echo "Deployment successful!"
                '''
            }
        }
    }

    post {
        success {
            echo """
                ====================================
                SUCCESS: WEBSITE DEPLOYED!
                Image: ${env.IMAGE_NAME}:${env.BUILD_NUMBER}
                Port: 9090
                ====================================
            """
        }

        failure {
            echo 'PIPELINE FAILED! Check Console Output.'
        }

        always {
            echo 'Jenkins pipeline execution completed.'
        }
    }
}
