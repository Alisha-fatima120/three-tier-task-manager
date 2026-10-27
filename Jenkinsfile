//Testgithub automated testing CI-CD
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        FRONTEND_IMAGE = 'alishafatima797/three-tier-frontend'
        BACKEND_IMAGE = 'alishafatima797/three-tier-backend'
        DB_USER = 'todouser'
        DB_NAME = 'todolist'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm

                script {
                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()
                }

                echo "Building image tag: ${IMAGE_TAG}"
            }
        }

        stage('Validate Compose') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'mysql-db-password',
                        variable: 'DB_PASSWORD'
                    )
                ]) {
                    sh '''
                        set +x
                        export DB_USER="${DB_USER}"
                        export DB_NAME="${DB_NAME}"
                        export IMAGE_TAG="${IMAGE_TAG}"
                        docker compose -p todolist config --quiet
                    '''
                }
            }
        }

        stage('Build Images') {
            steps {
                sh '''
                    set -e
                    docker build -t ${FRONTEND_IMAGE}:${IMAGE_TAG} ./client
                    docker build -t ${BACKEND_IMAGE}:${IMAGE_TAG} ./server
                '''
            }
        }

        stage('Push Images to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_TOKEN'
                    )
                ]) {
                    sh '''
                        set +x
                        set -e

                        echo "$DOCKERHUB_TOKEN" |
                            docker login \
                                --username "$DOCKERHUB_USER" \
                                --password-stdin

                        docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}
                        docker push ${BACKEND_IMAGE}:${IMAGE_TAG}

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy Application') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'mysql-db-password',
                        variable: 'DB_PASSWORD'
                    )
                ]) {
                    sh '''
                        set +x
                        set -e

                        docker compose -p todolist pull backend frontend

                        docker compose -p todolist up \
                            -d \
                            --no-build \
                            --remove-orphans
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    set -e

                    echo "Checking container status..."
                    docker compose -p todolist ps

                    echo "Checking MySQL container..."
                    test "$(docker inspect --format='{{.State.Running}}' todolist-mysql)" = "true"

                    echo "Checking backend container..."
                    test "$(docker inspect --format='{{.State.Running}}' todolist-backend)" = "true"

                    echo "Checking frontend container..."
                    test "$(docker inspect --format='{{.State.Running}}' todolist-frontend)" = "true"

                    echo "Checking frontend HTTP response..."
                    curl --fail --silent --show-error \
                        --retry 10 \
                        --retry-delay 5 \
                        --retry-connrefused \
                        http://localhost/

                    echo
                    echo "Deployment verification completed."
                '''
            }
        }
    }

    post {
        success {
            echo "CI/CD completed successfully. Image tag: ${IMAGE_TAG}"
        }

        failure {
            echo 'CI/CD failed. Review the console output for the failed stage.'
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}
// Webhook trigger test 2
