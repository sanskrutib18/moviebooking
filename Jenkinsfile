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
        DOCKER_REGISTRY      = "nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085"

        IMAGE_NAME           = "2401014_moviebooking/movie-website"
        IMAGE_TAG            = "${env.BUILD_NUMBER}"
        FULL_IMAGE_NAME      = "${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"

        SONAR_HOST           = "http://my-sonarqube-sonarqube.sonarqube.svc.cluster.local:9000"
        SONAR_PROJECT_KEY    = "2401014-moviebooking"
        SONAR_PROJECT_NAME   = "2401014"
        SONAR_TOKEN          = "sqp_34d6a677093e00da2be6a55db046c1dac3735a46"

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

        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
                container('dind') {
                    sh """
                        echo "=== Building Docker image ==="
                        # Dockerfile is at repo root, so use '.' as context
                        docker build -t ${FULL_IMAGE_NAME} .
                        docker tag ${FULL_IMAGE_NAME} ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest
                    """
                }
            }
        }

        stage('Push to Nexus Registry') {
            steps {
                echo 'Pushing Docker image to Nexus registry...'
                container('dind') {
                    sh """
                        echo "Logging in to Nexus Docker Registry..."
                        echo "Imcc@2025" | docker login -u student --password-stdin ${DOCKER_REGISTRY}

                        echo "Pushing image..."
                        docker push ${FULL_IMAGE_NAME}
                        docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                echo 'Deploying to Kubernetes cluster...'
                container('kubectl') {
                    withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIAL_ID}", variable: 'KUBECONFIG')]) {
                        sh """
                            # Update deployment image
                            sed -i "s|image:.*movie-website:latest|image: ${FULL_IMAGE_NAME}|g" movie-website/k8s/deployment.yaml

                            # Apply Kubernetes manifests to your namespace
                            kubectl apply -f movie-website/k8s/deployment.yaml -n 2401014
                            kubectl apply -f movie-website/k8s/service.yaml -n 2401014

                            # Wait for deployment to be ready
                            kubectl rollout status deployment/movie-website -n 2401014 --timeout=300s

                            # Get service information
                            kubectl get services movie-website -n 2401014
                        """
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
                            kubectl get pods -n 2401014 -l app=movie-website
                            kubectl describe deployment movie-website -n 2401014
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
