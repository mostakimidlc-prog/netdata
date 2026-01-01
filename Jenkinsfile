pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = 'netdata-custom'
        DOCKER_TAG = "${BUILD_NUMBER}"
        DOCKER_REGISTRY = 'docker.io'  // Change if using private registry
        DOCKER_CREDENTIALS = 'dockerhub-credentials'  // Jenkins credential ID
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code...'
                checkout scm
                sh 'git --version'
                sh 'git log -1'
            }
        }
        
        stage('Environment Info') {
            steps {
                echo 'Gathering environment information...'
                sh 'docker --version'
                sh 'docker compose version'
                sh 'free -h'
                sh 'df -h'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                echo "Building Docker image: ${DOCKER_IMAGE}:${DOCKER_TAG}"
                script {
                    sh "docker build -f Dockerfile.custom -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
                    sh "docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest"
                }
            }
        }
        
        stage('Test Image') {
            steps {
                echo 'Testing Docker image...'
                script {
                    // Start container for testing
                    sh """
                        docker run -d --name netdata-test-${BUILD_NUMBER} \
                            -p 19997:19999 \
                            ${DOCKER_IMAGE}:${DOCKER_TAG}
                    """
                    
                    // Wait for service to be ready
                    sh 'sleep 10'
                    
                    // Run tests
                    sh 'chmod +x scripts/test.sh'
                    sh 'scripts/test.sh || true'
                    
                    // Cleanup test container
                    sh "docker stop netdata-test-${BUILD_NUMBER} || true"
                    sh "docker rm netdata-test-${BUILD_NUMBER} || true"
                }
            }
        }
        
        stage('Security Scan') {
            steps {
                echo 'Running security scan...'
                script {
                    // Using Trivy for vulnerability scanning
                    sh """
                        docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                            aquasec/trivy image --severity HIGH,CRITICAL \
                            ${DOCKER_IMAGE}:${DOCKER_TAG} || true
                    """
                }
            }
        }
        
        stage('Push to Registry') {
            when {
                branch 'main'  // Only push from main branch
            }
            steps {
                echo 'Pushing image to registry...'
                script {
                    // Uncomment when you set up Docker Hub credentials
                    /*
                    withCredentials([usernamePassword(
                        credentialsId: DOCKER_CREDENTIALS,
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh "echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} --password-stdin"
                        sh "docker push ${DOCKER_IMAGE}:${DOCKER_TAG}"
                        sh "docker push ${DOCKER_IMAGE}:latest"
                    }
                    */
                    echo 'Image push skipped - configure Docker Hub credentials first'
                }
            }
        }
        
        stage('Deploy to Dev') {
            steps {
                echo 'Deploying to development environment...'
                script {
                    sh """
                        docker compose -f docker-compose.dev.yml down || true
                        docker compose -f docker-compose.dev.yml up -d
                    """
                }
            }
        }
    }
    
    post {
        success {
            echo 'Pipeline completed successfully!'
            sh "docker images | grep ${DOCKER_IMAGE}"
        }
        failure {
            echo 'Pipeline failed!'
        }
        always {
            echo 'Cleaning up...'
            sh 'docker system prune -f'
        }
    }
}
