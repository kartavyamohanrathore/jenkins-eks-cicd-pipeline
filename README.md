# Automated CI/CD Pipeline: GitHub → Jenkins → Docker Hub → AWS EKS

An end-to-end CI/CD pipeline that builds a Docker image on every code push and deploys it to a Kubernetes cluster on Amazon EKS using a zero-downtime rolling update.

![CI/CD Pipeline Flow](docs/cicd_pipeline_flow.png)

## Highlights

- Automated application builds and deployments to AWS EKS using Jenkins, Docker, GitHub and Kubernetes
- Streamlined Docker image publishing, Kubernetes rolling updates and deployment verification, reducing manual deployment effort
- Automated server configuration and setup using Ansible, reducing provisioning time from approximately 1 hour to under 10 minutes

## Tech Stack

| Area | Tools |
|---|---|
| Source control | GitHub (webhook trigger) |
| CI/CD | Jenkins (declarative pipeline) |
| Containers | Docker, Docker Hub |
| Cloud | AWS (EC2, EKS, IAM) |
| Orchestration | Kubernetes, kubectl, eksctl |
| Web server | Apache2 inside the container |
| OS | Ubuntu Server 24.04 LTS |

## How It Works

1. A developer pushes code to GitHub.
2. The GitHub webhook triggers the Jenkins pipeline.
3. Jenkins clones the code and builds a Docker image.
4. The image is tagged with the Jenkins build number and `latest`, then pushed to Docker Hub.
5. Jenkins authenticates with AWS and configures kubectl for the EKS cluster.
6. Kubernetes manifests are applied and the Deployment image is updated to the new tag.
7. Kubernetes performs a rolling update, so the application is updated without downtime.
8. Jenkins waits for the rollout to complete and reports success or failure.

## Infrastructure

| Component | Details |
|---|---|
| Dev server | EC2 `c7i-flex.large`, Ubuntu Server 24.04 LTS. Hosts the website source and Dockerfile |
| Jenkins master | EC2 `c7i-flex.large`, Ubuntu Server 24.04 LTS. Runs Jenkins, Docker, AWS CLI, kubectl, eksctl |
| EKS cluster | `cluster1` in `ap-south-1`, custom configuration (EKS Auto Mode disabled), managed node group |
| Application | 2 replicas behind a Kubernetes `LoadBalancer` service |

## Pipeline Stages

| Stage | What it does |
|---|---|
| Clone Code | Pulls the latest source from GitHub |
| Build Docker Image | Builds the image and tags it with the build number and `latest` |
| Push to DockerHub | Logs in with Jenkins credentials and pushes both tags |
| Configure AWS & EKS | Verifies AWS identity and runs `aws eks update-kubeconfig` |
| Deploy to EKS | Applies manifests, sets the new image, waits for `kubectl rollout status` |

Pipeline variables:

| Variable | Meaning |
|---|---|
| `DOCKER_IMAGE` | Docker Hub repository name |
| `IMAGE_TAG` | Jenkins build number, used as the image version (build 1 gives `<user>/docker-repo-5:1`) |
| `AWS_REGION` | Region where the EKS cluster exists |
| `CLUSTER_NAME` | EKS cluster name |
| `K8S_DIR` | Folder holding the Kubernetes YAML files |

---

## Setup Guide

Replace the placeholders in `<angle brackets>` with your own values.

### 1. Dev server: website and Dockerfile

```bash
cd /root
wget https://www.tooplate.com/zip-templates/2130_waso_strategy.zip
apt-get update -y
apt install unzip -y
unzip 2130_waso_strategy.zip
cd /root/2130_waso_strategy
```

Create a `Dockerfile`:

```dockerfile
FROM ubuntu
RUN apt-get update -y
RUN apt-get install -y apache2
COPY . /var/www/html
CMD ["/usr/sbin/apache2ctl", "-D", "FOREGROUND"]
```

Push the code to GitHub (create a repository first; its name must match the one in the pipeline):

```bash
apt install git -y
git init
git add *
git commit -m "Initial commit Waso Strategy Website"
git remote add origin https://github.com/<github-username>/git-repo-5.git
git push origin master
```

