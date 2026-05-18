pipeline {
    agent any

    environment {
        AWS_ACCOUNT_ID = '571573792371'
        AWS_REGION     = 'ap-south-1'
        ECR_REGISTRY   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        SNS_TOPIC_ARN  = "arn:aws:sns:ap-south-1:571573792371:streaming-app-notifications"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/sannnn1234/StreamingApp.git'
            }
        }

        stage('Login to ECR') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-ecr-creds'
                ]]) {
                    sh '''
                        aws ecr get-login-password --region $AWS_REGION \
                        | docker login --username AWS --password-stdin $ECR_REGISTRY
                    '''
                }
            }
        }

        stage('Build Docker Images') {
            parallel {

                stage('Build Frontend') {
                    steps {
                        dir('frontend') {
                            sh 'docker build -t streaming-frontend .'
                        }
                    }
                }

                stage('Build Auth Service') {
                    steps {
                        dir('backend/authService') {
                            sh 'docker build -t streaming-auth .'
                        }
                    }
                }

                stage('Build Streaming Service') {
                    steps {
                        sh 'docker build -t streaming-streaming -f backend/streamingService/Dockerfile backend'
                    }
                }

                stage('Build Admin Service') {
                    steps {
                        sh 'docker build -t streaming-admin -f backend/adminService/Dockerfile backend'
                    }
                }

                stage('Build Chat Service') {
                    steps {
                        sh 'docker build -t streaming-chat -f backend/chatService/Dockerfile backend'
                    }
                }
            }
        }

        stage('Tag & Push to ECR') {
            steps {
                sh '''
                    docker tag streaming-frontend:latest   $ECR_REGISTRY/streaming-frontend:latest
                    docker tag streaming-auth:latest       $ECR_REGISTRY/streaming-auth:latest
                    docker tag streaming-streaming:latest  $ECR_REGISTRY/streaming-streaming:latest
                    docker tag streaming-admin:latest      $ECR_REGISTRY/streaming-admin:latest
                    docker tag streaming-chat:latest       $ECR_REGISTRY/streaming-chat:latest

                    docker push $ECR_REGISTRY/streaming-frontend:latest
                    docker push $ECR_REGISTRY/streaming-auth:latest
                    docker push $ECR_REGISTRY/streaming-streaming:latest
                    docker push $ECR_REGISTRY/streaming-admin:latest
                    docker push $ECR_REGISTRY/streaming-chat:latest
                '''
            }
        }
    }

    post {
        success {
            withCredentials([[
                $class: 'AmazonWebServicesCredentialsBinding',
                credentialsId: 'aws-ecr-creds'
            ]]) {
                sh '''
                    aws sns publish \
                    --topic-arn $SNS_TOPIC_ARN \
                    --subject "BUILD SUCCESS - StreamingApp" \
                    --message "Jenkins build #${BUILD_NUMBER} completed successfully. All images pushed to ECR." \
                    --region $AWS_REGION
                '''
            }
        }

        failure {
            withCredentials([[
                $class: 'AmazonWebServicesCredentialsBinding',
                credentialsId: 'aws-ecr-creds'
            ]]) {
                sh '''
                    aws sns publish \
                    --topic-arn $SNS_TOPIC_ARN \
                    --subject "BUILD FAILED - StreamingApp" \
                    --message "Jenkins build #${BUILD_NUMBER} failed. Check Jenkins console logs." \
                    --region $AWS_REGION
                '''
            }
        }
    }
}