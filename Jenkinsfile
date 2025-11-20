pipeline {
    agent any

    environment {
        AWS_ACCOUNT_ID = '666098475707'
        AWS_REGION = 'us-east-1'
        ECR_REPOSITORY = 'my-web-app'
        DOCKER_IMAGE = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}"
        IMAGE_TAG = "${BUILD_NUMBER}"
        EC2_HOST = 'ec2-44-201-137-203.compute-1.amazonaws.com'
        EC2_USER = 'ec2-user'
        AWS_CREDENTIALS = 'aws-credentials-id'
        EMAIL_RECIPIENTS = 'your-email@example.com'
    }

    stages {
        stage('Checkout') {
            steps {
                echo '==================== Checking out code from GitHub ===================='
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo '==================== Building Application ===================='
                sh 'npm install'
            }
        }

        stage('Test') {
            steps {
                echo '==================== Running Tests ===================='
                sh 'npm test || true'
            }
            post {
                always {
                    script {
                        if (fileExists('test-results')) {
                            junit 'test-results/*.xml'
                        } else {
                            echo 'No test results found'
                        }
                    }
                }
            }
        }

        stage('Code Quality Analysis') {
            steps {
                echo '==================== Running Code Quality Checks ===================='
                sh 'npm run lint || true'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo '==================== Building Docker Image ===================='
                sh """
                    docker build -t ${ECR_REPOSITORY}:${IMAGE_TAG} .
                    docker tag ${ECR_REPOSITORY}:${IMAGE_TAG} ${DOCKER_IMAGE}:${IMAGE_TAG}
                    docker tag ${ECR_REPOSITORY}:${IMAGE_TAG} ${DOCKER_IMAGE}:latest
                """
            }
        }

        stage('Push to ECR') {
            steps {
                echo '==================== Pushing Image to AWS ECR ===================='
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: "${AWS_CREDENTIALS}"]]) {
                    sh """
                        aws ecr get-login-password --region ${AWS_REGION} \
                            | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

                        docker push ${DOCKER_IMAGE}:${IMAGE_TAG}
                        docker push ${DOCKER_IMAGE}:latest
                    """
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                echo '==================== Deploying to EC2 Instance ===================='
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding', credentialsId: "${AWS_CREDENTIALS}"],
                    [$class: 'SSHUserPrivateKeyBinding', credentialsId: 'ec2-ssh-key', keyFileVariable: 'SSH_KEY']
                ]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no -i \$SSH_KEY ${EC2_USER}@${EC2_HOST} << EOF
                            aws ecr get-login-password --region ${AWS_REGION} \
                                | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

                            docker stop my-web-app || true
                            docker rm my-web-app || true

                            docker pull ${DOCKER_IMAGE}:${IMAGE_TAG}

                            docker run -d --name my-web-app -p 80:3000 ${DOCKER_IMAGE}:${IMAGE_TAG}

                            docker image prune -f
EOF
                    """
                }
            }
        }

        stage('Health Check') {
            steps {
                echo '==================== Performing Health Check ===================='
                sh """
                    sleep 15
                    curl -f http://${EC2_HOST}/ || exit 1
                """
            }
        }
    }

    post {
        success {
            echo '==================== Pipeline Succeeded ===================='
            emailext(
                subject: "✔ SUCCESS: ${JOB_NAME} Build #${BUILD_NUMBER}",
                body: """
                <h2>Deployment Successful</h2>
                <p><b>Application URL:</b> http://${EC2_HOST}</p>
                <p><b>Docker Image:</b> ${DOCKER_IMAGE}:${IMAGE_TAG}</p>
                """,
                to: "${EMAIL_RECIPIENTS}",
                mimeType: 'text/html'
            )
        }

        failure {
            echo '==================== Pipeline Failed ===================='
            emailext(
                subject: "❌ FAILURE: ${JOB_NAME} Build #${BUILD_NUMBER}",
                body: """
                <h2>Build Failed</h2>
                <p><a href="${BUILD_URL}console">View Console Output</a></p>
                """,
                to: "${EMAIL_RECIPIENTS}",
                mimeType: 'text/html'
            )
        }

        always {
            cleanWs()
        }
    }
}
