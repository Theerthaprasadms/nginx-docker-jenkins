pipeline {
    agent any
    parameters {
        string(name: 'ENVIRONMENT', defaultValue: 'dev', description: 'Environment (dev/prod)')
    }
    environment {
        IMAGE_NAME = "nginx-${params.ENVIRONMENT}"
        AWS_ACCOUNT_ID = "774305596656"
        AWS_REGION = "ap-south-1"
        ECR_REPO = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${IMAGE_NAME}"
    }
    stages {
        stage('Clone Repo') {
            steps {
                git url: 'https://github.com/Theerthaprasadms/nginx-docker-jenkins.git'
            }
        }

        stage('Test Dockerfile') {
            steps {
                script {
                    sh '''
                        docker build -t temp-nginx .
                        docker run -d --name test-nginx -p 80:80 temp-nginx
                        sleep 5
                        STATUS=$(curl -o /dev/null -s -w "%{http_code}" http://localhost:8080)
                        echo "Status Code: $STATUS"
                        if [ "$STATUS" -ne 200 ]; then
                          echo "Health check failed!"
                          docker logs test-nginx
                          exit 1
                        fi
                        docker stop test-nginx && docker rm test-nginx
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t ${IMAGE_NAME} ."
                }
            }
        }

        stage('Login to ECR') {
            steps {
                script {
                    sh '''
                    aws ecr get-login-password --region $AWS_REGION | \
                    docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
                    '''
                }
            }
        }

        stage('Push to ECR') {
            steps {
                script {
                    sh '''
                    aws ecr describe-repositories --repository-names ${IMAGE_NAME} || \
                    aws ecr create-repository --repository-name ${IMAGE_NAME}
                    docker tag ${IMAGE_NAME} ${ECR_REPO}:latest
                    docker push ${ECR_REPO}:latest
                    '''
                }
            }
        }

        stage('Deploy to Slave Node') {
            steps {
                script {
                    sh '''
                    docker rm -f deployed-nginx || true
                    docker run -d --name deployed-nginx -p 80:80 ${ECR_REPO}:latest
                    '''
                }
            }
        }

        stage('Expose IP and Port') {
            steps {
                script {
                    def ip = sh(script: "hostname -I | awk '{print $1}'", returnStdout: true).trim()
                    echo "Nginx running at: http://${ip}:80"
                }
            }
        }
    }
}

