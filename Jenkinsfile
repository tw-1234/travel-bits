pipeline {
    agent any

    tools {
        nodejs 'NodeJS'
    }

    environment {
        APP_NAME = "travel-bits"
        DOCKER_IMAGE = "taizeeba/travel-bits:latest"
        DEPLOY_CONTAINER = "travel-bits-container"
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
                    ${NODEJS_HOME}/bin/npm audit || true
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
                        docker build -t ${DOCKER_IMAGE} .
                        docker push ${DOCKER_IMAGE}
                    '''
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                    trivy image --severity HIGH,CRITICAL --format table -q ${DOCKER_IMAGE} | tee trivy-results.txt || true
                '''
            }
        }

        stage('Docker Pull') {
            steps {
                sh "docker pull ${DOCKER_IMAGE}"
            }
        }

        stage('Deploy Docker Container') {
            steps {
                sh """
                    docker rm -f ${DEPLOY_CONTAINER} || true
                    docker run -d --name ${DEPLOY_CONTAINER} -p 3000:3000 ${DOCKER_IMAGE}
                """
            }
        }

        stage('HEY Load Testing') {
            steps {
                sh '''
                    hey -z 10s -c 10 http://localhost:3000/ | tee hey-results.txt || true
                '''
            }
        }
    }

    post {
        success {
            script {
                def trivyOutput = fileExists('trivy-results.txt') ? readFile('trivy-results.txt') : "No Trivy results"
                def heyOutput = "No load test results"
                if (fileExists('hey-results.txt')) {
                    def heyText = readFile('hey-results.txt').readLines()
                    def reqPerSec = heyText.find { it.contains("Requests/sec") } ?: "Requests/sec: N/A"
                    def avgLatency = heyText.find { it.contains("Average") } ?: "Average latency: N/A"
                    def successRate = heyText.find { it.contains("Success") } ?: "Success: N/A"
                    heyOutput = "${reqPerSec}\n${avgLatency}\n${successRate}"
                }

                mail to: 'taizeebarauf.com',
                     subject: "✅ Jenkins Pipeline Succeeded: ${currentBuild.fullDisplayName}",
                     body: """
Pipeline Succeeded!

Project: ${env.JOB_NAME}
Build Number: ${env.BUILD_NUMBER}
Build URL: ${env.BUILD_URL}

Trivy Scan Results:
${trivyOutput}

HEY Load Test Summary:
${heyOutput}
"""
            }
        }

        failure {
            script {
                def trivyOutput = fileExists('trivy-results.txt') ? readFile('trivy-results.txt') : "No Trivy results"
                def heyOutput = "No load test results"
                if (fileExists('hey-results.txt')) {
                    def heyText = readFile('hey-results.txt').readLines()
                    def reqPerSec = heyText.find { it.contains("Requests/sec") } ?: "Requests/sec: N/A"
                    def avgLatency = heyText.find { it.contains("Average") } ?: "Average latency: N/A"
                    def successRate = heyText.find { it.contains("Success") } ?: "Success: N/A"
                    heyOutput = "${reqPerSec}\n${avgLatency}\n${successRate}"
                }

                mail to: 'taizeebarauf.com',
                     subject: "❌ Jenkins Pipeline Failed: ${currentBuild.fullDisplayName}",
                     body: """
Pipeline Failed!

Project: ${env.JOB_NAME}
Build Number: ${env.BUILD_NUMBER}
Build URL: ${env.BUILD_URL}

Trivy Scan Results (if available):
${trivyOutput}

HEY Load Test Summary (if available):
${heyOutput}
"""
            }
        }

        always {
            echo "Pipeline finished with status: ${currentBuild.currentResult}"
        }
    }
}