### 2. Jenkins master: Jenkins and Docker

```bash
apt update -y
apt install fontconfig openjdk-21-jre -y
wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | tee /etc/apt/sources.list.d/jenkins.list > /dev/null
apt update -y
apt install jenkins -y
systemctl start jenkins
```

Give the `jenkins` user sudo access (`vim /etc/sudoers`):

```
jenkins ALL=(ALL) NOPASSWD: ALL
```

Install Docker and allow Jenkins to use it:

```bash
apt install docker.io -y
systemctl start docker
systemctl enable docker
usermod -aG docker jenkins
systemctl restart jenkins
systemctl restart docker
```

Open port **8080** in the Jenkins server's security group, then open `http://<jenkins-server-public-ip>:8080`.

### 3. GitHub webhook

In the GitHub repository: **Settings → Webhooks → Add webhook**

- Payload URL: `http://<jenkins-server-public-ip>:8080/github-webhook/`
- Event: push

### 4. Amazon EKS cluster

**Create the cluster** (region `ap-south-1`):

- Select **Custom configuration** and disable **Use EKS Auto Mode**
- Name: `cluster1`
- Cluster IAM role: `AmazonEKSClusterRole`
- Kubernetes version: as available in your console (creation takes about 10 minutes)

**IAM user for the pipeline:**

- Create an IAM user (`user1`) and generate an Access Key ID and Secret Access Key

**Install AWS CLI on the Jenkins master:**

```bash
cd /root
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
apt update -y
apt install -y unzip
unzip awscliv2.zip
/root/aws/install
aws configure
```

**Create worker nodes:** EKS → Clusters → `cluster1` → Compute → Node groups → Add (about 5 minutes).

**Open port 443:** The EKS API server always uses port 443. Allow it in the cluster security group (EKS → Clusters → `cluster1` → Networking → Cluster security group).

**Connect kubectl to the cluster:**

```bash
aws eks --region ap-south-1 update-kubeconfig --name cluster1
```

**Install eksctl:**

```bash
cd /root
curl --silent --location "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz" | tar xz -C /tmp
mv /tmp/eksctl /usr/local/bin
eksctl version
```

**Install kubectl:**

```bash
cd /root
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x /root/kubectl
mv /root/kubectl /usr/local/bin/kubectl
kubectl version --client
```

**Grant the IAM user access to the cluster:**

EKS → Clusters → `cluster1` → Access → IAM access entries → Create

- IAM principal ARN: `arn:aws:iam::<aws-account-id>:user/user1`
- Access policy: `AmazonEKSClusterAdminPolicy`

```bash
aws eks associate-access-policy \
  --cluster-name cluster1 \
  --principal-arn arn:aws:iam::<aws-account-id>:user/user1 \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster
```

**Give Jenkins the kubeconfig:**

```bash
mkdir /var/lib/jenkins/.kube
cp /root/.kube/config /var/lib/jenkins/.kube
chown -R jenkins:jenkins /var/lib/jenkins/.kube
chmod 600 /var/lib/jenkins/.kube/config
aws eks update-kubeconfig --region ap-south-1 --name cluster1
```

**Verify:**

```bash
kubectl get nodes
```

### 5. Kubernetes manifests

```bash
mkdir /k8s
cd /k8s
```

`deployment.yml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: waso-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: waso-app
  template:
    metadata:
      labels:
        app: waso-app
    spec:
      containers:
      - name: waso-container
        image: <dockerhub-username>/docker-repo-5:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 80
```

`service.yml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: waso-service
spec:
  type: LoadBalancer
  selector:
    app: waso-app
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f deployment.yml
kubectl apply -f service.yml
```

### 6. Docker Hub and Jenkins credentials

1. Create a Docker Hub repository named `docker-repo-5` (it must match the name in the pipeline).
2. In Jenkins: **Manage Jenkins → Credentials → Add → Username with Password**. Add your Docker Hub username and password with ID `dockerhub`.
3. In Jenkins: **Manage Jenkins → Plugins → Available plugins**, search for **AWS Credentials** and install it.
4. In Jenkins: **Manage Jenkins → Credentials → Add → AWS Credentials**. Add the Access Key and Secret Access Key of `user1` with ID `aws-cred`.

