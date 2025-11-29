pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:

  - name: sonar-scanner
    image: sonarsource/sonar-scanner-cli
    command: ['cat']
    tty: true

  - name: kubectl
    image: bitnami/kubectl:latest
    command: ['cat']
    tty: true
    securityContext:
      runAsUser: 0
      readOnlyRootFilesystem: false
    env:
    - name: KUBECONFIG
      value: /kube/config
    volumeMounts:
    - name: kubeconfig-secret
      mountPath: /kube/config
      subPath: kubeconfig

  - name: dind
    image: docker:dind
    args: ["--storage-driver=overlay2",
           "--insecure-registry=nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085"]
    securityContext:
      privileged: true
    env:
    - name: DOCKER_TLS_CERTDIR
      value: ""

  volumes:
  - name: kubeconfig-secret
    secret:
      secretName: kubeconfig-secret
'''
        }
    }

    environment {
        // College Nexus Docker registry (inside cluster)
        DOCKER_REGISTRY = "nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085"

        IMAGE_NAME       = "2401014_moviebooking/movie-website"
        IMAGE_TAG        = "${env.BUILD_NUMBER}"
        FULL_IMAGE_NAME  = "${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"

        // SonarQube (SQ) details for your project
        SONAR_HOST        = "http://my-sonarqube-sonarqube.sonarqube.svc.cluster.local:9000"
        SONAR_PROJECT_KEY = "2401014-moviebooking"
        SONAR_PROJECT_NAME = "2401014"
        SONAR_TOKEN       = "sqp_0f5cf6f44677d8313612fd2c2eea0d8e3ff950e5"

        KUBECONFIG_CREDENTIAL_ID = "kubeconfig-secret"
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
                container('sonar-scanner') {
                    sh '''
                        sonar-scanner \
                          -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                          -Dsonar.projectName=${SONAR_PROJECT_NAME} \
                          -Dsonar.sources=. \
                          -Dsonar.host.url=${SONAR_HOST} \
                          -Dsonar.token=${SONAR_TOKEN}
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                echo 'Waiting for SonarQube Quality Gate...'
                container('sonar-scanner') {
                    timeout(time: 5, unit: 'MINUTES') {
                        script {
                            def qg = waitForQualityGate()
                            if (qg.status != 'OK') {
                                error "Pipeline aborted due to quality gate failure: ${qg.status}"
                            }
                        }
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
                container('dind') {
                    sh """
                        echo "=== Building Docker image ==="
                        docker build -t ${FULL_IMAGE_NAME} movie-website
                        docker tag ${FULL_IMAGE_NAME} ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest
                    """
                }
            }
        }

        stage('Push to Nexus Registry') {
            steps {
                echo 'Pushing Docker image to Nexus registry...'
                container('dind') {
                    sh '''
                        echo "Logging in to Nexus Docker Registry..."
                        docker login nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085 \
                          -u student -p Imcc@2025

                        echo "Pushing image..."
                        docker push ${FULL_IMAGE_NAME}
                        docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                echo 'Deploying to Kubernetes cluster...'
                container('kubectl') {
                    withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIAL_ID}", variable: 'KUBECONFIG')]) {
                        sh '''
                            # Update deployment image
                            sed -i "s|nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085/2401014_moviebooking/movie-website:latest|${FULL_IMAGE_NAME}|g" movie-website/k8s/deployment.yaml

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
                container('kubectl') {
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
            container('dind') {
                sh '''
                    docker rmi ${FULL_IMAGE_NAME} || true
                    docker rmi ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest || true
                '''
            }
        }
        success {
            echo "Pipeline succeeded! Image: ${FULL_IMAGE_NAME}"
            echo "Application deployed successfully to Kubernetes"
        }
        failure {
            echo "Pipeline failed. Check logs for details."
        }
        unstable {
            echo "Pipeline completed with warnings."
        }
    }
}
