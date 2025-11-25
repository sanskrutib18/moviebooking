pipeline {
    agent any
    environment {
        IMAGE = "your-docker-repo/movie-website:${env.BUILD_NUMBER}"
        DOCKER_REGISTRY = "your-docker-repo"
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                // Requires Jenkins SonarQube plugin and a configured SonarQube server named 'SonarQube'
                withSonarQubeEnv('SonarQube') {
                    sh 'sonar-scanner -Dproject.settings=sonar-project.properties'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${IMAGE} movie-website"
            }
        }

        stage('Push Docker Image') {
            steps {
                // Configure credentials in Jenkins and replace 'docker-creds' with your credentials id
                withCredentials([usernamePassword(credentialsId: 'docker-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin '
                    sh "docker push ${IMAGE}"
                }
            }
        }
    }
    post {
        success {
            echo "Pipeline succeeded. Image: ${IMAGE}"
        }
        failure {
            echo 'Pipeline failed.'
        }
    }
}
