# Movie Website - CI/CD Deployment Guide

## Overview
This guide provides step-by-step instructions for deploying the Movie Website project using your college's CI/CD pipeline infrastructure (Jenkins, SonarQube, Nexus, and Kubernetes).

## Prerequisites

### Infrastructure Requirements
- Jenkins server with the following plugins:
  - SonarQube Scanner
  - Docker Pipeline
  - Kubernetes CLI
  - Nexus Artifact Uploader
- SonarQube server for code quality analysis
- Nexus repository manager for Docker registry
- Kubernetes cluster for deployment

### Required Credentials in Jenkins
1. **Nexus Docker Registry Credentials** (`nexus-docker-creds`)
   - Username/Password for Nexus Docker registry
2. **Kubernetes Config** (`kubeconfig`)
   - Kubeconfig file for cluster access
3. **SonarQube Token** (configured in SonarQube environment)

## Step-by-Step Deployment Process

### Step 1: Prepare Your Environment

#### 1.1 Configure Jenkins Credentials
```bash
# In Jenkins > Manage Jenkins > Manage Credentials
# Add the following credentials:

# 1. Nexus Docker Registry
ID: nexus-docker-creds
Type: Username with password
Username: [your-nexus-username]
Password: [your-nexus-password]

# 2. Kubernetes Config
ID: kubeconfig
Type: Secret file
File: [path-to-your-kubeconfig-file]
```

#### 1.2 Configure SonarQube in Jenkins
```bash
# In Jenkins > Manage Jenkins > Configure System > SonarQube servers
Name: SonarQube
Server URL: http://[your-sonarqube-server]:9000
Server authentication token: [your-sonar-token]
```

### Step 2: Update Configuration Files

#### 2.1 Update Jenkinsfile Environment Variables
Edit `movie-website/Jenkinsfile` and update these variables:
```groovy
environment {
    DOCKER_REGISTRY = "[your-nexus-server]:5000"  // e.g., "nexus.college.edu:5000"
    IMAGE_NAME = "movie-website"
    SONAR_PROJECT_KEY = "movie-website"
    KUBECONFIG_CREDENTIAL_ID = "kubeconfig"
    DOCKER_CREDENTIAL_ID = "nexus-docker-creds"
}
```

#### 2.2 Update SonarQube Configuration
Edit `movie-website/sonar-project.properties`:
```properties
sonar.projectKey=movie-website
sonar.projectName=Movie Website
# Update other properties as needed
```

#### 2.3 Update Kubernetes Deployment
Edit `movie-website/k8s/deployment.yaml`:
```yaml
# Update the image reference
image: [your-nexus-server]:5000/movie-website:latest
```

### Step 3: Set Up Git Repository

#### 3.1 Initialize Git Repository
```bash
cd movie-website
git init
git add .
git commit -m "Initial commit: Movie website with CI/CD configuration"
```

#### 3.2 Push to Remote Repository
```bash
# Add your college's Git server as remote
git remote add origin [your-git-repository-url]
git push -u origin main
```

### Step 4: Create Jenkins Pipeline Job

#### 4.1 Create New Pipeline Job
1. Open Jenkins dashboard
2. Click "New Item"
3. Enter job name: `movie-website-pipeline`
4. Select "Pipeline" and click "OK"

#### 4.2 Configure Pipeline
```groovy
# In Pipeline section, select "Pipeline script from SCM"
SCM: Git
Repository URL: [your-git-repository-url]
Branch: */main
Script Path: movie-website/Jenkinsfile
```

### Step 5: Prepare Kubernetes Cluster

#### 5.1 Create Namespace (Optional)
```bash
kubectl create namespace movie-website
```

#### 5.2 Set up Ingress Controller (if not already installed)
```bash
# Install NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml
```

#### 5.3 Add DNS Entry (for local testing)
```bash
# Add to /etc/hosts (Linux/Mac) or C:\Windows\System32\drivers\etc\hosts (Windows)
[your-kubernetes-node-ip] movie-website.local
```

