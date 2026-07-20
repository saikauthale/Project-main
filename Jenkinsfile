pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "saikauthale/java_applicationdevopsproject"
        DOCKER_TAG = "${BUILD_NUMBER}"

        REGION = "ap-south-1"
        CLUSTER_NAME = "java-eks-cluster"

        SONARQUBE_URL = "http://13.204.64.103:9000""
        SONAR_PROJECT_KEY = "java-app"
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            environment {
                SONAR_TOKEN = credentials('sonar-token')
            }

            steps {
                script {
                    docker.image('sonarsource/sonar-scanner-cli:latest').inside('--user root --entrypoint=""') {
                        sh '''
                            mkdir -p $WORKSPACE/.sonar

                            sonar-scanner \
                              -Dsonar.projectKey=$SONAR_PROJECT_KEY \
                              -Dsonar.sources=. \
                              -Dsonar.host.url=$SONARQUBE_URL \
                              -Dsonar.token=$SONAR_TOKEN \
                              -Dsonar.userHome=$WORKSPACE/.sonar
                        '''
                    }
                }
            }
        }

        stage('Clean Sonar Files') {
            steps {
                sh '''
                    rm -rf $WORKSPACE/.sonar || true
                    rm -rf $WORKSPACE/.scannerwork || true
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t $DOCKER_IMAGE:$DOCKER_TAG .
                '''
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker-hub-cred',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    '''
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                sh '''
                    docker push $DOCKER_IMAGE:$DOCKER_TAG

                    docker tag $DOCKER_IMAGE:$DOCKER_TAG $DOCKER_IMAGE:latest

                    docker push $DOCKER_IMAGE:latest
                '''
            }
        }

        stage('Deploy to AWS EKS') {
            steps {
                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials'
                    ]
                ]) {
                    sh '''
                        aws eks update-kubeconfig \
                            --region $REGION \
                            --name $CLUSTER_NAME

                        kubectl set image deployment/java-war-deployment \
                            java-war-container=$DOCKER_IMAGE:$DOCKER_TAG

                        kubectl rollout status deployment/java-war-deployment
                    '''
                }
            }
        }
    }

    post {
        success {
            echo '✅ Deployment Successful'
        }

        failure {
            echo '❌ Pipeline Failed'
        }

        always {
            cleanWs()
        }
    }
}
