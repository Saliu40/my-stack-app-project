pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        TRIVY_TIMEOUT = '10m'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/Saliu40/my-stack-app-project.git'
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn compile --settings /path/to/jenkins-managed-settings.xml'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test --settings /path/to/jenkins-managed-settings.xml'
                junit '**/target/surefire-reports/*.xml'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package --settings /path/to/jenkins-managed-settings.xml'
            }
        }

        stage('Publish Artifacts') {
            steps {
                withMaven(globalMavenSettingsConfig: 'nexus-creds', jdk: 'jdk17', maven: 'maven3') {
                    sh 'mvn deploy --settings /path/to/jenkins-managed-settings.xml'
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