### 7. Jenkins pipeline job

1. Create a new Jenkins job of type **Pipeline**.
2. Add the GitHub repository URL and enable the **GitHub hook trigger**.
3. Paste the pipeline script below.

```groovy
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "<dockerhub-username>/docker-repo-5"
        IMAGE_TAG = "${BUILD_NUMBER}"
        AWS_REGION = "ap-south-1"
        CLUSTER_NAME = "cluster1"
        K8S_DIR = "/k8s"
    }

    stages {

        stage('Clone Code') {
            steps {
                git 'https://github.com/<github-username>/git-repo-5.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                docker build -t $DOCKER_IMAGE:$IMAGE_TAG .
                docker tag $DOCKER_IMAGE:$IMAGE_TAG $DOCKER_IMAGE:latest
                """
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                    echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                    docker push $DOCKER_IMAGE:$IMAGE_TAG
                    docker push $DOCKER_IMAGE:latest
                    """
                }
            }
        }

        stage('Configure AWS & EKS') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-cred'
                ]]) {
                    sh """
                    export AWS_DEFAULT_REGION=$AWS_REGION

                    aws sts get-caller-identity

                    aws eks update-kubeconfig \
                        --region $AWS_REGION \
                        --name $CLUSTER_NAME
                    """
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-cred'
                ]]) {
                    sh """
                    export AWS_DEFAULT_REGION=$AWS_REGION

                    kubectl apply -f /$K8S_DIR/ --validate=false

                    kubectl set image deployment/waso-deployment \
                        waso-container=$DOCKER_IMAGE:$IMAGE_TAG

                    kubectl rollout status deployment/waso-deployment
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Deployment Successful"
        }
        failure {
            echo "Deployment Failed"
        }
    }
}
```

---

## Testing the Pipeline

1. **Manual run:** Run the Jenkins job once, then open the load balancer DNS name in a browser.
   ```bash
   kubectl get svc waso-service
   ```
2. **Automatic run:** Edit `index.html` on the dev server, then:
   ```bash
   git add .
   git commit -m "Update index.html"
   git push origin master
   ```
   The webhook triggers Jenkins, a new image is built and rolled out, and the updated site appears at the load balancer DNS.
3. **Optional:** Attach the load balancer DNS to a custom domain.

### Checking both replicas

The Deployment runs 2 pods behind the load balancer. To change the page inside a pod for a quick demo:

```bash
kubectl get pods
kubectl exec -it <pod-name> -- /bin/bash
apt update -y && apt install -y vim
vim /var/www/html/index.html
```

Repeat for the second pod and refresh the load balancer URL to see traffic served by different pods. These edits are temporary and are replaced on the next deployment.

## Rollback

```bash
kubectl rollout history deployment/waso-deployment
kubectl rollout undo deployment/waso-deployment
```

Every image is tagged with its build number, so any previous version can also be redeployed by setting that tag.

## Security Notes and Possible Improvements

- Use a least-privilege IAM policy for the pipeline user instead of Administrator access
- Restrict the `jenkins` sudo entry and avoid running day-to-day work as root
- Never commit access keys, passwords or kubeconfig files to GitHub; keep them in Jenkins credentials
- Restrict the Jenkins (8080) security group to trusted IP addresses and add a webhook secret
- Store the Kubernetes manifests and the `Jenkinsfile` in the repository, and use **Pipeline script from SCM**, so the whole pipeline is version controlled
- Add automated tests and an image vulnerability scan stage before pushing the image

## Key Learnings

- Building a complete CI/CD pipeline from commit to production on Kubernetes
- Containerizing an application and versioning images by build number
- Configuring Amazon EKS, IAM access entries and kubectl access for Jenkins
- Zero-downtime deployments with Kubernetes rolling updates
- Automating server provisioning to cut setup time
