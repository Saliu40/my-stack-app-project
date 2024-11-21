pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        TRIVY_TIMEOUT = '10m'
        PROXY_USER = credentials('proxy-username-id')
        PROXY_PASS = credentials('proxy-password-id')
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
                junit '**/target/surefire-reports/*.xml'
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh '''
                export TRIVY_TIMEOUT=$TRIVY_TIMEOUT
                trivy fs --exit-code 1 --severity HIGH,CRITICAL --format table -o fs.html .
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
                    -Dsonar.login=$SONAR_TOKEN
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    def qg = waitForQualityGate()
                    if (qg.status != 'OK') {
                        error "Pipeline failed due to SonarQube quality gate failure: ${qg.status}"
                    }
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
                        DOCKER_BUILDKIT=1 docker buildx build --platform linux/amd64,linux/arm64 -t saliu21/bloggingapp:latest .
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
            }
        }

        stage('Docker Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh 'docker push saliu21/bloggingapp:latest'
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
                        kubectl rollout status deployment/<deployment-name>
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
