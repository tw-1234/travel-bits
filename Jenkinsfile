pipeline {
    agent any

    tools {
        nodejs 'NodeJS'
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
                    node -v
                    npm -v
                '''
            }
        }

        stage('Decrypt Secrets using SOPS') {
            steps {
                withCredentials([
                    string(credentialsId: 'sops-age-key', variable: 'SOPS_AGE_KEY')
                ]) {
                    sh '''
                        echo "Decrypting secrets..."
                        export SOPS_AGE_KEY=$SOPS_AGE_KEY
                        mkdir -p secrets
                        sops -d secrets/secrets.enc.yaml > secrets/secrets.dec.yaml
                        echo "Secrets decrypted successfully"
                    '''
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    npm install
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build -t travel-bits .
                '''
            }
        }

        stage('Run Container') {
            steps {
                sh '''
                    docker rm -f travel-bits || true
                    docker run -d -p 3000:3000 --name travel-bits travel-bits
                '''
            }
        }

        stage('HEY Load Test') {
            steps {
                sh '''
                    hey -z 10s -c 10 http://localhost:3000 || true
                '''
            }
        }
    }

    post {
        always {
            sh '''
                rm -f secrets/secrets.dec.yaml
            '''
            echo "Pipeline finished: ${currentBuild.currentResult}"
        }
    }
}
