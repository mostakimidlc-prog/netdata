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
                    echo 'Stage 1/9: Checking out code'
                    echo '========================================='
                    checkout scm
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    echo '========================================='
                    echo 'Stage 2/9: Building Docker image'
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
                    echo 'Stage 3/9: Testing Docker image'
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
        
        stage('Security Scan') {
            steps {
                script {
                    echo '========================================='
                    echo 'Stage 4/9: Running security scan'
                    echo '========================================='
                    sh """
                        # Install trivy if not exists
                        if ! command -v trivy &> /dev/null; then
                            echo "Installing Trivy..."
                            wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
                            echo "deb https://aquasecurity.github.io/trivy-repo/deb \$(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
                            sudo apt-get update
                            sudo apt-get install trivy -y
                        fi
                        
                        # Scan image (don't fail build on vulnerabilities for now)
                        echo "Scanning ${DOCKER_IMAGE}:${DOCKER_TAG}..."
                        trivy image --severity HIGH,CRITICAL ${DOCKER_IMAGE}:${DOCKER_TAG} || true
                    """
                }
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                script {
                    echo '========================================='
                    echo 'Stage 5/9: Pushing to Docker Hub'
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
                    echo 'Stage 6/9: Updating Kubernetes manifest'
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
                    echo 'Stage 7/9: Deploying to Kubernetes'
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
                    echo 'Stage 8/9: Verifying deployment'
                    echo '========================================='
                    sh """
                        # Check pods
                        echo "Checking pods..."
                        kubectl get pods -n ${K8S_NAMESPACE}
                        
                        # Wait for pods to be ready
                        echo "Waiting for pods to be ready..."
                        kubectl wait --for=condition=ready pod -l app=netdata -n ${K8S_NAMESPACE} --timeout=300s
                        
                        # Get pod details
                        echo "Pod details:"
                        kubectl describe pods -n ${K8S_NAMESPACE} -l app=netdata | grep -A 5 "Status:"
                        
                        echo "✅ Deployment verified!"
                    """
                }
            }
        }
        
        stage('Health Check') {
            steps {
                script {
                    echo '========================================='
                    echo 'Stage 9/9: Running health check'
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
                    echo "Checking logs..."
                    kubectl get pods -n ${K8S_NAMESPACE} || true
                    kubectl logs -n ${K8S_NAMESPACE} -l app=netdata --tail=50 || true
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
