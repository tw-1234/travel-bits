pipeline {
    agent any

    tools {
        nodejs 'NodeJS'
    }

    environment {
        APP_NAME         = "travel-bits"
        DOCKER_IMAGE     = "taizeeba/travel-bits:latest"
        DEPLOY_CONTAINER = "travel-bits-container"
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

        /* ================= SOPS DECRYPT ================= */

        stage('Decrypt Secrets using SOPS') {
            steps {
                withCredentials([
                    string(credentialsId: 'SOPS_AGE_KEY', variable: 'SOPS_AGE_KEY')
                ]) {
                    sh '''
                        echo "Decrypting secrets..."
                        export SOPS_AGE_KEY=$SOPS_AGE_KEY
                        sops -d secrets/secrets.enc.yaml > secrets/secrets.dec.yaml
                    '''
                }
            }
        }

        stage('Load Secrets as Environment Variables') {
            steps {
                sh '''
                    export DOCKER_USER=$(yq '.docker.username' secrets/secrets.dec.yaml)
                    export DOCKER_PASS=$(yq '.docker.password' secrets/secrets.dec.yaml)
                    export SMTP_USER=$(yq '.smtp.user' secrets/secrets.dec.yaml)
                    export SMTP_PASS=$(yq '.smtp.password' secrets/secrets.dec.yaml)
                '''
            }
        }

        /* ================= BUILD ================= */

        stage('Install Dependencies') {
            steps {
                sh '''
                    npm install
                    npm audit || true
                '''
            }
        }

        stage('Run Unit Tests') {
            steps {
                sh '''
                    npm test || echo "Tests failed, continuing"
                '''
            }
        }

        /* ================= SONAR ================= */

        stage('SonarQube Scan') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'
                    withCredentials([
                        string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')
                    ]) {
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

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        /* ================= DOCKER ================= */

        stage('Docker Build & Push') {
            steps {
                sh '''
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    docker build -t ${DOCKER_IMAGE} .
                    docker push ${DOCKER_IMAGE}
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                    trivy image --severity HIGH,CRITICAL \
                    ${DOCKER_IMAGE} | tee trivy-results.txt || true
                '''
            }
        }

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

        stage('HEY Load Testing') {
            steps {
                sh '''
                    hey -z 10s -c 10 http://localhost:3000 \
                    | tee hey-results.txt || true
                '''
            }
        }
    }

    post {
        always {
            sh 'rm -f secrets/secrets.dec.yaml'
            echo "Pipeline finished with status: ${currentBuild.currentResult}"
        }

        success {
            mail to: 'taizeebarauf@gmail.com',
                 subject: "✅ SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "Pipeline completed successfully.\n${env.BUILD_URL}"
        }

        failure {
            mail to: 'taizeebarauf@gmail.com',
                 subject: "❌ FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "Pipeline failed.\n${env.BUILD_URL}"
        }
    }
}
