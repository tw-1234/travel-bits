pipeline {
    agent any

    tools {
        nodejs 'NodeJS'  // Make sure NodeJS plugin is installed in Jenkins
    }

    environment {
        SONAR_HOST_URL = 'http://localhost:9000'
        SONAR_PROJECT_KEY = 'TravelApp'
    }

    stages {

        stage('Check Node & NPM') {
            steps {
                sh '''
                node -v
                npm -v
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('SonarQube Scan') {
            steps {
                // Use the correct token ID you just provided
                withCredentials([string(credentialsId: 'sqa_2d4f48d392b6fbb09669c5ec1f9df3d793c63ba3', variable: 'SONAR_TOKEN')]) {
                    withSonarQubeEnv('SonarQube') {
                        sh "${tool 'SonarScanner'}/bin/sonar-scanner \
                            -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=${SONAR_HOST_URL} \
                            -Dsonar.login=${SONAR_TOKEN}"
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
    }
}
