pipeline {
    agent any

    tools {
        jdk "java17"
        nodejs "Node18"
    }

    environment {
        // ---------------- SONAR ----------------
        SONAR_HOST_URL = 'https://v2code.rtwohealthcare.com'

        // ---------------- DOCKER / NEXUS ----------------
        DOCKER_REGISTRY = "v2deploy.rtwohealthcare.com"
        DOCKER_REPO     = "docker-hosted"

        IMAGE_NAME = "test-v1"
        IMAGE_TAG  = "v${BUILD_NUMBER}"
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
                        sh '''
                            npx sonar-scanner \
                              -Dsonar.projectKey=test \
                              -Dsonar.sources=. \
                              -Dsonar.host.url=$SONAR_HOST_URL \
                              -Dsonar.token=$SONAR_TOKEN \
                              -Dsonar.exclusions=node_modules/**,coverage/**,**/*.test.js \
                              -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info \
                              -Dsonar.coverage.exclusions=**/*.test.js
                        '''
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

        // ---------------- DOCKER BUILD ----------------
        stage('Docker Build') {
            steps {
                sh '''
                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .

                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} \
                      ${DOCKER_REGISTRY}/${DOCKER_REPO}/${IMAGE_NAME}:${IMAGE_TAG}

                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} \
                      ${DOCKER_REGISTRY}/${DOCKER_REPO}/${IMAGE_NAME}:latest
                '''
            }
        }

        // ---------------- DOCKER PUSH ----------------
        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-repo',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login ${DOCKER_REGISTRY} \
                          -u "$DOCKER_USER" --password-stdin

                        docker push ${DOCKER_REGISTRY}/${DOCKER_REPO}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${DOCKER_REGISTRY}/${DOCKER_REPO}/${IMAGE_NAME}:latest

                        docker logout ${DOCKER_REGISTRY}
                    '''
                }
            }
        }

        // ---------------- DEPLOY (FIXED) ----------------
        stage('Deploy Container') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-repo',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login ${DOCKER_REGISTRY} \
                          -u "$DOCKER_USER" --password-stdin

                        docker rm -f test-v1 || true

                        docker pull ${DOCKER_REGISTRY}/${DOCKER_REPO}/${IMAGE_NAME}:${IMAGE_TAG}

                        docker run -d \
                          --name test-v1 \
                          -p 3000:3000 \
                          ${DOCKER_REGISTRY}/${DOCKER_REPO}/${IMAGE_NAME}:${IMAGE_TAG}

                        docker logout ${DOCKER_REGISTRY}
                    '''
                }
            }
        }
    }
}
