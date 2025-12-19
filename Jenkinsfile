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
    command: ["cat"]
    tty: true

  - name: kubectl
    image: bitnami/kubectl:latest
    command: ["cat"]
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
    securityContext:
      privileged: true
    env:
    - name: DOCKER_TLS_CERTDIR
      value: ""
    volumeMounts:
    - name: docker-config
      mountPath: /etc/docker/daemon.json
      subPath: daemon.json

  volumes:
  - name: docker-config
    configMap:
      name: docker-daemon-config
  - name: kubeconfig-secret
    secret:
      secretName: kubeconfig-secret
'''
        }
    }

    environment {
        SONAR_HOST = "http://my-sonarqube-sonarqube.sonarqube.svc.cluster.local:9000"
        REGISTRY   = "nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085"
        IMAGE      = "2401014_moviebooking/movie-website:v1"
    }

    stages {
        stage('Build Docker Image') {
            steps {
                container('dind') {
                    sh '''
                        echo "Building movie-website Docker image..."
                        sleep 10
                        docker build -t movie-website:latest .
                    '''
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                container('sonar-scanner') {
                    sh '''
                        sonar-scanner \
                          -Dsonar.projectKey=2401014-moviebooking \
                          -Dsonar.sources=. \
                          -Dsonar.exclusions=node_modules/,dist/** \
                          -Dsonar.host.url=${SONAR_HOST} \
                          -Dsonar.token=sqp_34d6a677093e00da2be6a55db046c1dac3735a46
                    '''
                }
            }
        }

        stage('Login to Nexus') {
            steps {
                container('dind') {
                    sh '''
                        docker login ${REGISTRY} \
                          -u student -p Imcc@2025
                    '''
                }
            }
        }

        stage('Tag & Push Image') {
            steps {
                container('dind') {
                    sh '''
                        docker tag movie-website:latest ${REGISTRY}/${IMAGE}
                        docker push ${REGISTRY}/${IMAGE}
                        docker image ls
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                container('kubectl') {
                    dir('k8s') {
                        sh '''
                            echo "Deploying movie-website application..."
                            kubectl apply -f deployment.yaml -n 2401014
                            kubectl apply -f service.yaml -n 2401014
                            kubectl rollout status deployment/movie-website -n 2401014 --timeout=120s
                        '''
                    }
                }
            }
        }
    }
}
