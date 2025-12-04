pipeline {
    agent any
    // ... other sections

    environment {
        // ... other env vars

        // Nexus Docker registry details
        // **CRITICAL FIX**: Explicitly set the port here (e.g., 9064)
        DOCKER_REGISTRY_URL = "v2deploy.rtwohealthcare.com:9064" 
        
        // The registry host now includes the port 
        REGISTRY_PATH = "/repository/docker-hosted"
        REGISTRY_HOST = "${DOCKER_REGISTRY_URL}${REGISTRY_PATH}" // Will be v2deploy.rtwohealthcare.com:9064/repository/docker-hosted

        IMAGE_NAME = "test-v1"
        IMAGE_TAG  = "v${BUILD_NUMBER}"
    }

    // ... rest of the pipeline
    
    stage('Docker Build') {
        steps {
            sh """
                // Tagging needs the full path including the port:
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
                    // Login uses DOCKER_REGISTRY_URL (host:port)
                    echo "$PASS" | docker login ${DOCKER_REGISTRY_URL} -u "$USER" --password-stdin
                    
                    // Pushing uses the full REGISTRY_HOST path
                    docker push ${REGISTRY_HOST}/${IMAGE_NAME}:${IMAGE_TAG}
                    docker push ${REGISTRY_HOST}/${IMAGE_NAME}:latest

                    docker logout ${DOCKER_REGISTRY_URL}
                """
            }
        }
    }
    // ...
}
