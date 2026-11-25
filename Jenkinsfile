pipeline {
    agent any

    environment {
        DOCKER_IMAGE   = "purshath/kanban-dashboard"
        CONTAINER_PORT = "8080"
        UPSTREAM_CONF  = "/etc/nginx/conf.d/kanban-upstream.conf"
    }

    stages {
        stage('Checkout Source') {
            steps {
                checkout scm
                script {
                    env.GIT_SHA   = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
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

        stage('Deploy Inactive Slot (Blue/Green)') {
            steps {
                script {
                    def blueRunning = sh(script: "docker ps -q -f name=^kanban-blue\$", returnStdout: true).trim()
                    if (blueRunning) {
                        env.TARGET_NAME = "kanban-green"
                        env.TARGET_PORT = "8082"
                        env.OLD_NAME    = "kanban-blue"
                    } else {
                        env.TARGET_NAME = "kanban-blue"
                        env.TARGET_PORT = "8081"
                        env.OLD_NAME    = "kanban-green"
                    }
                    echo "Deploying ${env.DOCKER_IMAGE}:${env.IMAGE_TAG} to ${env.TARGET_NAME} on port ${env.TARGET_PORT} (live: ${env.OLD_NAME})"
                }

                sh '''
                    docker rm -f ${TARGET_NAME} 2>/dev/null || true
                    docker run -d \
                        --name ${TARGET_NAME} \
                        --restart unless-stopped \
                        --memory="512m" \
                        --cpus="0.5" \
                        -p 127.0.0.1:${TARGET_PORT}:${CONTAINER_PORT} \
                        ${DOCKER_IMAGE}:${IMAGE_TAG}
                '''
            }
        }

        stage('Health Check Inactive Slot') {
            steps {
                sh '''
                    echo "Waiting for ${TARGET_NAME} on port ${TARGET_PORT} to become healthy..."
                    for i in $(seq 1 12); do
                        STATUS=$(docker inspect --format='{{.State.Health.Status}}' ${TARGET_NAME} 2>/dev/null || echo "starting")
                        echo "Attempt $i: Health status = $STATUS"
                        if [ "$STATUS" = "healthy" ]; then
                            break
                        fi
                        sleep 5
                    done

                    if [ "$STATUS" != "healthy" ]; then
                        echo "ERROR: ${TARGET_NAME} failed health check (status: $STATUS)"
                        exit 1
                    fi

                    curl -f -I http://127.0.0.1:${TARGET_PORT}
                '''
            }
        }

        stage('Switch Traffic & Clean Up') {
            steps {
                sh '''
                    echo "upstream kanban_backend { server 127.0.0.1:${TARGET_PORT}; }" > ${UPSTREAM_CONF}
                    sudo /usr/sbin/nginx -t
                    sudo /usr/sbin/nginx -s reload

                    echo "Traffic switched to ${TARGET_NAME} (${TARGET_PORT}). Verifying live port 80..."
                    curl -f -I http://127.0.0.1:80

                    echo "Removing old container: ${OLD_NAME}"
                    docker rm -f ${OLD_NAME} 2>/dev/null || true
                '''
            }
        }
    }

    post {
        success {
            echo "Zero-downtime deployment succeeded! Live container: ${env.TARGET_NAME} (${DOCKER_IMAGE}:${env.IMAGE_TAG})"
        }
        failure {
            echo "Deployment failed! Live traffic on ${env.OLD_NAME} was never interrupted. Cleaning up failed slot ${env.TARGET_NAME}..."
            sh '''
                if [ -n "${TARGET_NAME}" ]; then
                    docker rm -f ${TARGET_NAME} 2>/dev/null || true
                fi
            '''
        }
    }
}