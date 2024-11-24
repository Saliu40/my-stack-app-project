pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS = credentials('docker-credentials-id') // Replace with your Docker credentials ID
        KUBECONFIG_PATH = '/home/vagrant/.kube/config'
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh '''
                    docker build -t saliu21/bloggingapp:latest .
                    '''
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry([url: 'https://index.docker.io/v1/', credentialsId: DOCKER_CREDENTIALS]) {
                        sh '''
                        docker push saliu21/bloggingapp:latest
                        '''
                    }
                }
            }
        }

        stage('Debug Environment') {
            steps {
                sh '''
                echo "Debugging Environment..."
                whoami
                ls -l ${KUBECONFIG_PATH}
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    withEnv(["KUBECONFIG=${KUBECONFIG_PATH}"]) {
                        sh '''
                        kubectl apply -f /home/vagrant/deployment-service.yml
                        kubectl get pods
                        '''
                    }
                }
            }
        }

        stage('Verify Kubernetes Deployment') {
            steps {
                script {
                    withEnv(["KUBECONFIG=${KUBECONFIG_PATH}"]) {
                        sh '''
                        kubectl get deployments
                        kubectl get services
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Cleaning workspace...'
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check logs for details.'
        }
    }
}
