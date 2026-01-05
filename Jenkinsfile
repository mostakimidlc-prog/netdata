pipeline {
    agent any
    
    environment {
        DOCKER_HUB_CREDS = credentials('dockerhub-creds')
        DOCKER_IMAGE = 'mostakimidlc/netdata'
        DOCKER_TAG = "${BUILD_NUMBER}"
        K8S_NAMESPACE = 'netdata-prod'
        GIT_REPO = 'https://github.com/mostakimidlc-prog/netdata.git'
    }
    
    stages {
        stage('Checkout') {
            steps {
                script {
                    echo '========================================='
                    echo 'Stage 1/8: Checking out code'
                    echo '========================================='
                    checkout scm
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    echo '========================================='
                    echo 'Stage 2/8: Building Docker image'
                    echo '========================================='
                    sh """
                        docker build -f Dockerfile.production \
                        -t ${DOCKER_IMAGE}:${DOCKER_TAG} \
                        -t ${DOCKER_IMAGE}:latest .
                        
                        echo "Built image: ${DOCKER_IMAGE}:${DOCKER_TAG}"
                    """
                }
            }
        }
        
        stage('Test Docker Image') {
            steps {
                script {
                    echo '========================================='
                    echo 'Stage 3/8: Testing Docker image'
                    echo '========================================='
                    sh """
                        # Stop any existing test container
                        docker stop netdata-test-${BUILD_NUMBER} 2>/dev/null || true
                        docker rm netdata-test-${BUILD_NUMBER} 2>/dev/null || true
                        
                        # Run container for testing
                        docker run -d --name netdata-test-${BUILD_NUMBER} \
                          -p 19998:19999 \
                          ${DOCKER_IMAGE}:${DOCKER_TAG}
                        
                        # Wait for startup
                        echo "Waiting for Netdata to start..."
                        sleep 30
                        
                        # Test API
                        echo "Testing API endpoint..."
                        curl -f http://localhost:19998/api/v1/info || exit 1
                        
                        echo "✅ Image test passed!"
                        
                        # Cleanup
                        docker stop netdata-test-${BUILD_NUMBER}
                        docker rm netdata-test-${BUILD_NUMBER}
                    """
                }
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                script {
                    echo '========================================='
                    echo 'Stage 4/8: Pushing to Docker Hub'
                    echo '========================================='
                    sh """
                        echo "Logging in to Docker Hub..."
                        echo \${DOCKER_HUB_CREDS_PSW} | docker login -u \${DOCKER_HUB_CREDS_USR} --password-stdin
                        
                        echo "Pushing ${DOCKER_IMAGE}:${DOCKER_TAG}..."
                        docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                        
                        echo "Pushing ${DOCKER_IMAGE}:latest..."
                        docker push ${DOCKER_IMAGE}:latest
                        
                        echo "✅ Images pushed successfully!"
                    """
                }
            }
        }
        
        stage('Update K8s Manifest') {
            steps {
                script {
                    echo '========================================='
                    echo 'Stage 5/8: Updating Kubernetes manifest'
                    echo '========================================='
                    sh """
                        sed -i 's|image: ${DOCKER_IMAGE}:.*|image: ${DOCKER_IMAGE}:${DOCKER_TAG}|g' \
                          /opt/netdata-project/kubernetes/02-deployment.yaml
                        
                        echo "Updated manifest with image tag: ${DOCKER_TAG}"
                        grep "image:" /opt/netdata-project/kubernetes/02-deployment.yaml
                    """
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo '========================================='
                    echo 'Stage 6/8: Deploying to Kubernetes'
                    echo '========================================='
                    sh """
                        echo "Applying Kubernetes manifests..."
                        kubectl apply -f /opt/netdata-project/kubernetes/01-namespace.yaml
                        kubectl apply -f /opt/netdata-project/kubernetes/04-configmap.yaml
                        kubectl apply -f /opt/netdata-project/kubernetes/02-deployment.yaml
                        kubectl apply -f /opt/netdata-project/kubernetes/03-service.yaml
                        kubectl apply -f /opt/netdata-project/kubernetes/05-hpa.yaml || true
                        
                        echo "Waiting for rollout to complete..."
                        kubectl rollout status deployment/netdata -n ${K8S_NAMESPACE} --timeout=5m
                        
                        echo "✅ Deployment successful!"
                    """
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    echo '========================================='
                    echo 'Stage 7/8: Verifying deployment'
                    echo '========================================='
                    sh """
                        # Check pods
                        echo "Checking pods..."
                        kubectl get pods -n ${K8S_NAMESPACE}
                        
                        # Wait for pods to be ready
                        echo "Waiting for pods to be ready..."
                        kubectl wait --for=condition=ready pod -l app=netdata -n ${K8S_NAMESPACE} --timeout=300s
                        
                        echo "✅ Deployment verified!"
                    """
                }
            }
        }
        
        stage('Health Check') {
            steps {
                script {
                    echo '========================================='
                    echo 'Stage 8/8: Running health check'
                    echo '========================================='
                    sh """
                        # Get pod IP
                        POD_IP=\$(kubectl get pod -n ${K8S_NAMESPACE} -l app=netdata -o jsonpath='{.items[0].status.podIP}')
                        echo "Testing API at pod IP: \$POD_IP"
                        
                        # Test API (with retries)
                        for i in {1..5}; do
                            if curl -f http://\$POD_IP:19999/api/v1/info; then
                                echo "✅ Health check passed!"
                                break
                            else
                                echo "Retry \$i/5..."
                                sleep 10
                            fi
                        done
                        
                        # Show final status
                        echo ""
                        echo "Final deployment status:"
                        kubectl get all -n ${K8S_NAMESPACE}
                    """
                }
            }
        }
    }
    
    post {
        success {
            script {
                echo '========================================='
                echo '✅ PIPELINE SUCCEEDED!'
                echo '========================================='
                sh """
                    echo "Build Number: ${BUILD_NUMBER}"
                    echo "Image: ${DOCKER_IMAGE}:${DOCKER_TAG}"
                    echo "Deployment Details:"
                    kubectl get all -n ${K8S_NAMESPACE}
                    echo ""
                    echo "Access Netdata at: http://localhost:30199"
                    echo "(Use port-forward if needed)"
                """
            }
        }
        failure {
            script {
                echo '========================================='
                echo '❌ PIPELINE FAILED!'
                echo '========================================='
                sh """
                    echo "Build Number: ${BUILD_NUMBER}"
                    echo "Failed at stage. Check logs above."
                """
            }
        }
        always {
            script {
                echo 'Cleaning up...'
                sh """
                    docker logout || true
                    docker system prune -f || true
                """
            }
        }
    }
}
