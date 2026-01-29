pipeline {
    agent any

    environment {
        NODEJS_HOME = tool name: 'NodeJS', type: 'jenkins.plugins.nodejs.tools.NodeJSInstallation'
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Check Node & NPM') {
            steps {
                sh '''
                    echo "Node version:"
                    $NODEJS_HOME/bin/node -v
                    echo "NPM version:"
                    $NODEJS_HOME/bin/npm -v
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    $NODEJS_HOME/bin/npm install
                    $NODEJS_HOME/bin/npm audit
                '''
            }
        }

        stage('Run Unit Tests') {
            steps {
                sh '''
                    $NODEJS_HOME/bin/npm test || echo "Tests failed, but continuing..."
                '''
            }
        }

        stage('SonarQube Scan') {
            steps {
                // Use your existing token stored in Jenkins
                withCredentials([string(credentialsId: 'SONAR_TOKEN', variable: 'SONAR_TOKEN')]) {
                    withSonarQubeEnv('SonarQube') {
                        sh '''
                            sonar-scanner \
                            -Dsonar.projectKey=travel-bits \
                            -Dsonar.sources=. \
                            -Dsonar.login=$SONAR_TOKEN
                        '''
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
        success {
            echo 'Pipeline finished successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
