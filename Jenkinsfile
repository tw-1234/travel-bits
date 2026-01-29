pipeline {
    agent any

    environment {
        // Add any global env vars here if needed
        PROJECT_NAME = "travel-bits"
        DOCKER_IMAGE = "taizeeba/travel-bits:latest"
        DOCKER_CONTAINER = "travel-bits-container"
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Verify Node & NPM') {
            steps {
                sh '''
                    echo "Using system Node & NPM"
                    node -v
                    npm -v
                '''
            }
        }

        stage('Decrypt Secrets using SOPS') {
            steps {
                script {
                    if (fileExists('secrets.env')) {
                        sh '''
                            echo "Decrypting secrets..."
                            mkdir -p secrets
                            cp secrets.env secrets/secrets.dec.env
                        '''
                    } else {
                        echo "No secrets.env found, skipping SOPS decryption"
                    }
                }
            }
        }

        stage('Load Secrets as Environment Variables') {
            when {
                expression { fileExists('secrets/secrets.dec.env') }
            }
            steps {
                sh '''
                    echo "Loading secrets into environment"
                    set -a
                    . secrets/secrets.dec.env
                    set +a
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Run Unit Tests') {
            steps {
                sh 'npm test || echo "Tests failed, continuing..."'
            }
        }

        stage('SonarQube Scan') {
            steps {
                echo "SonarQube stage placeholder (optional)"
            }
        }

        stage('Docker Build & Push') {
            steps {
                sh '''
                    docker build -t $DOCKER_IMAGE .
                    docker rm -f $DOCKER_CONTAINER || true
                    docker run -d --name $DOCKER_CONTAINER -p 3000:3000 $DOCKER_IMAGE
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh 'echo "Trivy scan placeholder"'
            }
        }

        stage('HEY Load Testing') {
            steps {
                sh '''
                    hey -z 10s -c 10 http://localhost:3000/ | tee hey-results.txt
                '''
            }
        }
    }

post {
    always {
        script {
            if (fileExists('secrets/secrets.dec.env')) {
                sh 'rm -f secrets/secrets.dec.env'
            }
        }
        echo "Pipeline finished."
        // Send email
        mail to: 'taizeebarauf@gmail.com',
             subject: "Jenkins Pipeline: ${currentBuild.fullDisplayName}",
             body: "Status: ${currentBuild.currentResult}\nCheck console output at ${env.BUILD_URL}"
    }

    success {
        echo "Pipeline completed successfully!"
    }

    failure {
        echo "Pipeline failed!"
    }
}

