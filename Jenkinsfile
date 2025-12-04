pipeline {
    agent any

    tools {
        jdk "java17"
        nodejs "Node18"
    }

    environment {
        SONAR_HOST_URL = 'https://v2code.rtwohealthcare.com'

        // --- DOCKER REGISTRY FIX START ---
        // 1. DOCKER_REGISTRY_URL: Used for 'docker login' and 'docker logout' (Host:Port only)
        // Set to the correct host and non-standard port (9064 is used here based on your daemon.json)
        DOCKER_REGISTRY_URL = "v2deploy.rtwohealthcare.com:9064"
        
        // 2. REGISTRY_HOST: Used for 'docker tag', 'docker push', and 'docker pull' (Host:Port/Path)
        REGISTRY_PATH = "/repository/docker-hosted"
        REGISTRY_HOST = "${DOCKER_REGISTRY_URL}${REGISTRY_PATH}"

        IMAGE_NAME = "test-v1"
        IMAGE_TAG  = "v${BUILD_NUMBER}"
        // --- DOCKER REGISTRY FIX END ---
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Run Tests') {
            steps {
                sh '''
                    if ! npm ls jest-environment-jsdom >/dev/null 2>&1; then
                        npm install jest-environment-jsdom@29.7.0 --no-save
                    fi

                    npm run test:coverage
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withCredentials([string(credentialsId: 'test', variable: 'SONAR_TOKEN')]) {
                    withSonarQubeEnv('SonarQube') {
                        sh """
                            npx sonar-scanner \\
                                -Dsonar.projectKey=test \\
                                -Dsonar.sources=. \\
                                -Dsonar.host.url=${SONAR_HOST_URL} \\
                                -Dsonar.token=$SONAR_TOKEN \\
                                -Dsonar.exclusions=node_modules/**,coverage/**,**/*.test.js \\
                                -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info \\
                                -Dsonar.coverage.exclusions=**/*.test.js
                        """
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

        stage('Docker Build') {
            steps {
                sh """
                    # Build local image
                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .

                    # Tag for Nexus registry using the full REGISTRY_HOST path
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${REGISTRY_HOST}/${IMAGE_NAME}:${IMAGE_TAG}
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${REGISTRY_HOST}/${IMAGE_NAME}:latest
                """
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-docker-cred',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {
                    sh """
                        # Login using DOCKER_REGISTRY_URL (Host:Port)
                        echo "$PASS" | docker login ${DOCKER_REGISTRY_URL} -u "$USER" --password-stdin

                        # Push using the full REGISTRY_HOST path
                        docker push ${REGISTRY_HOST}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${REGISTRY_HOST}/${IMAGE_NAME}:latest

                        # Logout using DOCKER_REGISTRY_URL (Host:Port)
                        docker logout ${DOCKER_REGISTRY_URL}
                    """
                }
            }
        }

        stage('Deploy Container') {
            steps {
                sh """
                    docker rm -f test-v1 || true

                    docker pull ${REGISTRY_HOST}/${IMAGE_NAME}:${IMAGE_TAG}

                    docker run -d \\
                        --name test-v1 \\
                        -p 3000:3000 \\
                        ${REGISTRY_HOST}/${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }
    }
}
