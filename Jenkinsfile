pipeline {
    agent any
    environment {
        IMAGE_NAME = "netdata-custom"
    }
    stages {
        stage('Code Analysis') {
            steps {
                echo "Analyzing code from repository..."
                sh 'ls -lah'
            }
        }
        stage('Docker Build') {
            steps {
                sh "docker build -t $IMAGE_NAME:$BUILD_NUMBER -f Dockerfile.dev ."
            }
        }
        stage('Security Scan') {
            steps {
                echo "Scanning for vulnerabilities..."
                // In a real prod environment, you would run 'trivy' here
            }
        }
    }
    post {
        success {
            echo "Build successful! Image $IMAGE_NAME:$BUILD_NUMBER is ready."
        }
    }
}
