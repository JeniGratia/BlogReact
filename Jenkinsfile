

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
                            docker run --rm --network host \
                              --user "$(id -u):$(id -g)" \
                              -e HOME=/tmp \
                              -e SONAR_USER_HOME=/tmp/sonar-home \
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
                withSonarQubeEnv('SonarQube') {
                    withCredentials([
                        string(
                            credentialsId: 'sonar-token',
                            variable: 'SONAR_TOKEN'
                        )
                    ]) {
                        sh '''
                            set -eu

                            if [ ! -f report-task.txt ]; then
                                echo "ERROR: report-task.txt missing"
                                exit 1
                            fi

                            TASK_ID=$(sed -n 's/^ceTaskId=//p' report-task.txt)

                            if [ -z "$TASK_ID" ]; then
                                echo "ERROR: SonarQube task ID missing"
                                exit 1
                            fi

                            ANALYSIS_ID=""

                            for i in $(seq 1 60); do
                                RESPONSE=$(curl -fsS \
                                    -u "$SONAR_TOKEN:" \
                                    "$SONAR_HOST_URL/api/ce/task?id=$TASK_ID")

                                STATUS=$(printf '%s' "$RESPONSE" |
                                    python3 -c 'import json,sys; print(json.load(sys.stdin)["task"]["status"])')

                                echo "SonarQube task status: $STATUS"

                                if [ "$STATUS" = "SUCCESS" ]; then
                                    ANALYSIS_ID=$(printf '%s' "$RESPONSE" |
                                        python3 -c 'import json,sys; print(json.load(sys.stdin)["task"].get("analysisId",""))')
                                    break
                                fi

                                if [ "$STATUS" = "FAILED" ] || [ "$STATUS" = "CANCELED" ]; then
                                    echo "ERROR: SonarQube processing failed"
                                    exit 1
                                fi

                                sleep 5
                            done

                            if [ -z "$ANALYSIS_ID" ]; then
                                echo "ERROR: SonarQube processing timed out"
                                exit 1
                            fi

                            GATE=$(curl -fsS \
                                -u "$SONAR_TOKEN:" \
                                "$SONAR_HOST_URL/api/qualitygates/project_status?analysisId=$ANALYSIS_ID" |
                                python3 -c 'import json,sys; print(json.load(sys.stdin)["projectStatus"]["status"])')

                            echo "Quality Gate: $GATE"

                            if [ "$GATE" != "OK" ]; then
                                echo "ERROR: Quality Gate did not pass"
                                exit 1
                            fi

                            echo "Quality Gate passed successfully"
                        '''
                    }
                }
            }
        }
        stage('Build Docker Images') {
            steps {
                sh '''
                    docker build -t blogapp-backend:${IMAGE_TAG} ./backend
                    docker build -t blogapp-frontend:${IMAGE_TAG} ./frontend

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
            echo 'Pipeline failed. Check Console Output for the exact error.'
        }
    }
}
