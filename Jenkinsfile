pipeline {
    agent any

    tools {
        jdk 'jdk17'  // Matches the configured JDK in Jenkins Global Tool Configuration
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        TRIVY_TIMEOUT = '10m'  // Timeout for Trivy operations
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

        stage('Trivy FS Scan') {
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

        stage('Publish Artifacts') {
            steps {
                withMaven(globalMavenSettingsConfig: 'anything', jdk: 'jdk17', maven: 'maven3', traceability: true) {
                    sh 'mvn deploy -Dmaven.repo.local=/var/lib/jenkins/.m2/repository'
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
                trivy image --exit-code 1 --severity HIGH,CRITICAL --format table -o image.html saliu21/bloggingapp:latest
                '''
                archiveArtifacts artifacts: 'image.html', allowEmptyArchive: true // Save Trivy Image report
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
                        kubectl rollout status deployment/<deployment-name> --timeout=60s
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
