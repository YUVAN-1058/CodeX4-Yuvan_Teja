pipeline {
    agent any

    environment {
        IMAGE_NAME = 'novabank-transfer:jenkins'
        CONTAINER_NAME = 'novabank-transfer-jenkins'
        APP_PORT = '8082'
    }

    stages {

        stage('Git Checkout') {
            steps {
                checkout scm
                sh 'git status'
            }
        }

        stage('JUnit Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit allowEmptyResults: true,
                          testResults: 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Maven Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Gitleaks') {
            steps {
                sh '''
                    rm -rf target
                    gitleaks dir \
                      --redact \
                      --report-format json \
                      --report-path gitleaks-report.json \
                      --exit-code 0 \
                      .
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'gitleaks-report.json',
                                     allowEmptyArchive: true
                }
            }
        }

        stage('Docker Image Build') {
            steps {
                sh '''
                    mvn package -DskipTests
                    docker build -t ${IMAGE_NAME} .
                '''
            }
        }

        stage('Trivy Scan') {
            steps {
                sh '''
                    trivy image \
                      --severity HIGH,CRITICAL \
                      --format json \
                      --output trivy-report.json \
                      --exit-code 0 \
                      ${IMAGE_NAME}
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.json',
                                     allowEmptyArchive: true
                }
            }
        }

        stage('Docker Run') {
            steps {
                sh '''
                    docker rm -f ${CONTAINER_NAME} 2>/dev/null || true

                    docker run -d \
                      --name ${CONTAINER_NAME} \
                      -p ${APP_PORT}:${APP_PORT} \
                      ${IMAGE_NAME}

                    echo "Waiting for application..."
                    sleep 15

                    echo "Health check:"
                    curl --fail http://localhost:${APP_PORT}/actuator/health

                    echo
                    echo "Container status:"
                    docker ps --filter "name=${CONTAINER_NAME}"
                '''
            }
        }
    }

    post {
        always {
            echo 'Secure pipeline execution completed.'
        }

        success {
            echo 'Pipeline completed successfully.'
        }

        unstable {
            echo 'Pipeline completed with security findings recorded in reports.'
        }

        failure {
            echo 'Pipeline failed. Review the stage console output.'
        }
    }
}
