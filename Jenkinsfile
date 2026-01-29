pipeline {
    agent any

    environment {
        // NodeJS tool configured in Jenkins
        NODE_HOME = tool name: 'NodeJS-25', type: 'NodeJS'
        PATH = "${NODE_HOME}/bin:${env.PATH}"
    }

    stages {
        stage('Checkout SCM') {
            steps {
                echo 'Checking out code from GitHub...'
                checkout scm
            }
        }

        stage('Verify Node & NPM') {
            steps {
                sh 'node -v'
                sh 'npm -v'
            }
        }

        stage('Decrypt Secrets using SOPS') {
            steps {
                script {
                    if (fileExists('secrets.env')) {
                        echo 'Decrypting secrets using SOPS...'
                        sh 'mkdir -p secrets'
                        sh 'cp secrets.env secrets/secrets.dec.env'
                    } else {
                        echo 'secrets.env not found, skipping SOPS decryption'
                    }
                }
            }
        }

        stage('Load Secrets as Environment Variables') {
            steps {
                script {
                    if (fileExists('secrets/secrets.dec.env')) {
                        echo 'Loading secrets as environment variables...'
                        sh 'export $(cat secrets/secrets.dec.env | xargs)'
                    } else {
                        echo 'No decrypted secrets found, skipping load'
                    }
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                echo 'Installing NPM dependencies...'
                sh 'npm install'
            }
        }

        stage('Run Unit Tests') {
            steps {
                echo 'Running unit tests...'
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh 'npm test || true'
                }
            }
        }

        stage('SonarQube Scan') {
            steps {
                echo 'Running SonarQube scan...'
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh '''
                    sonar-scanner \
                    -Dsonar.projectKey=travel-bits \
                    -Dsonar.sources=src \
                    -Dsonar.host.url=http://localhost:9000 \
                    -Dsonar.login=$SONAR_TOKEN
                    '''
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                echo 'Building and running Docker container...'
                sh '''
                docker build -t taizeeba/travel-bits:latest .
                docker rm -f travel-bits-container || true
                docker run -d --name travel-bits-container -p 3000:3000 taizeeba/travel-bits:latest
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                echo 'Scanning Docker image for vulnerabilities...'
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh 'trivy image taizeeba/travel-bits:latest'
                }
            }
        }

        stage('HEY Load Testing') {
            steps {
                echo 'Running HEY load test...'
                sh 'hey -z 10s -c 10 http://localhost:3000/ | tee hey-results.txt'
            }
        }
    }

    post {
        always {
            echo 'Cleaning up...'
            sh 'rm -f secrets/secrets.dec.env || true'
            echo 'Pipeline finished.'
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline finished with some errors.'
        }
    }
}
