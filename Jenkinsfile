pipeline {
    agent any

    tools {
        nodejs 'NodeJS'  // Only if NodeJS plugin installed
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

        // <-- Replace your old SonarQube Scan stage with this one
        stage('SonarQube Scan') {
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    withSonarQubeEnv('SonarQube') { // must match Jenkins SonarQube server name
                        tool name: 'SonarScanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation'
                        sh '''
                        sonar-scanner \
                        -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=${SONAR_HOST_URL} \
                        -Dsonar.login=${SONAR_TOKEN}
                        '''
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
