pipeline {
    agent any

    tools {
        jdk 'jdk17'  // Ensure this matches the exact JDK name in Global Tool Configuration
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        TRIVY_TIMEOUT = '10m' // Increased timeout for Trivy operations
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
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh '''
                export TRIVY_TIMEOUT=$TRIVY_TIMEOUT
                trivy fs --format table -o fs.html .
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=Blogging-app \
                    -Dsonar.projectKey=Blogging-app \
                    -Dsonar.sources=src/main/java \
                    -Dsonar.java.binaries=target/classes \
                    -Dsonar.language=java
                    '''
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Publish Artifacts') {
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
                        docker build -t saliu21/bloggingapp:latest . --no-cache
                        '''
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                export TRIVY_TIMEOUT=$TRIVY_TIMEOUT
                trivy image --format table -o image.html saliu21/bloggingapp:latest
                '''
            }
        }

        stage('Docker Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh '''
                        docker push saliu21/bloggingapp:latest
                        '''
                    }
                }
            }
        }

        stage('K8s Deploy') {
            steps {
                script {
                    withKubeConfig(kubeconfig: '/home/vagrant/jenkins-kube/config') {
                        sh '''
                        kubectl apply -f /home/vagrant/deployment-service.yml
                        sleep 20
                        '''
                    }
                }
            }
        }

        stage('Verify the Deployment') {
            steps {
                script {
                    withKubeConfig(kubeconfig: '/home/vagrant/jenkins-kube/config') {
                        sh '''
                        kubectl get pods
                        kubectl get svc
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline completed.'
        }

        failure {
            echo 'Pipeline failed. Check logs for details.'
        }

        success {
            echo 'Pipeline completed successfully.'
        }
    }
}
