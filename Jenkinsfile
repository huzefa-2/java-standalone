pipeline {

    agent any

    environment {
        AWS_REGION = "us-east-1"
        ECR_REGISTRY = "177203142049.dkr.ecr.us-east-1.amazonaws.com"
        ECR_REPOSITORY = "my-app"
        IMAGE_NAME = "177203142049.dkr.ecr.us-east-1.amazonaws.com/my-app"
        CONTAINER_NAME = "huzef-java"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'master',
                    url: 'https://github.com/huzefa-2/java-standalone.git'
            }
        }

        stage('Build') {
            steps {
                sh '''
                    mvn clean package
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build -t java-app:latest .
                '''
            }
        }

        stage('Push to Amazon ECR') {
            steps {
                sh '''
                    aws ecr get-login-password --region $AWS_REGION | \
                    docker login \
                    --username AWS \
                    --password-stdin $ECR_REGISTRY

                    docker tag java-app:latest $IMAGE_NAME:latest

                    docker push $IMAGE_NAME:latest
                '''
            }
        }

        stage('Deploy Docker Container') {
            steps {
                sh '''
                    docker pull $IMAGE_NAME:latest

                    docker stop $CONTAINER_NAME || true
                    docker rm $CONTAINER_NAME || true

                    docker run -d \
                        --name $CONTAINER_NAME \
                        -p 8081:8080 \
                        $IMAGE_NAME:latest
                '''
            }
        }
    }

    post {
        success {
            echo 'CI/CD Pipeline completed successfully!'
        }

        failure {
            echo 'CI/CD Pipeline failed. Check the console output.'
        }
    }
}
