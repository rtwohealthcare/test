pipeline {
    agent any

    tools {
        nodejs "Node18"
    }

    environment {
        SONAR_HOST_URL = 'http://10.80.5.127:9000'

        REGISTRY_URL = '10.80.5.127:8082'
        REGISTRY_REPO = 'demo'
        IMAGE_NAME = 'demo-app'
        IMAGE_TAG = "v${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Install Dependencies') {
            steps { sh 'npm install' }
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
                withCredentials([string(credentialsId: 'sonar-test-v1', variable: 'SONAR_TOKEN')]) {
                    withSonarQubeEnv('SonarQube') {
                        sh """
                            npx sonar-scanner \
                                -Dsonar.projectKey=demo \
                                -Dsonar.sources=. \
                                -Dsonar.host.url=${SONAR_HOST_URL} \
                                -Dsonar.token=${SONAR_TOKEN} \
                                -Dsonar.exclusions=node_modules/**,coverage/**,**/*.test.js \
                                -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info \
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
                    docker build -t ${REGISTRY_URL}/${REGISTRY_REPO}/${IMAGE_NAME}:${IMAGE_TAG} .
                    docker tag ${REGISTRY_URL}/${REGISTRY_REPO}/${IMAGE_NAME}:${IMAGE_TAG} \
                              ${REGISTRY_URL}/${REGISTRY_REPO}/${IMAGE_NAME}:latest
                """
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-nexus',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {
                    sh """
                        echo "${PASS}" | docker login ${REGISTRY_URL} -u "${USER}" --password-stdin
                        docker push ${REGISTRY_URL}/${REGISTRY_REPO}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${REGISTRY_URL}/${REGISTRY_REPO}/${IMAGE_NAME}:latest
                    """
                }
            }
        }

        stage('Deploy Container') {
            steps {
                sh """
                    docker rm -f demo-app || true

                    docker pull ${REGISTRY_URL}/${REGISTRY_REPO}/${IMAGE_NAME}:${IMAGE_TAG}

                    docker run -d \
                        --name demo-app \
                        -p 3000:3000 \
                        ${REGISTRY_URL}/${REGISTRY_REPO}/${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }
    }
}


