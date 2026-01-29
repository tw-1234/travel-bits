pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/tw-1234/travel-bits.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Run App') {
            steps {
                sh 'npm start &'
            }
        }

        stage('Docker Build & Deploy') {
            steps {
                sh '''
                    docker build -t taizeeba/travel-bits:latest .
                    docker rm -f travel-bits-container || true
                    docker run -d --name travel-bits-container -p 3000:3000 taizeeba/travel-bits:latest
                '''
            }
        }

        stage('HEY Load Testing') {
            steps {
                sh 'hey -z 10s -c 10 http://localhost:3000/ | tee hey-results.txt'
            }
        }
    }

    post {
        always {
            echo 'Cleaning up...'
            sh 'rm -f secrets/secrets.dec.env || true'
            
            mail to: 'your-email@example.com',
                 subject: "Jenkins Pipeline: ${currentBuild.fullDisplayName}",
                 body: "Status: ${currentBuild.currentResult}\nCheck console output at ${env.BUILD_URL}"
        }

        success {
            echo 'Pipeline completed successfully!'
        }

        failure {
            echo 'Pipeline failed!'
        }
    }
}
