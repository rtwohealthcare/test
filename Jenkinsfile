pipeline {
    agent any

    tools {
        jdk "java17"
        nodejs "Node18"
    }

    environment {
        SONAR_HOST_URL = 'https://v2code.rtwohealthcare.com'

        // FIXED: Registry must be host:port of Nexus registry
        //REGISTRY_URL  = "v2dock.rtwohealthcare.com:9062"
        REGISTRY_URL  = "https://v2deploy.rtwohealthcare.com/repository/docker-hosted:9064"

        IMAGE_NAME = 'test-v1'
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
                withCredentials([string(credentialsId: 'test', variable: 'SONAR_TOKEN')]) {
                    withSonarQubeEnv('SonarQube') {
                        sh """
                            npx sonar-scanner \
                                -Dsonar.projectKey=test \
                                -Dsonar.sources=. \
                                -Dsonar.host.url=${SONAR_HOST_URL} \
                                -Dsonar.token=$SONAR_TOKEN \
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
                    docker build -t ${REGISTRY_URL}/${IMAGE_NAME}:${IMAGE_TAG} .
                    docker tag ${REGISTRY_URL}/${IMAGE_NAME}:${IMAGE_TAG} ${REGISTRY_URL}/${IMAGE_NAME}:latest
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
                        echo "$PASS" | docker login ${REGISTRY_URL} -u "$USER" --password-stdin

                        docker push ${REGISTRY_URL}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${REGISTRY_URL}/${IMAGE_NAME}:latest

                        docker logout ${REGISTRY_URL}
                    """
                }
            }
        }

        stage('Deploy Container') {
            steps {
                sh """
                    docker rm -f test-v1 || true

                    docker pull ${REGISTRY_URL}/${IMAGE_NAME}:${IMAGE_TAG}

                    docker run -d \
                        --name test-v1 \
                        -p 3000:3000 \
                        ${REGISTRY_URL}/${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }
    }
}










