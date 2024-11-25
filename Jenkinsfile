pipeline {
    agent any

    tools {
        jdk 'jdk17'  // Configured JDK in Jenkins
        maven 'maven3'  // Configured Maven version
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        TRIVY_TIMEOUT = '10m'  // Timeout for Trivy operations
        DOCKER_IMAGE = 'bloggingapp:latest'  // Docker image tag for Minikube
        KUBECONFIG = '/home/vagrant/.kube/config'  // Minikube kubeconfig path
    }

    stages {

        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/Saliu40/my-stack-app-project.git'
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
                junit '**/target/surefire-reports/*.xml'  // Capture test results for reporting
            }
        }

        stage('Static Code Analysis with Trivy FS') {
            steps {
                sh '''
                export TRIVY_TIMEOUT=$TRIVY_TIMEOUT
                trivy fs --exit-code 0 --severity HIGH,CRITICAL --format table -o fs.html .
                '''
                archiveArtifacts artifacts: 'fs.html', allowEmptyArchive: true  // Save Trivy FS report
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                        sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=Blogging-app \
                        -Dsonar.projectKey=Blogging-app \
                        -Dsonar.sources=src/main/java \
                        -Dsonar.java.binaries=target/classes \
                        -Dsonar.login=$SONAR_TOKEN
                        '''
                    }
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package'
                archiveArtifacts artifacts: 'target/*.jar', allowEmptyArchive: true  // Save the JAR
            }
        }

        stage('Docker Build for Minikube') {
            steps {
                script {
                    // Use Minikube's Docker daemon
                    sh 'eval $(minikube -p minikube docker-env)'
                    sh '''
                    docker build -t $DOCKER_IMAGE .
                    '''
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                export TRIVY_TIMEOUT=$TRIVY_TIMEOUT
                trivy image --exit-code 0 --severity HIGH,CRITICAL --format table -o image.html $DOCKER_IMAGE
                '''
                archiveArtifacts artifacts: 'image.html', allowEmptyArchive: true  // Save Trivy Image report
            }
        }

        stage('k8-Deploy') {
            steps {
                script {
                    sh '''
                    kubectl apply -f deployment-service.yml
                    sleep 20
                    '''
                }
            }
        }

        stage('Verify the Deployment') {
            steps {
                script {
                    sh '''
                    kubectl get pods
                    kubectl get svc
                    '''
                }
            }
        }
    }
}
