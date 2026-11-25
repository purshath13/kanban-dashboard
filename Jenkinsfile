pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "purshath/kanban-dashboard"
        CONTAINER_NAME = "kanban-app"
        APP_PORT = "80"
        CONTAINER_PORT = "8080"
    }

    stages {
        stage('Checkout Source') {
            steps {
                checkout scm
                script {
                    env.GIT_SHA = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_SHA}"
                    echo "Building version: ${env.DOCKER_IMAGE}:${env.IMAGE_TAG}"
                }
            }
        }

        stage('Docker Build & Tag') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:${env.IMAGE_TAG} ."
            }
        }

        stage('Registry Login & Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${DOCKER_IMAGE}:${IMAGE_TAG}
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                sh '''
                    # Save current working image for rollback if it exists
                    docker inspect --format='{{.Config.Image}}' ${CONTAINER_NAME} > /tmp/prev_image.txt 2>/dev/null || true

                    # Stop and remove existing container
                    docker rm -f ${CONTAINER_NAME} 2>/dev/null || true

                    # Run new container with resource limits
                    docker run -d \
                        --name ${CONTAINER_NAME} \
                        --restart unless-stopped \
                        --memory="512m" \
                        --cpus="0.5" \
                        -p ${APP_PORT}:${CONTAINER_PORT} \
                        ${DOCKER_IMAGE}:${IMAGE_TAG}
                '''
            }
        }

        stage('Health Check & Validation') {
            steps {
                sh '''
                    echo "Checking container is running..."
                    docker ps --filter "name=${CONTAINER_NAME}" --filter "status=running"

                    echo "Waiting for Docker HEALTHCHECK to report healthy..."
                    for i in $(seq 1 12); do
                        STATUS=$(docker inspect --format='{{.State.Health.Status}}' ${CONTAINER_NAME} 2>/dev/null || echo "starting")
                        echo "Attempt $i: Health status = $STATUS"
                        if [ "$STATUS" = "healthy" ]; then
                            break
                        fi
                        sleep 5
                    done

                    if [ "$STATUS" != "healthy" ]; then
                        echo "ERROR: Container failed health check (status: $STATUS)"
                        exit 1
                    fi

                    echo "Verifying HTTP response with curl..."
                    curl -f -I http://127.0.0.1:${APP_PORT}
                '''
            }
        }
    }

    post {
        success {
            echo "Deployment succeeded for ${DOCKER_IMAGE}:${env.IMAGE_TAG}!"
        }
        failure {
            echo "Deployment failed! Triggering rollback..."
            sh '''
                if [ -s /tmp/prev_image.txt ]; then
                    PREV_IMAGE=$(cat /tmp/prev_image.txt)
                    echo "Rolling back to previous working image: $PREV_IMAGE"
                    docker rm -f ${CONTAINER_NAME} 2>/dev/null || true
                    docker run -d \
                        --name ${CONTAINER_NAME} \
                        --restart unless-stopped \
                        --memory="512m" \
                        --cpus="0.5" \
                        -p ${APP_PORT}:${CONTAINER_PORT} \
                        "$PREV_IMAGE"
                else
                    echo "No previous image found to roll back to."
                fi
            '''
        }
    }
}