### Step 6: Run the Pipeline

#### 6.1 Trigger Build
1. Go to Jenkins dashboard
2. Click on `movie-website-pipeline`
3. Click "Build Now"

#### 6.2 Monitor Pipeline Stages
The pipeline will execute these stages:
1. **Checkout** - Pull source code from Git
2. **SonarQube Analysis** - Code quality analysis
3. **Quality Gate** - Wait for SonarQube quality gate
4. **Build Docker Image** - Create Docker image
5. **Push to Nexus** - Upload image to registry
6. **Deploy to Kubernetes** - Deploy to cluster
7. **Verify Deployment** - Check deployment status

### Step 7: Access Your Application

#### 7.1 Check Deployment Status
```bash
# Check if pods are running
kubectl get pods -l app=movie-website

# Check service
kubectl get service movie-website

# Check ingress
kubectl get ingress movie-website-ingress
```

#### 7.2 Access the Website
- **Via NodePort**: `http://[node-ip]:30080`
- **Via Ingress**: `http://movie-website.local`
- **Via Port Forward**: 
  ```bash
  kubectl port-forward service/movie-website 8080:80
  # Then access: http://localhost:8080
  ```

## Troubleshooting

### Common Issues and Solutions

#### 1. Docker Build Fails
```bash
# Check Docker daemon is running
sudo systemctl status docker

# Check Dockerfile syntax
docker build -t test-image movie-website/
```

#### 2. SonarQube Analysis Fails
```bash
# Check SonarQube server connectivity
curl -u [token]: http://[sonarqube-server]:9000/api/system/status

# Verify sonar-project.properties
cat movie-website/sonar-project.properties
```

#### 3. Kubernetes Deployment Fails
```bash
# Check cluster connectivity
kubectl cluster-info

# Check deployment logs
kubectl logs deployment/movie-website

# Describe deployment for events
kubectl describe deployment movie-website
```

#### 4. Image Pull Errors
```bash
# Check if image exists in registry
curl -u [username]:[password] http://[nexus-server]:5000/v2/movie-website/tags/list

# Check image pull secrets
kubectl get secrets
```

### Monitoring and Maintenance

#### Health Checks
```bash
# Check application health
kubectl get pods -l app=movie-website
kubectl top pods -l app=movie-website

# Check service endpoints
kubectl get endpoints movie-website
```

#### Scaling
```bash
# Scale deployment
kubectl scale deployment movie-website --replicas=5

# Check horizontal pod autoscaler (if configured)
kubectl get hpa
```

#### Updates
```bash
# Trigger new deployment
# Push changes to Git repository
# Jenkins will automatically trigger new build

# Manual rollback if needed
kubectl rollout undo deployment/movie-website
```

## Security Considerations

1. **Use non-root containers** - Already configured in deployment.yaml
2. **Resource limits** - Set in deployment.yaml
3. **Network policies** - Consider implementing for production
4. **Secrets management** - Use Kubernetes secrets for sensitive data
5. **Image scanning** - Configure in Nexus for vulnerability scanning

## Performance Optimization

1. **Resource requests/limits** - Tune based on actual usage
2. **Horizontal Pod Autoscaler** - Configure based on CPU/memory metrics
3. **Ingress caching** - Configure nginx ingress for static content caching
4. **CDN integration** - Consider for image assets

## Backup and Recovery

1. **Database backups** - If you add a database later
2. **Configuration backups** - Keep Git repository as source of truth
3. **Disaster recovery** - Document cluster recreation procedures

## Next Steps

1. **Add monitoring** - Implement Prometheus/Grafana
2. **Add logging** - Implement ELK stack or similar
3. **Add tests** - Unit tests, integration tests
4. **Security scanning** - Implement SAST/DAST tools
5. **Performance testing** - Load testing with tools like JMeter

## Support

For issues related to:
- **Jenkins**: Contact your college's DevOps team
- **Kubernetes**: Check cluster documentation
- **Application**: Review application logs and this guide

---

**Last Updated**: November 2024
**Version**: 1.0.0