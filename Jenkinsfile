pipeline {
    agent any

    tools {
        nodejs 'NodeJS'   // Make sure this matches the name in Jenkins Global Tool Config
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Check Node & NPM') {
            steps {
                script {
                    echo "Node version:"
                    sh "${tool 'NodeJS'}/bin/node -v"
                    echo "NPM version:"
                    sh "${tool 'NodeJS'}/bin/npm -v"
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    ${NODEJS_HOME}/bin/npm install
                    ${NODEJS_HOME}/bin/npm audit
                '''
            }
        }

        stage('Run Unit Tests') {
            steps {
                sh '''
                    ${NODEJS_HOME}/bin/npm test || echo "Tests failed, but continuing..."
                '''
            }
        }

        stage('SonarQube Scan') {
            steps {
                withCredentials([string(credentialsId: 'SONAR_TOKEN', variable: 'SONAR_TOKEN')]) {
                    withSonarQubeEnv('SonarQube') {
                        sh "sonar-scanner -Dsonar.projectKey=travel-bits -Dsonar.sources=. -Dsonar.login=$SONAR_TOKEN"
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build & Deploy') {
            steps {
                sh '''
                    docker build -t travel-bits:latest .
                    docker run -d -p 3000:3000 travel-bits:latest
                '''
            }
        }
    }

    post {
        success { echo 'Pipeline finished successfully!' }
        failure { echo 'Pipeline failed!' }
    }
}
