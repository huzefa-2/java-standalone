pipeline {
    agent none

    environment {
        AWS_REGION     = 'us-east-1'
        AWS_ACCOUNT_ID = '177203142049'

        ECR_REGISTRY   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        ECR_REPO       = "${ECR_REGISTRY}/java-webapp"
        IMAGE_TAG      = "v${BUILD_NUMBER}"

        SONAR_HOST_URL = 'http://54.163.200.53:9000'
    }

    stages {

        stage('Checkout & Maven Build') {
            agent { label 'sonarqube' }

            steps {
                deleteDir()

                git branch: 'master',
                    url: 'https://github.com/huzefa-2/java-standalone.git'

                sh '''
                    echo "===== CHECKOUT ====="
                    ls -la

                    echo "===== JAVA ====="
                    java -version

                    echo "===== MAVEN ====="
                    mvn -version

                    echo "===== MAVEN BUILD ====="
                    mvn clean package -DskipTests
                '''

                stash name: 'source-code',
                      includes: '**/*',
                      useDefaultExcludes: false
            }
        }

        stage('SonarQube Analysis') {
            agent { label 'sonarqube' }

            steps {
                withSonarQubeEnv('SonarQube') {
                    withCredentials([
                        string(
                            credentialsId: 'sonar-token',
                            variable: 'SONAR_TOKEN'
                        )
                    ]) {
                        sh '''
                            echo "===== SONARQUBE ANALYSIS ====="

                            /opt/sonar-scanner/bin/sonar-scanner \
                              -Dsonar.projectKey=docker-sample-java-webapp \
                              -Dsonar.sources=src \
                              -Dsonar.java.binaries=target/classes \
                              -Dsonar.host.url=$SONAR_HOST_URL \
                              -Dsonar.token=$SONAR_TOKEN
                        '''
                    }
                }
            }
        }

        stage('Quality Gate') {
            agent none

            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            agent { label 'docker-worker' }

            steps {
                deleteDir()

                unstash 'source-code'

                sh '''
                    echo "===== DOCKER BUILD ====="

                    docker build \
                      -t ${ECR_REPO}:${IMAGE_TAG} .

                    echo "===== SAVE IMAGE ====="

                    docker save \
                      ${ECR_REPO}:${IMAGE_TAG} \
                      -o image.tar
                '''

                stash name: 'docker-image',
                      includes: 'image.tar',
                      useDefaultExcludes: false
            }
        }

        stage('Trivy Scan') {
            agent { label 'trivy' }

            steps {
                deleteDir()

                unstash 'docker-image'

                sh '''
                    echo "===== TRIVY SCAN ====="

                    mkdir -p trivy-tmp

                    export TMPDIR=$PWD/trivy-tmp
                    export TRIVY_CACHE_DIR=$PWD/trivy-tmp

                    trivy image \
                      --input image.tar \
                      --severity HIGH,CRITICAL \
                      --exit-code 1
                '''
            }
        }

        stage('Push Image to ECR') {
            agent { label 'docker-worker' }

            steps {
                deleteDir()

                unstash 'docker-image'

                sh '''
                    echo "===== LOAD IMAGE ====="

                    docker load -i image.tar

                    echo "===== AWS IDENTITY ====="

                    aws sts get-caller-identity

                    echo "===== ECR LOGIN ====="

                    aws ecr get-login-password \
                      --region ${AWS_REGION} | \
                    docker login \
                      --username AWS \
                      --password-stdin ${ECR_REGISTRY}

                    echo "===== PUSH IMAGE ====="

                    docker push ${ECR_REPO}:${IMAGE_TAG}
                '''
            }
        }

        stage('Pull Image from ECR') {
            agent { label 'docker-worker' }

            steps {
                sh '''
                    echo "===== ECR LOGIN ====="

                    aws ecr get-login-password \
                      --region ${AWS_REGION} | \
                    docker login \
                      --username AWS \
                      --password-stdin ${ECR_REGISTRY}

                    echo "===== PULL IMAGE ====="

                    docker pull ${ECR_REPO}:${IMAGE_TAG}
                '''
            }
        }

        stage('Deploy Application') {
            agent { label 'docker-worker' }

            steps {
                sh '''
                    echo "===== REMOVE OLD CONTAINER ====="

                    docker rm -f java-webapp-container || true

                    echo "===== START NEW CONTAINER ====="

                    docker run -d \
                      --name java-webapp-container \
                      -p 8080:8080 \
                      ${ECR_REPO}:${IMAGE_TAG}

                    echo "===== CONTAINER STATUS ====="

                    docker ps

                    echo "===== APPLICATION LOGS ====="

                    sleep 5

                    docker logs \
                      --tail 50 \
                      java-webapp-container
                '''
            }
        }
    }

    post {
        success {
            echo '======================================'
            echo 'PIPELINE SUCCESSFUL'
            echo '======================================'
        }

        failure {
            echo '======================================'
            echo 'PIPELINE FAILED'
            echo 'Check the failed stage in Console Output.'
            echo '======================================'
        }
    }
}
