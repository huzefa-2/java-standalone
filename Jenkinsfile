pipeline {
    agent none

    environment {
        AWS_REGION     = 'us-east-1'
        AWS_ACCOUNT_ID = '177203142049'

        ECR_REGISTRY   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        ECR_REPO       = "${ECR_REGISTRY}/java-webapp"
        IMAGE_TAG      = "v${BUILD_NUMBER}"

        SONAR_HOST_URL = 'http://54.92.217.177:9000'

        K8S_NAMESPACE  = 'java-webapp'
        K8S_DEPLOYMENT = 'java-webapp'
    }

    stages {

        stage('Checkout') {
            agent { label 'sonarqube' }

            steps {
                git branch: 'master',
                    url: 'https://github.com/huzefa-2/java-standalone.git'

                sh '''
                    echo "===== CHECKOUT ====="
                    ls -la
                '''
            }
        }

        stage('Maven Build') {
            agent { label 'sonarqube' }

            steps {
                sh '''
                    echo "===== JAVA ====="
                    java -version

                    echo "===== MAVEN ====="
                    mvn -version

                    echo "===== MAVEN BUILD ====="
                    mvn clean package -DskipTests
                '''
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

                git branch: 'master',
                    url: 'https://github.com/huzefa-2/java-standalone.git'

                sh '''
                    echo "===== DOCKER VERSION ====="
                    docker --version

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
                    echo "===== TRIVY VERSION ====="
                    trivy --version

                    mkdir -p trivy-tmp

                    export TMPDIR=$PWD/trivy-tmp
                    export TRIVY_CACHE_DIR=$PWD/trivy-tmp

                    echo "===== TRIVY SCAN ====="

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

        stage('Verify EKS Access') {
            agent { label 'docker-worker' }

            steps {

                sh '''
                    echo "===== AWS IDENTITY ====="

                    aws sts get-caller-identity

                    echo "===== KUBECTL ====="

                    kubectl version --client

                    echo "===== EKS NODES ====="

                    kubectl get nodes

                    echo "===== NAMESPACE ====="

                    kubectl get namespace ${K8S_NAMESPACE}
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            agent { label 'docker-worker' }

            steps {

                deleteDir()

                git branch: 'master',
                    url: 'https://github.com/huzefa-2/java-standalone.git'

                sh '''
                    echo "===== KUBERNETES DEPLOYMENT ====="

                    echo "===== APPLY SERVICE ====="

                    kubectl apply \
                      -f k8s/service.yaml

                    echo "===== UPDATE IMAGE ====="

                    sed "s|PLACEHOLDER|${IMAGE_TAG}|g" \
                      k8s/deployment.yaml \
                      > deployment-${BUILD_NUMBER}.yaml

                    echo "===== APPLY DEPLOYMENT ====="

                    kubectl apply \
                      -f deployment-${BUILD_NUMBER}.yaml

                    echo "===== ROLLOUT STATUS ====="

                    kubectl rollout status \
                      deployment/${K8S_DEPLOYMENT} \
                      -n ${K8S_NAMESPACE} \
                      --timeout=5m
                '''
            }
        }

        stage('Verify Application') {
            agent { label 'docker-worker' }

            steps {

                sh '''
                    echo "===== DEPLOYMENT ====="

                    kubectl get deployment \
                      -n ${K8S_NAMESPACE}

                    echo "===== PODS ====="

                    kubectl get pods \
                      -n ${K8S_NAMESPACE} \
                      -o wide

                    echo "===== SERVICE ====="

                    kubectl get svc \
                      -n ${K8S_NAMESPACE}
                '''
            }
        }
    }

    post {

        success {
            echo '''
            =========================================
                 PIPELINE SUCCESSFUL
            =========================================

            Application successfully:

            Git → Maven → SonarQube → Trivy
                → ECR → EKS → Kubernetes

            =========================================
            '''
        }

        failure {
            echo '''
            =========================================
                    PIPELINE FAILED
            =========================================

            Check the failed stage above.

            =========================================
            '''
        }
    }
}
