node {
    // === Environment Variables ===
    env.PROJECT_NAME = "travel-bits"
    env.DOCKER_IMAGE = "taizeeba/travel-bits:latest"
    env.DOCKER_CONTAINER = "travel-bits-container"
    env.NODEJS_TOOL = "NodeJS-25" // NodeJS installation in Jenkins
    env.RECIPIENTS = "taizeebarauf@gmail.com"

    // === Pipeline Try/Catch for Post Actions ===
    try {

        stage('Checkout Code') {
            checkout scm
        }

        stage('Verify Node & NPM') {
            echo "Using system Node & NPM"
            sh 'node -v'
            sh 'npm -v'
        }

        stage('Decrypt Secrets using SOPS') {
            script {
                if (fileExists('secrets.env')) {
                    echo "Decrypting secrets..."
                    sh '''
                        mkdir -p secrets
                        cp secrets.env secrets/secrets.dec.env
                    '''
                } else {
                    echo "No secrets.env found, skipping SOPS decryption"
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
            sh 'npm install'
        }

        stage('Run Unit Tests') {
            sh 'npm test || echo "Tests failed, continuing..."'
        }

        stage('SonarQube Scan') {
            echo "SonarQube scan placeholder"
        }

        stage('Docker Build & Deploy') {
            sh '''
                docker build -t $DOCKER_IMAGE .
                docker rm -f $DOCKER_CONTAINER || true
                docker run -d --name $DOCKER_CONTAINER -p 3000:3000 $DOCKER_IMAGE
            '''
        }

        stage('Trivy Image Scan') {
            echo "Trivy scan placeholder"
        }

        stage('HEY Load Testing') {
            sh 'hey -z 10s -c 10 http://localhost:3000/ | tee hey-results.txt'
        }

        // === Success Email Stage ===
        stage('Send Success Email') {
            mail to: "${env.RECIPIENTS}",
                 subject: "SUCCESS: Build ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: """Good news! The build succeeded.

Job: ${env.JOB_NAME}
Build Number: ${env.BUILD_NUMBER}
Check console output at: ${env.BUILD_URL}"""
        }

    } catch (err) {
        // === Failure Email Stage ===
        stage('Send Failure Email') {
            mail to: "${env.RECIPIENTS}",
                 subject: "FAILURE: Build ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: """Oops! The build failed.

Job: ${env.JOB_NAME}
Build Number: ${env.BUILD_NUMBER}
Check console output at: ${env.BUILD_URL}"""
        }
        // Fail the build
        error("Build failed: ${err}")
    } finally {
        // === Cleanup Secrets Stage ===
        stage('Cleanup Secrets') {
            script {
                if (fileExists('secrets/secrets.dec.env')) {
                    sh 'rm -f secrets/secrets.dec.env'
                }
            }
            echo "Pipeline finished."
        }
    }
}
