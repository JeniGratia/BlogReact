
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        IMAGE_TAG = "v${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Backend Dependencies and Tests') {
            steps {
                dir('backend') {
                    sh 'npm ci'
                    sh 'npm test --if-present -- --passWithNoTests'
                }
            }
        }

        stage('Frontend Dependencies and Build') {
            steps {
                dir('frontend') {
                    sh 'npm ci'
                    sh 'npm run build'
                }
            }
        }


        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    withCredentials([
                        string(
                            credentialsId: 'sonar-token',
                            variable: 'SONAR_TOKEN'
                        )
                    ]) {
                        sh '''
                            mkdir -p "$WORKSPACE/.scannerwork"
                            chmod -R 777 "$WORKSPACE/.scannerwork"

                            docker run --rm --network host \
                              -e SONAR_HOST_URL="$SONAR_HOST_URL" \
                              -e SONAR_TOKEN="$SONAR_TOKEN" \
                              -v "$WORKSPACE:/usr/src" \
                              sonarsource/sonar-scanner-cli \
                              -Dsonar.working.directory=/tmp/sonar-work \
                              -Dsonar.scanner.metadataFilePath=/usr/src/report-task.txt
                        '''
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    docker build \
                      -t blogapp-backend:${IMAGE_TAG} ./backend

                    docker build \
                      -t blogapp-frontend:${IMAGE_TAG} ./frontend

                    docker tag blogapp-backend:${IMAGE_TAG} blogapp-backend:latest
                    docker tag blogapp-frontend:${IMAGE_TAG} blogapp-frontend:latest
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                    trivy image --skip-db-update \
                      --severity HIGH,CRITICAL \
                      blogapp-backend:${IMAGE_TAG}

                    trivy image --skip-db-update \
                      --severity HIGH,CRITICAL \
                      blogapp-frontend:${IMAGE_TAG}
                '''
            }
        }

        stage('Validate Kubernetes Access') {
            steps {
                sh '''
                    kubectl get nodes
                    kubectl get pods -n blogapp
                '''
            }
        }
    }

    post {
        always {
            echo 'BlogReact CI/CD pipeline finished.'
        }

        success {
            echo 'All configured pipeline stages completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Check the stage console output for the exact error.'
        }
    }
}

