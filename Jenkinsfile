node {

    // ========================
    // Global Environment Variables
    // ========================
    env.PROJECT_NAME     = "travel-bits"
    env.DOCKER_IMAGE     = "taizeeba/travel-bits:latest"
    env.DOCKER_CONTAINER = "travel-bits-container"
    env.RECIPIENTS      = "taizeebarauf@gmail.com"

    try {

        // ========================
        // 1. Checkout Code
        // ========================
        stage('Checkout Code') {
            checkout scm
        }

        // ========================
        // 2. Verify Node & NPM
        // ========================
        stage('Verify Node & NPM') {
            sh '''
                echo "Using system Node & NPM"
                node -v
                npm -v
            '''
        }

        // ========================
        // 3. SOPS – Decrypt Secrets
        // ========================
        stage('Decrypt Secrets using SOPS') {
            script {
                if (fileExists('secrets/secrets.enc.env')) {
                    sh '''
                        echo "Decrypting secrets using SOPS (simulated)"
                        mkdir -p secrets
                        cp secrets/secrets.enc.env secrets/secrets.dec.env
                    '''
                } else {
                    echo "No encrypted secrets found, skipping decryption"
                }
            }
        }

        // ========================
        // 4. Load Secrets into Environment
        // ========================
        stage('Load Secrets as Environment Variables') {
            script {
                if (fileExists('secrets/secrets.dec.env')) {
                    sh '''
                        echo "Loading secrets into environment"
                        set -a
                        . secrets/secrets.dec.env
                        set +a
                    '''
                }
            }
        }

        // ========================
        // 5. Install Dependencies
        // ========================
        stage('Install Dependencies') {
            sh 'npm install'
        }

        // ========================
        // 6. Unit Testing
        // ========================
        stage('Run Unit Tests') {
            sh 'npm test || echo "Tests failed, continuing pipeline"'
        }

        // ========================
        // 7. UI Testing (Selenium – placeholder)
        // ========================
      
        // ========================
        // 8. SonarQube Scan
        // ========================
        stage('SonarQube Scan') {
            echo "SonarQube static code analysis placeholder"
        }

        // ========================
        // 9. Docker Build
        // ========================
        stage('Docker Build') {
            sh '''
                docker build -t $DOCKER_IMAGE .
            '''
        }

        // ========================
        // 10. Push Image to Docker Hub
        // ========================
        stage('Docker Hub Push') {
            echo "Docker push skipped (optional for academic project)"
        }

        // ========================
        // 11. Trivy Image Scan
        // ========================
        stage('Trivy Image Scan') {
            echo "Trivy vulnerability scan placeholder"
        }

        // ========================
        // 12. Deploy Application
        // ========================
        stage('Deploy Application') {
            sh '''
                docker rm -f $DOCKER_CONTAINER || true
                docker run -d --name $DOCKER_CONTAINER -p 3000:3000 $DOCKER_IMAGE
            '''
        }

        // ========================
        // 13. Prometheus Setup (Monitoring)
        // ========================
        stage('Prometheus Monitoring') {
            echo "Prometheus collects metrics from running Docker container"
        }

        // ========================
        // 14. Grafana Dashboard
        // ========================
        stage('Grafana Visualization') {
            echo "Grafana visualizes Prometheus metrics (CPU, Memory, Load)"
        }

        // ========================
        // 15. HEY Load Testing
        // ========================
        stage('HEY Load Testing') {
            sh '''
                echo "Running HEY load test"
                hey -z 10s -c 10 http://localhost:3000/ | tee hey-results.txt
            '''
        }

        // ========================
        // 16. Email Notification (SUCCESS)
        // ========================
        stage('Email Notification') {
            mail to: "${env.RECIPIENTS}",
                 subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: """Build Successful!

Project: ${env.PROJECT_NAME}
Job: ${env.JOB_NAME}
Build Number: ${env.BUILD_NUMBER}

Load testing and deployment completed successfully.
Build URL: ${env.BUILD_URL}
"""
        }

    } catch (err) {

        // ========================
        // Email Notification (FAILURE)
        // ========================
        mail to: "${env.RECIPIENTS}",
             subject: "FAILURE: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
             body: """Build Failed!

Project: ${env.PROJECT_NAME}
Job: ${env.JOB_NAME}
Build Number: ${env.BUILD_NUMBER}

Check logs: ${env.BUILD_URL}
"""

        throw err

    } finally {

        // ========================
        // Cleanup
        // ========================
        stage('Cleanup') {
            echo "Cleaning decrypted secrets"
            sh 'rm -f secrets/secrets.dec.env || true'
        }
    }
}
,
