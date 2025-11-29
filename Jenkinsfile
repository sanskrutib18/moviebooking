pipeline {
    agent any
    
    environment {
        // Update these values according to your college setup
        DOCKER_REGISTRY = "localhost:5000"  // Nexus Docker registry
        IMAGE_NAME = "movie-website"
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        FULL_IMAGE_NAME = "${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
        SONAR_PROJECT_KEY = "2401014-moviebooking"
        KUBECONFIG_CREDENTIAL_ID = "kubeconfig"
        DOCKER_CREDENTIAL_ID = "nexus-docker-creds"
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }
        
        stage('Code Quality - SonarQube Analysis') {
            steps {
                echo 'Running SonarQube analysis...'
                script {
                    // Use SonarQube scanner
                    withSonarQubeEnv('SonarQube') {
                        sh '''
                            sonar-scanner \
                            -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                            -Dsonar.sources=movie-website \
                            -Dsonar.host.url=${SONAR_HOST_URL} \
                            -Dsonar.login=${SONAR_AUTH_TOKEN}
                        '''
                    }
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                echo 'Waiting for SonarQube Quality Gate...'
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
                script {
                    dir('movie-website') {
                        sh "docker build -t ${FULL_IMAGE_NAME} ."
                        sh "docker tag ${FULL_IMAGE_NAME} ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest"
                    }
                }
            }
        }
        
        stage('Push to Nexus Registry') {
            steps {
                echo 'Pushing Docker image to Nexus registry...'
                script {
                    withCredentials([usernamePassword(
                        credentialsId: "${DOCKER_CREDENTIAL_ID}",
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh '''
                            echo $DOCKER_PASS | docker login ${DOCKER_REGISTRY} -u $DOCKER_USER --password-stdin
                            docker push ${FULL_IMAGE_NAME}
                            docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest
                        '''
                    }
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                echo 'Deploying to Kubernetes cluster...'
                script {
                    withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIAL_ID}", variable: 'KUBECONFIG')]) {
                        sh '''
                            # Update deployment image
                            sed -i "s|your-docker-repo/movie-website:latest|${FULL_IMAGE_NAME}|g" movie-website/k8s/deployment.yaml
                            
                            # Apply Kubernetes manifests
                            kubectl apply -f movie-website/k8s/deployment.yaml
                            kubectl apply -f movie-website/k8s/service.yaml
                            
                            # Wait for deployment to be ready
                            kubectl rollout status deployment/movie-website --timeout=300s
                            
                            # Get service information
                            kubectl get services movie-website
                        '''
                    }
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                echo 'Verifying deployment...'
                script {
                    withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIAL_ID}", variable: 'KUBECONFIG')]) {
                        sh '''
                            # Check if pods are running
                            kubectl get pods -l app=movie-website
                            
                            # Check deployment status
                            kubectl describe deployment movie-website
                        '''
                    }
                }
            }
        }
    }
    
    post {
        always {
            echo 'Cleaning up...'
            sh '''
                # Clean up local Docker images to save space
                docker rmi ${FULL_IMAGE_NAME} || true
                docker rmi ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest || true
            '''
        }
        success {
            echo "✅ Pipeline succeeded! Image: ${FULL_IMAGE_NAME}"
            echo "🚀 Application deployed successfully to Kubernetes"
        }
        failure {
            echo "❌ Pipeline failed. Check logs for details."
        }
        unstable {
            echo "⚠️ Pipeline completed with warnings."
        }
    }
}
