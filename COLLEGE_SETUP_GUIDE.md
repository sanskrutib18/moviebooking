# College CI/CD Pipeline Setup Guide

## Quick Configuration Checklist

### 🔧 Before You Start
- [ ] Access to college Jenkins server
- [ ] Access to college SonarQube server
- [ ] Access to college Nexus repository
- [ ] Access to college Kubernetes cluster
- [ ] Git repository access

### 📋 Configuration Values You Need to Collect

#### 1. Jenkins Server Information
```bash
Jenkins URL: http://[college-jenkins-server]:8080
Username: [your-college-username]
Password: [your-college-password]
```

#### 2. SonarQube Server Information
```bash
SonarQube URL: http://[college-sonarqube-server]:9000
Username: [your-college-username]
Token: [generate-from-sonarqube-ui]
```

#### 3. Nexus Repository Information
```bash
Nexus URL: http://[college-nexus-server]:8081
Docker Registry: [college-nexus-server]:5000
Username: [your-college-username]
Password: [your-college-password]
```

#### 4. Kubernetes Cluster Information
```bash
Cluster API Server: https://[college-k8s-master]:6443
Namespace: [your-assigned-namespace] or default
Kubeconfig: [download-from-cluster-admin]
```

#### 5. Git Repository Information
```bash
Git Server: http://[college-git-server] or https://github.com/[your-org]
Repository URL: [your-repo-url]
Branch: main or master
```

## Step-by-Step Configuration

### Step 1: Update Configuration Files

#### 1.1 Update Jenkinsfile
Replace these placeholders in `movie-website/Jenkinsfile`:

```groovy
# BEFORE (lines 5-6):
DOCKER_REGISTRY = "localhost:5000"
IMAGE_NAME = "movie-website"

# AFTER (replace with your college values):
DOCKER_REGISTRY = "[college-nexus-server]:5000"  # e.g., "nexus.college.edu:5000"
IMAGE_NAME = "movie-website"
```

#### 1.2 Update Kubernetes Deployment
Replace in `movie-website/k8s/deployment.yaml`:

```yaml
# BEFORE (line 25):
image: localhost:5000/movie-website:latest

# AFTER (replace with your college values):
image: [college-nexus-server]:5000/movie-website:latest
```

#### 1.3 Update SonarQube Properties
Update `movie-website/sonar-project.properties` if needed:

```properties
# Update project key to match your college naming convention
sonar.projectKey=[your-student-id]_movie-website
# Example: sonar.projectKey=2021CS001_movie-website
```

### Step 2: Jenkins Setup

#### 2.1 Create Jenkins Credentials
Log into Jenkins and create these credentials:

**Nexus Docker Registry Credentials:**
```
Navigate to: Jenkins > Manage Jenkins > Manage Credentials
Click: Add Credentials
Type: Username with password
ID: nexus-docker-creds
Username: [your-nexus-username]
Password: [your-nexus-password]
Description: Nexus Docker Registry Access
```

**Kubernetes Config:**
```
Navigate to: Jenkins > Manage Jenkins > Manage Credentials
Click: Add Credentials
Type: Secret file
ID: kubeconfig
File: [upload-your-kubeconfig-file]
Description: Kubernetes Cluster Access
```

#### 2.2 Configure SonarQube Integration
```
Navigate to: Jenkins > Manage Jenkins > Configure System
Find: SonarQube servers section
Click: Add SonarQube
Name: SonarQube
Server URL: http://[college-sonarqube-server]:9000
Server authentication token: [your-sonar-token]
```

#### 2.3 Create Pipeline Job
```
1. Click "New Item"
2. Enter name: [your-student-id]-movie-website
3. Select "Pipeline"
4. In Pipeline section:
   - Definition: Pipeline script from SCM
   - SCM: Git
   - Repository URL: [your-git-repo-url]
   - Branch: */main
   - Script Path: movie-website/Jenkinsfile
5. Save
```

### Step 3: SonarQube Setup

#### 3.1 Generate Authentication Token
```
1. Login to SonarQube: http://[college-sonarqube-server]:9000
2. Go to: My Account > Security
3. Generate Token:
   - Name: jenkins-integration
   - Type: User Token
   - Expires: No expiration (or as per college policy)
4. Copy the token (you'll need it for Jenkins)
```

#### 3.2 Create Project (if required)
```
1. Go to: Projects > Create Project
2. Project key: [your-student-id]_movie-website
3. Display name: Movie Website - [Your Name]
4. Set up: Manually
5. Baseline: Previous version
```

### Step 4: Nexus Repository Setup

#### 4.1 Verify Docker Registry Access
```bash
# Test login to Nexus Docker registry
docker login [college-nexus-server]:5000
Username: [your-username]
Password: [your-password]

# If successful, you should see "Login Succeeded"
```

