pipeline {
    agent any

    tools {
        jdk 'jdk17'  // Matches the configured JDK in Jenkins Global Tool Configuration
        maven 'maven3'  // Matches the configured Maven version
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        TRIVY_TIMEOUT = '10m'  // Timeout for Trivy operations
        DOCKER_IMAGE = 'saliu21/bloggingapp:latest'  // Centralized Docker image tag
        KUBECONFIG = '/home/vagrant/jenkins-kube/config'  // Path to kubeconfig
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
                archiveArtifacts artifacts: 'fs.html', allowEmptyArchive: true // Save Trivy FS report
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
                archiveArtifacts artifacts: 'target/*.jar', allowEmptyArchive: true // Save the JAR
            }
        }

        stage('Publish Artifacts to Repository') {
            steps {
                withMaven(globalMavenSettingsConfig: 'anything', jdk: 'jdk17', maven: 'maven3', traceability: true) {
                    sh 'mvn deploy'
                }
            }
        }

        stage('Docker Build & Tag') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh '''
                        docker build -t $DOCKER_IMAGE . --no-cache
                        '''
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                export TRIVY_TIMEOUT=$TRIVY_TIMEOUT
                trivy image --exit-code 0 --severity HIGH,CRITICAL --format table -o image.html $DOCKER_IMAGE
                '''
                archiveArtifacts artifacts: 'image.html', allowEmptyArchive: true // Save Trivy Image report
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh '''
                        docker push $DOCKER_IMAGE
                        '''
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    withKubeConfig([credentialsId: 'k8-cred']) {
                        sh '''
                        kubectl apply -f /home/vagrant/deployment-service.yml
                        DEPLOYMENT_NAME=$(kubectl get deployment -n webapps --selector=app=my-app -o jsonpath="{.items[0].metadata.name}")
                        kubectl rollout status deployment/$DEPLOYMENT_NAME -n webapps --timeout=60s
                        '''
                    }
                }
            }
        }

        stage('Verify Kubernetes Deployment') {
            steps {
                script {
                    withKubeConfig([credentialsId: 'k8-cred']) {
                        sh '''
                        kubectl get pods -n webapps
                        kubectl get svc -n webapps
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline completed.'
            cleanWs() // Clean up the workspace after the pipeline
        }

        failure {
            echo 'Pipeline failed. Check logs for details.'
        }

        success {
            echo 'Pipeline completed successfully.'
        }
    }
}
