
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

                // 2. Verify tools
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

                            echo "Running tests with Python 3.10..."

                            docker run --rm \
                                --user "$(id -u):$(id -g)" \
                                -e MONGODB_URI= \
                                -e FLASK_ENV=testing \
                                -v "$WORKSPACE:/app" \
                                -w /app \
                                python:3.10-slim \
                                sh -c '
                                    python -m venv /tmp/test-venv

                                    /tmp/test-venv/bin/pip install \
                                        --no-cache-dir \
                                        -r requirements.txt \
                                        pytest pytest-cov

                                    mkdir -p junit

                                    /tmp/test-venv/bin/pytest \
                                        tests/ \
                                        --doctest-modules \
                                        --junitxml=junit/test-results.xml \
                                        --cov=backend \
                                        --cov-report=xml
                                '
                        '''
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
        // PART 2: DEPLOYMENT - UBUNTU CLIENT
        // ==========================================

        stage('Deploy on Ubuntu Client') {

            agent {
                label 'ubuntu-client'
            }

            steps {

                // 7. Pull image
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
                    '''
                }

                // 8. Deploy container
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

                        echo "Removing previous container..."

                        docker rm -f smart-inventory || true

                        echo "Starting new application container..."

                        docker run -d \
                            --name smart-inventory \
                            --restart unless-stopped \
                            -p 5002:5000 \
                            -e PORT=5000 \
                            -e FLASK_DEBUG=False \
                            -e SECRET_KEY="$APP_SECRET_KEY" \
                            -e MONGODB_URI="$MONGODB_URI" \
                            -e MONGODB_DATABASE=smart_inventory \
                            "$IMAGE_REPO:$BUILD_NUMBER"

                        echo "Application deployed!"

                        docker ps
                    '''
                }

                // 9. Verify application
                stage('Verify Deployment') {
                    steps {
                        sh '''
                            python3 - <<'PY'
import time
import urllib.request

url = "http://127.0.0.1:5002/api/health"

for attempt in range(1, 31):
    try:
        with urllib.request.urlopen(url, timeout=5) as response:
            print("HTTP Status:", response.status)
            print("Application is healthy!")
            break

    except Exception as error:
        if attempt == 30:
            raise

        print("Waiting for application...", attempt)
        time.sleep(2)
PY
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'CI/CD Pipeline completed successfully!'
            echo 'Image built, tested, scanned, pushed and deployed!'
        }

        failure {
            echo 'Pipeline failed! Check the failed stage logs.'
        }
    }
}
