pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        ECR_REGISTRY = '862770535694.dkr.ecr.ap-south-1.amazonaws.com'
        SERVICE_NAME = 'patient-portal'
        IMAGE_TAG = "${BUILD_NUMBER}"
        IMAGE_NAME = '862770535694.dkr.ecr.ap-south-1.amazonaws.com/portal'
        SONARQUBE_SERVER = 'SonarQube'
        SONARQUBE_TOKEN = credentials('sonar-token-portal')
    }

    options {
        timestamps()
        timeout(time: 45, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    echo "Code checked out successfully"
                    sh 'git log --oneline -1'
                }
            }
        }

        stage('Build') {
            steps {
                script {
                    echo "Building Patient Portal..."
                    sh '''
                        npm install
                        npm run lint
                        npm run build
                    '''
                }
            }
        }

        stage('Test & Security Scan') {
            parallel {
                stage('Unit Tests & SonarQube') {
                    steps {
                        script {
                            echo "Running unit tests and SonarQube analysis..."
                            sh '''
                                npm run test:coverage || true

                                docker run --rm \
                                    --network host \
                                    -v ${WORKSPACE}:/usr/src \
                                    -e SONAR_TOKEN=${SONARQUBE_TOKEN} \
                                    sonarsource/sonar-scanner-cli:latest \
                                    -Dsonar.projectKey=${SERVICE_NAME} \
                                    -Dsonar.sources=/usr/src/src \
                                    -Dsonar.exclusions=/usr/src/node_modules/**,/usr/src/dist/** \
                                    -Dsonar.host.url=http://localhost:9000
                            '''
                        }
                    }
                }

                stage('Gitleaks Scan') {
                    steps {
                        script {
                            echo "Running Gitleaks scan for secrets detection..."
                            sh '''
                                if ! command -v gitleaks > /dev/null 2>&1; then
                                    docker run --rm -v ${WORKSPACE}:/path \
                                        zricethezav/gitleaks:latest version
                                fi

                                docker run --rm -v ${WORKSPACE}:/path \
                                    zricethezav/gitleaks:latest detect \
                                    --source /path \
                                    --verbose \
                                    --report-path /path/gitleaks-report.json \
                                    --exit-code 1 || true

                                echo "Gitleaks scan completed"
                            '''
                        }
                    }
                }
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    echo "Building Docker image..."
                    sh '''
                        docker build -t ${SERVICE_NAME}:${IMAGE_TAG} .
                        docker tag ${SERVICE_NAME}:${IMAGE_TAG} ${SERVICE_NAME}:latest
                        echo "Docker image built: ${SERVICE_NAME}:${IMAGE_TAG}"
                    '''
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                script {
                    echo "Scanning Docker image with Trivy..."
                    sh '''
                        docker run --rm \
                            -v /var/run/docker.sock:/var/run/docker.sock \
                            -v ${WORKSPACE}:/workspace \
                            aquasec/trivy:latest image \
                            --severity CRITICAL,HIGH \
                            --exit-code 0 \
                            --no-progress \
                            --format json \
                            --output /workspace/trivy-report.json \
                            ${SERVICE_NAME}:${IMAGE_TAG}

                        echo "Trivy scan completed"
                    '''
                }
            }
        }

        stage('ECR Login & Push') {
            steps {
                echo "Pushing Docker image to ECR..."
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-credentials'
                ]]) {
                    sh '''
                        export AWS_DEFAULT_REGION=${AWS_REGION}

                        aws ecr get-login-password --region ${AWS_REGION} | \
                            docker login --username AWS --password-stdin ${ECR_REGISTRY}

                        docker tag ${SERVICE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:${IMAGE_TAG}
                        docker tag ${SERVICE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest

                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${IMAGE_NAME}:latest

                        echo "Image pushed to ECR:"
                        echo "  - ${IMAGE_NAME}:${IMAGE_TAG}"
                        echo "  - ${IMAGE_NAME}:latest"
                    '''
                }
            }
        }
    }

    post {
        always {
            script {
                echo "Pipeline completed"
                archiveArtifacts artifacts: '**/coverage/**,gitleaks-report.json,trivy-report.json',
                                  allowEmptyArchive: true
            }
        }
        success {
            script {
                echo "Pipeline succeeded - Patient Portal image is ready"
            }
        }
        failure {
            script {
                echo "Pipeline failed - Check logs for details"
            }
        }
    }
}
