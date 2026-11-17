
pipeline {

    agent none

    options {
        skipDefaultCheckout(true)
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        IMAGE_REPO = 'marwan304/smart-inventory-system'
    }

    stages {

        // ==========================================
        // PART 1: CI - UBUNTU SERVER
        // ==========================================

        stage('CI on Ubuntu Server') {

            agent {
                label 'ubuntu-server'
            }

            stages {

                // 1. Checkout
                stage('Checkout') {
                    steps {
                        checkout scm

                        echo 'Source code downloaded successfully!'
                    }
                }

                // 2. Verify server tools
                stage('Check Server Tools') {
                    steps {
                        sh '''
                            set -e

                            echo "===== SERVER ====="
                            hostname
                            whoami

                            echo "===== DOCKER ====="
                            docker --version
                            docker info --format '{{.ServerVersion}}'

                            echo "===== TRIVY ====="
                            trivy --config "" --version

                            echo "===== PROJECT ====="
                            ls -la
                        '''
                    }
                }

                // 3. Install dependencies and run tests
                stage('Run Tests') {
                    steps {
                        sh '''
                            set -e

                            echo "Running tests with Python 3.11..."

                            docker run --rm \
                                --user "$(id -u):$(id -g)" \
                                -e MONGODB_URI= \
                                -e FLASK_ENV=testing \
                                -v "$WORKSPACE:/app" \
                                -w /app \
                                python:3.11-slim \
                                sh -c "python -m venv /tmp/test-venv && \
                                /tmp/test-venv/bin/pip install \
                                    --no-cache-dir \
                                    -r requirements.txt \
                                    pytest pytest-cov && \
                                mkdir -p junit && \
                                /tmp/test-venv/bin/pytest \
                                    tests/ \
                                    --doctest-modules \
                                    --junitxml=junit/test-results.xml \
                                    --cov=backend \
                                    --cov-report=xml"
                        '''
                    }

                    post {
                        always {
                            junit(
                                testResults: 'junit/test-results.xml',
                                allowEmptyResults: true
                            )

                            archiveArtifacts(
                                artifacts: 'coverage.xml',
                                allowEmptyArchive: true
                            )
                        }
                    }
                }

                // 4. Build Docker image
                stage('Build Docker Image') {
                    steps {
                        sh '''
                            set -e

                            echo "Building Docker image..."

                            docker build --pull \
                                -t "$IMAGE_REPO:$BUILD_NUMBER" \
                                -t "$IMAGE_REPO:latest" .

                            echo "Docker image built successfully!"

                            docker images "$IMAGE_REPO"
                        '''
                    }
                }

                // 5. Security scan
                stage('Security Scan') {
                    steps {
                        sh '''
                            set -e

                            echo "Scanning Docker image using Trivy..."

                            trivy --config "" image \
                                --scanners vuln \
                                --severity HIGH,CRITICAL \
                                --exit-code 1 \
                                --ignore-unfixed \
                                "$IMAGE_REPO:$BUILD_NUMBER"

                            echo "Security scan passed!"
                        '''
                    }
                }

                // 6. Push to Docker Hub
                stage('Push to Docker Hub') {
                    steps {
                        withCredentials([
                            usernamePassword(
                                credentialsId: 'dockerhub-credentials',
                                usernameVariable: 'DOCKERHUB_USERNAME',
                                passwordVariable: 'DOCKERHUB_TOKEN'
                            )
                        ]) {
                            sh '''
                                set -eu
                                set +x

                                trap 'docker logout >/dev/null 2>&1 || true' EXIT

                                echo "$DOCKERHUB_TOKEN" | docker login \
                                    --username "$DOCKERHUB_USERNAME" \
                                    --password-stdin

                                docker push "$IMAGE_REPO:$BUILD_NUMBER"

                                docker push "$IMAGE_REPO:latest"

                                echo "Image pushed successfully!"
                            '''
                        }
                    }
                }
            }
        }

        // ==========================================
        // PART 2: CD - UBUNTU CLIENT
        // ==========================================

        stage('Deploy on Ubuntu Client') {

            agent {
                label 'ubuntu-client'
            }

            stages {

                // 7. Verify client agent
                stage('Verify Client Agent') {
                    steps {
                        sh '''
                            set -eu

                            echo "===== CLIENT AGENT ====="
                            hostname
                            whoami

                            echo "===== DOCKER ====="
                            docker --version
                            docker info --format '{{.ServerVersion}}'

                            echo "===== RUNNING CONTAINERS ====="
                            docker ps

                            echo "===== MONGODB CA CERTIFICATE ====="
                            test -r /home/marwan/mongodb-ca.crt

                            ls -l /home/marwan/mongodb-ca.crt

                            echo "Client agent is ready!"
                        '''
                    }
                }

                // 8. Pull Docker image
                stage('Pull Docker Image') {
                    steps {
                        withCredentials([
                            usernamePassword(
                                credentialsId: 'dockerhub-credentials',
                                usernameVariable: 'DOCKERHUB_USERNAME',
                                passwordVariable: 'DOCKERHUB_TOKEN'
                            )
                        ]) {
                            sh '''
                                set -eu
                                set +x

                                trap 'docker logout >/dev/null 2>&1 || true' EXIT

                                echo "$DOCKERHUB_TOKEN" | docker login \
                                    --username "$DOCKERHUB_USERNAME" \
                                    --password-stdin

                                echo "Pulling approved Docker image..."

                                docker pull "$IMAGE_REPO:$BUILD_NUMBER"

                                echo "Image pulled successfully!"
                            '''
                        }
                    }
                }

                // 9. Deploy application container
                stage('Deploy Application') {

                    steps {

                        withCredentials([
                            string(
                                credentialsId: 'smart-inventory-mongodb-uri',
                                variable: 'MONGODB_URI'
                            ),
                            string(
                                credentialsId: 'smart-inventory-secret-key',
                                variable: 'APP_SECRET_KEY'
                            )
                        ]) {

                            sh '''
                                set -eu
                                set +x

                                echo "===== CHECKING MONGODB CA ====="

                                test -r /home/marwan/mongodb-ca.crt

                                echo "MongoDB CA certificate found!"

                                echo "===== REMOVING PREVIOUS CONTAINER ====="

                                docker rm -f smart-inventory || true

                                echo "===== STARTING APPLICATION ====="

                                docker run -d \
                                    --name smart-inventory \
                                    --restart unless-stopped \
                                    -p 5002:5000 \
                                    --add-host mongo1.test:192.168.56.101 \
                                    -v /home/marwan/mongodb-ca.crt:/etc/ssl/certs/mongodb-ca.crt:ro \
                                    -e PORT=5000 \
                                    -e FLASK_DEBUG=False \
                                    -e SECRET_KEY="$APP_SECRET_KEY" \
                                    -e MONGODB_URI="$MONGODB_URI" \
                                    -e MONGODB_DATABASE=smart_inventory \
                                    -e SSL_CERT_FILE=/etc/ssl/certs/mongodb-ca.crt \
                                    -e MONGODB_TLS_ALLOW_INVALID=false \
                                    "$IMAGE_REPO:$BUILD_NUMBER"

                                echo "Application container started!"

                                docker ps --filter name=smart-inventory
                            '''
                        }
                    }
                }

                // 10. Verify application AND MongoDB
                stage('Verify Deployment') {
                    steps {
                        sh '''
                            set -eu

                            echo "===== VERIFYING APPLICATION AND MONGODB ====="

                            python3 - <<'PY'
import json
import time
import urllib.request

url = "http://127.0.0.1:5002/api/health"

for attempt in range(1, 31):

    try:
        with urllib.request.urlopen(url, timeout=5) as response:

            data = json.loads(response.read().decode())

            app_status = data.get("status")
            mongodb = data.get("mongodb", {})
            db_status = mongodb.get("status")

            print("Application status:", app_status)
            print("MongoDB status:", db_status)

            if app_status == "online" and db_status == "connected":

                print("================================")
                print("APPLICATION: HEALTHY")
                print("MONGODB: CONNECTED")
                print("DEPLOYMENT: SUCCESS")
                print("================================")

                break

            raise RuntimeError(
                "Application is responding but MongoDB is not connected: "
                + str(mongodb)
            )

    except Exception as error:

        if attempt == 30:
            print("Deployment verification failed:", error)
            raise

        print(
            "Waiting for application and MongoDB...",
            attempt,
            "/30"
        )

        time.sleep(2)

PY

                            echo "Deployment verification completed!"
                        '''
                    }
                }
            }
        }
    }

    // ==========================================
    // PIPELINE RESULT
    // ==========================================

    post {

        success {
            echo '========================================'
            echo 'CI/CD PIPELINE COMPLETED SUCCESSFULLY!'
            echo '========================================'
            echo 'Tests passed.'
            echo 'Docker image built.'
            echo 'Trivy security scan passed.'
            echo 'Image pushed to Docker Hub.'
            echo 'Application deployed on Ubuntu Client.'
            echo 'MongoDB TLS connection verified.'
        }

        failure {
            echo '========================================'
            echo 'PIPELINE FAILED!'
            echo 'Check the failed stage logs.'
            echo '========================================'
        }
    }
}
