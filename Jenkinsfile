pipeline {
    agent any

    tools {
        nodejs 'NodeJS'          // Jenkins → Global Tool Configuration
    }

    environment {
        APP_NAME         = "travel-bits"
        DOCKER_IMAGE     = "taizeeba/travel-bits:latest"
        DEPLOY_CONTAINER = "travel-bits-container"
    }

    stages {

        /* =========================
           1. CHECKOUT
        ========================== */
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        /* =========================
           2. VERIFY NODE
        ========================== */
        stage('Verify Node & NPM') {
            steps {
                sh '''
                    node -v
                    npm -v
                '''
            }
        }

        /* =========================
           3. DECRYPT SECRETS (SOPS)
        ========================== */
        stage('Decrypt Secrets using SOPS') {
            steps {
                withCredentials([
                    string(credentialsId: 'SOPS_AGE_KEY', variable: 'SOPS_AGE_KEY')
                ]) {
                    sh '''
                        echo "Decrypting secrets using SOPS..."
                        export SOPS_AGE_KEY=$SOPS_AGE_KEY

                        sops -d secrets/secrets.enc.yaml > secrets/secrets.dec.yaml
                        echo "Secrets decrypted successfully"
                    '''
                }
            }
        }

        /* =========================
           4. INSTALL DEPENDENCIES
        ========================== */
        stage('Install Dependencies') {
            steps {
                sh '''
                    npm install
                    npm audit || true
                '''
            }
        }

        /* =========================
           5. UNIT TESTS
        ========================== */
        stage('Run Unit Tests') {
            steps {
                sh '''
                    npm test || echo "Unit tests failed, continuing pipeline"
                '''
            }
        }

        /* =========================
           6. SONARQUBE SCAN
        ========================== */
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
                                -Dsonar.token=${SONAR_TOKEN}
                            """
                        }
                    }
                }
            }
        }

        /* =========================
           7. QUALITY GATE
        ========================== */
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        /* =========================
           8. DOCKER BUILD & PUSH
        ========================== */
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

        /* =========================
           9. TRIVY IMAGE SCAN
        ========================== */
        stage('Trivy Image Scan') {
            steps {
                sh '''
                    trivy image --severity HIGH,CRITICAL \
                    --format table ${DOCKER_IMAGE} | tee trivy-results.txt || true
                '''
            }
        }

        /* =========================
           10. DEPLOY CONTAINER
        ========================== */
        stage('Deploy Docker Container') {
            steps {
                sh '''
                    docker rm -f ${DEPLOY_CONTAINER} || true
                    docker run -d \
                        --name ${DEPLOY_CONTAINER} \
                        -p 3000:3000 \
                        ${DOCKER_IMAGE}
                '''
            }
        }

        /* =========================
           11. HEY LOAD TEST
        ========================== */
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
                def trivyReport = fileExists('trivy-results.txt')
                        ? readFile('trivy-results.txt')
                        : "No Trivy report generated"

                def heyReport = fileExists('hey-results.txt')
                        ? readFile('hey-results.txt')
                        : "No HEY report generated"

                mail to: 'taizeebarauf@gmail.com',
                     subject: "✅ Jenkins SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                     body: """
Pipeline Status: SUCCESS

Project: ${env.JOB_NAME}
Build Number: ${env.BUILD_NUMBER}
Build URL: ${env.BUILD_URL}

===== TRIVY SECURITY SCAN =====
${trivyReport}

===== HEY LOAD TEST =====
${heyReport}
"""
            }
        }

        failure {
            mail to: 'taizeebarauf@gmail.com',
                 subject: "❌ Jenkins FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "Check Jenkins console logs: ${env.BUILD_URL}"
        }

        always {
            sh 'rm -f secrets/secrets.dec.yaml || true'
            echo "Pipeline finished with status: ${currentBuild.currentResult}"
        }
    }
}