#### 4.2 Create Repository (if needed)
```
1. Login to Nexus: http://[college-nexus-server]:8081
2. Go to: Repositories
3. Create repository > docker (hosted)
4. Name: docker-hosted
5. HTTP port: 5000
6. Enable Docker V1 API: checked
7. Save
```

### Step 5: Kubernetes Setup

#### 5.1 Verify Cluster Access
```bash
# Test kubectl access
kubectl cluster-info
kubectl get nodes
kubectl get namespaces

# If you have a specific namespace assigned:
kubectl config set-context --current --namespace=[your-namespace]
```

#### 5.2 Create Namespace (if allowed)
```bash
# Only if you're allowed to create namespaces
kubectl create namespace [your-student-id]-apps
kubectl config set-context --current --namespace=[your-student-id]-apps
```

### Step 6: Common College Environment Configurations

#### 6.1 Proxy Settings (if your college uses proxy)
Add to `movie-website/Jenkinsfile` in the environment section:

```groovy
environment {
    // ... existing variables ...
    HTTP_PROXY = "http://[proxy-server]:[port]"
    HTTPS_PROXY = "http://[proxy-server]:[port]"
    NO_PROXY = "localhost,127.0.0.1,[internal-servers]"
}
```

#### 6.2 Resource Limits (for shared environments)
Update `movie-website/k8s/deployment.yaml`:

```yaml
resources:
  requests:
    memory: "32Mi"    # Reduced for shared environment
    cpu: "25m"        # Reduced for shared environment
  limits:
    memory: "64Mi"    # Reduced for shared environment
    cpu: "50m"        # Reduced for shared environment
```

#### 6.3 Security Policies (if required)
Add security context to `movie-website/k8s/deployment.yaml`:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 1000
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: false
  capabilities:
    drop:
      - ALL
```

## Testing Your Setup

### 1. Test Docker Build Locally
```bash
cd movie-website
docker build -t test-movie-website .
docker run -p 8080:80 test-movie-website
# Visit http://localhost:8080
```

### 2. Test SonarQube Analysis
```bash
# Install SonarQube scanner locally (optional)
sonar-scanner \
  -Dsonar.projectKey=[your-project-key] \
  -Dsonar.sources=. \
  -Dsonar.host.url=http://[college-sonarqube-server]:9000 \
  -Dsonar.login=[your-token]
```

### 3. Test Kubernetes Deployment
```bash
# Apply manifests manually first
kubectl apply -f movie-website/k8s/deployment.yaml
kubectl apply -f movie-website/k8s/service.yaml

# Check status
kubectl get pods -l app=movie-website
kubectl get service movie-website
```

## Troubleshooting Common College Environment Issues

### Issue 1: Network/Firewall Restrictions
```bash
# Test connectivity to services
curl -I http://[jenkins-server]:8080
curl -I http://[sonarqube-server]:9000
curl -I http://[nexus-server]:8081

# If blocked, contact your college IT team
```

### Issue 2: Permission Denied
```bash
# Check your user permissions
kubectl auth can-i create deployments
kubectl auth can-i create services

# If denied, request permissions from cluster admin
```

### Issue 3: Resource Quotas
```bash
# Check resource quotas in your namespace
kubectl describe quota
kubectl describe limitrange

# Adjust resource requests/limits accordingly
```

### Issue 4: Image Pull Errors
```bash
# Check if image exists
curl -u [username]:[password] \
  http://[nexus-server]:5000/v2/movie-website/tags/list

# Check image pull secrets
kubectl get secrets
kubectl describe secret [secret-name]
```

## College-Specific Best Practices

### 1. Naming Conventions
- Use your student ID as prefix: `[student-id]-movie-website`
- Follow college naming standards for resources
- Use descriptive labels and annotations

### 2. Resource Management
- Always set resource limits
- Use minimal resource requests
- Clean up unused resources regularly

### 3. Security
- Never commit credentials to Git
- Use college-provided certificates
- Follow college security policies

### 4. Collaboration
- Document your setup for teammates
- Share knowledge with classmates
- Follow college code review processes

## Getting Help

### College Resources
- **IT Helpdesk**: [college-it-email] or [phone]
- **DevOps Team**: [devops-team-email]
- **Faculty Advisor**: [faculty-email]

### Documentation
- **Jenkins**: http://[jenkins-server]:8080/help
- **SonarQube**: http://[sonarqube-server]:9000/documentation
- **Nexus**: http://[nexus-server]:8081/help
- **Kubernetes**: College cluster documentation

### Common Commands Reference
```bash
# Jenkins CLI (if available)
java -jar jenkins-cli.jar -s http://[jenkins-server]:8080 help

# SonarQube CLI
sonar-scanner -h

# Docker commands
docker --help
docker login [registry]
docker push [image]

# Kubernetes commands
kubectl --help
kubectl get all
kubectl describe [resource] [name]
kubectl logs [pod-name]
```

---

**Note**: Replace all `[placeholder]` values with your actual college infrastructure details.

**Last Updated**: November 2024
**Version**: 1.0.0