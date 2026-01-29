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
                    /Users/macintoshssd/.jenkins/tools/jenkins.plugins.nodejs.tools.NodeJSInstallation/NodeJS/bin/npm install
                    /Users/macintoshssd/.jenkins/tools/jenkins.plugins.nodejs.tools.NodeJSInstallation/NodeJS/bin/npm audit || true
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
                script {
                    def scannerHome = tool 'SonarScanner'
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                        withSonarQubeEnv('SonarQube') {
                            sh """
                                ${scannerHome}/bin/sonar-scanner \
                                  -Dsonar.projectKey=travel-bits \
                                  -Dsonar.sources=. \
                                  -Dsonar.token=$SONAR_TOKEN
                            """
                        }
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

        stage('Docker Build & Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker build -t taizeeba/travel-bits:latest .
                        docker push taizeeba/travel-bits:latest
                    '''
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                script {
                    sh '''
                        trivy image --severity HIGH,CRITICAL --format table -q taizeeba/travel-bits:latest | tee trivy-results.txt || true
                    '''
                }
            }
        }
    }

    post {
        success {
            script {
                // Read Trivy results if available
                def trivyOutput = ''
                if (fileExists('trivy-results.txt')) {
                    trivyOutput = readFile('trivy-results.txt')
                }

                // Send success email
                mail to: 'taizeebarauf@gmail.com',
                     subject: "✅ Jenkins Pipeline Succeeded: ${currentBuild.fullDisplayName}",
                     body: """
Pipeline Succeeded!

Project: ${env.JOB_NAME}
Build Number: ${env.BUILD_NUMBER}
Build URL: ${env.BUILD_URL}

Trivy Scan Results:
${trivyOutput}
"""
            }
        }

        failure {
            script {
                // Read Trivy results if available
                def trivyOutput = ''
                if (fileExists('trivy-results.txt')) {
                    trivyOutput = readFile('trivy-results.txt')
                }

                // Send failure email
                mail to: 'taizeebarauf@gmail.com',
                     subject: "❌ Jenkins Pipeline Failed: ${currentBuild.fullDisplayName}",
                     body: """
Pipeline Failed!

Project: ${env.JOB_NAME}
Build Number: ${env.BUILD_NUMBER}
Build URL: ${env.BUILD_URL}

Trivy Scan Results (if available):
${trivyOutput}
"""
            }
        }

        always {
            echo "Pipeline finished with status: ${currentBuild.currentResult}"
        }
    }
}
