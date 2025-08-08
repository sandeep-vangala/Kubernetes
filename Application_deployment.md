Using Jenkins, Terraform, AWS, Nexus for Artifactory, Sonarqube for Vulnerability Scanning, ECR, EKS, Helm, ARGO CD

Below is a detailed project plan for deploying a Java-based microservices application on AWS using Jenkins, Terraform, Nexus, SonarQube, Amazon ECR, EKS, Helm, and ArgoCD. The project focuses on building a CI/CD pipeline to automate the build, test, scan, and deployment of a Java Spring Boot application to an Amazon EKS cluster, incorporating DevSecOps practices. I’ll provide manifest files, configurations, and explanations for each component, tailored to a scalable and secure setup.

### Project Overview: Deploying a Java Microservices Application

**Objective**: Build and deploy a Spring Boot-based Java application (e.g., a simple REST API for a "To-Do" service) to an Amazon EKS cluster using a CI/CD pipeline. The pipeline will:
- Use Jenkins for Continuous Integration (CI) to build, test, and push Docker images.
- Use Terraform to provision AWS infrastructure (VPC, EKS, ECR).
- Use Nexus for storing Maven artifacts.
- Use SonarQube for code quality and vulnerability scanning.
- Use Amazon ECR to store Docker images.
- Use Helm and ArgoCD for Continuous Deployment (CD) to EKS.

**Architecture**:
- **Application**: A Spring Boot REST API with endpoints to manage a to-do list, using Maven for dependency management.
- **Infrastructure**: AWS VPC with public/private subnets, EKS cluster, and ECR repository.
- **CI/CD Pipeline**:
  - **CI**: Jenkins builds the Java app, runs tests, scans with SonarQube, packages it into a Docker image, and pushes it to ECR.
  - **CD**: ArgoCD monitors a Git repository with Helm charts and deploys the application to EKS.
- **Monitoring**: Basic setup for observability using Helm-installed Prometheus (optional, mentioned briefly).

**Prerequisites**:
- AWS account with admin privileges (or sufficient IAM permissions).
- Jenkins server installed on an EC2 instance or container.
- Nexus repository set up for Maven artifacts.
- SonarQube server running (can be on EC2 or containerized).
- GitHub repository for source code and Helm charts.
- Tools installed: `kubectl`, `aws-cli`, `terraform`, `helm`, `docker`.

---

### Step-by-Step Project Plan and Manifest Files

#### 1. **Spring Boot Application**
Create a simple Spring Boot application with a REST API for managing to-do items.

<xaiArtifact artifact_id="04c03a39-62d3-4a1f-8de9-2302047a86a7" artifact_version_id="a73d323c-edcd-4451-8592-4c40ab80b74a" title="pom.xml" contentType="text/xml">
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>todo-app</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>

    <properties>
        <java.version>17</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

    <repositories>
        <repository>
            <id>nexus</id>
            <name>Nexus Repository</name>
            <url>http://<nexus-host>:8081/repository/maven-public/</url>
        </repository>
    </repositories>

    <distributionManagement>
        <repository>
            <id>nexus</id>
            <name>Nexus Release Repository</name>
            <url>http://<nexus-host>:8081/repository/maven-releases/</url>
        </repository>
        <snapshotRepository>
            <id>nexus</id>
            <name>Nexus Snapshot Repository</name>
            <url>http://<nexus-host>:8081/repository/maven-snapshots/</url>
        </snapshotRepository>
    </distributionManagement>
</project>
</xaiArtifact>

**Explanation**:
- **POM File**: Defines the Spring Boot project with dependencies for a web application and testing. Configures Nexus as the repository for dependencies and artifact uploads.
- **Nexus Integration**: Replace `<nexus-host>` with your Nexus server’s IP or DNS. Ensure Nexus credentials are added to `~/.m2/settings.xml`:
  ```xml
  <servers>
      <server>
          <id>nexus</id>
          <username>admin</username>
          <password>your-nexus-password</password>
      </server>
  </servers>
  ```

<xaiArtifact artifact_id="a1612011-2c75-46e4-b3c1-b3cba650fbe0" artifact_version_id="574949e1-a8fd-49d6-926d-0b70aaa93367" title="TodoController.java" contentType="text/x-java-source">
package com.example.todoapp;

import org.springframework.web.bind.annotation.*;

import java.util.ArrayList;
import java.util.List;

@RestController
@RequestMapping("/api/todos")
public class TodoController {
    private List<String> todos = new ArrayList<>();

    @GetMapping
    public List<String> getTodos() {
        return todos;
    }

    @PostMapping
    public String addTodo(@RequestBody String todo) {
        todos.add(todo);
        return todo;
    }

    @DeleteMapping("/{index}")
    public String deleteTodo(@PathVariable int index) {
        return todos.remove(index);
    }
}
</xaiArtifact>

**Explanation**:
- **REST API**: Provides endpoints to list (`GET /api/todos`), add (`POST /api/todos`), and delete (`DELETE /api/todos/{index}`) to-do items.
- **Location**: Place in `src/main/java/com/example/todoapp/`.

<xaiArtifact artifact_id="138621c0-0fce-476d-80af-15acb36e78df" artifact_version_id="d568659d-19b5-44f1-bb68-09a8812928b4" title="Dockerfile" contentType="text/plain">
FROM eclipse-temurin:17-jdk-alpine
WORKDIR /app
COPY target/todo-app-1.0.0-SNAPSHOT.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
</xaiArtifact>

**Explanation**:
- **Dockerfile**: Builds a Docker image for the Spring Boot app using a lightweight Java 17 base image. Copies the compiled JAR and runs it.
- **Location**: Place in the project root.

#### 2. **Terraform Infrastructure**
Use Terraform to provision a VPC, EKS cluster, and ECR repository.

<xaiArtifact artifact_id="530e9427-549a-43c9-b299-8b1fb111e505" artifact_version_id="e8d6ff64-1cec-4461-adfd-c457cd4135a9" title="main.tf" contentType="text/x-terraform">
provider "aws" {
  region = "us-east-1"
}

# VPC
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"

  name = "todo-vpc"
  cidr = "10.0.0.0/16"
  azs  = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]
  enable_nat_gateway = true
  single_nat_gateway = true
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    "kubernetes.io/cluster/todo-eks" = "shared"
  }
}

# ECR Repository
resource "aws_ecr_repository" "todo_app" {
  name                 = "todo-app"
  image_tag_mutability = "MUTABLE"
  image_scanning_configuration {
    scan_on_push = true
  }
}

# EKS Cluster
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "19.15.3"

  cluster_name    = "todo-eks"
  cluster_version = "1.27"
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnets

  eks_managed_node_groups = {
    default = {
      min_size     = 2
      max_size     = 4
      desired_size = 2
      instance_types = ["t3.medium"]
    }
  }

  tags = {
    Environment = "production"
  }
}

# IAM Role for EKS Node Group
resource "aws_iam_role_policy_attachment" "eks_worker_policy" {
  role       = module.eks.eks_managed_node_groups["default"].iam_role_name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
}

resource "aws_iam_role_policy_attachment" "eks_cni_policy" {
  role       = module.eks.eks_managed_node_groups["default"].iam_role_name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
}

resource "aws_iam_role_policy_attachment" "ecr_read_policy" {
  role       = module.eks.eks_managed_node_groups["default"].iam_role_name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
}
</xaiArtifact>

**Explanation**:
- **Provider**: Sets AWS region to `us-east-1`.
- **VPC**: Creates a VPC with public and private subnets for EKS, using the Terraform AWS VPC module.
- **ECR**: Provisions a repository named `todo-app` with image scanning enabled.
- **EKS**: Sets up an EKS cluster with a managed node group (2-4 `t3.medium` instances).
- **IAM Roles**: Attaches policies for EKS worker nodes and ECR read access.
- **Execution**: Run `terraform init`, `terraform plan`, and `terraform apply` to provision.

#### 3. **Jenkins Pipeline**
Configure Jenkins to build, test, scan, and push the Docker image to ECR.

<xaiArtifact artifact_id="4942b9a8-9f56-4bd9-9284-844147955685" artifact_version_id="87f343f8-53f0-4452-88ab-3a1bd09f94cb" title="Jenkinsfile" contentType="text/plain">
pipeline {
    agent any
    environment {
        AWS_REGION = 'us-east-1'
        ECR_REGISTRY = '<your-account-id>.dkr.ecr.us-east-1.amazonaws.com'
        APP_REGISTRY = "${ECR_REGISTRY}/todo-app"
        REGISTRY_CREDENTIAL = 'aws-ecr-credentials'
        NEXUS_URL = 'http://<nexus-host>:8081/repository/maven-releases/'
        NEXUS_CREDENTIAL = 'nexus-credentials'
        SONAR_SERVER = 'http://<sonar-host>:9000'
        SONAR_TOKEN = credentials('sonar-token')
    }
    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/<your-repo>/todo-app.git', branch: 'main'
            }
        }
        stage('Build and Test') {
            steps {
                sh 'mvn clean package'
            }
        }
        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv('sonarserver') {
                    sh "mvn sonar:sonar -Dsonar.host.url=${SONAR_SERVER} -Dsonar.login=${SONAR_TOKEN}"
                }
            }
        }
        stage('Publish to Nexus') {
            steps {
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: '<nexus-host>:8081',
                    groupId: 'com.example',
                    version: '1.0.0-SNAPSHOT',
                    repository: 'maven-releases',
                    credentialsId: 'nexus-credentials',
                    artifacts: [
                        [artifactId: 'todo-app', classifier: '', file: 'target/todo-app-1.0.0-SNAPSHOT.jar', type: 'jar']
                    ]
                )
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    dockerImage = docker.build("${APP_REGISTRY}:${env.BUILD_NUMBER}")
                }
            }
        }
        stage('Push to ECR') {
            steps {
                script {
                    docker.withRegistry("https://${ECR_REGISTRY}", REGISTRY_CREDENTIAL) {
                        dockerImage.push("${env.BUILD_NUMBER}")
                        dockerImage.push('latest')
                    }
                }
            }
        }
    }
}
</xaiArtifact>

**Explanation**:
- **Environment Variables**: Defines AWS region, ECR registry, Nexus, and SonarQube details. Replace `<your-account-id>`, `<nexus-host>`, and `<sonar-host>` accordingly.
- **Stages**:
  - **Checkout**: Pulls code from GitHub.
  - **Build and Test**: Runs `mvn clean package` to build and test the app.
  - **SonarQube Scan**: Performs code quality and vulnerability scanning.
  - **Nexus Publish**: Uploads the JAR to Nexus.
  - **Docker Build/Push**: Builds and pushes the Docker image to ECR.
- **Prerequisites**:
  - Install Jenkins plugins: Git, Maven, SonarQube Scanner, Nexus Artifact Uploader, Docker Pipeline, Amazon ECR.
  - Configure AWS credentials (`aws-ecr-credentials`), Nexus credentials (`nexus-credentials`), and SonarQube token (`sonar-token`) in Jenkins.
  - Install Docker on the Jenkins server and add the `jenkins` user to the `docker` group:
    ```bash
    sudo apt update
    sudo apt install docker.io
    sudo usermod -aG docker jenkins
    sudo systemctl restart jenkins
    ```

#### 4. **Helm Chart for Deployment**
Create a Helm chart for deploying the application to EKS.

<xaiArtifact artifact_id="3aebca37-65f9-4bf4-b529-17c36d9aaab8" artifact_version_id="74537b28-e8d6-4224-a889-48c213f273e7" title="Chart.yaml" contentType="text/yaml">
apiVersion: v2
name: todo-app
description: Helm chart for To-Do App
version: 0.1.0
appVersion: "1.0.0"
</xaiArtifact>

<xaiArtifact artifact_id="da5477dd-0e60-4a70-b6e6-a5d13aceb874" artifact_version_id="4b9ffe02-b00c-4314-a18f-e54cb846ebca" title="values.yaml" contentType="text/yaml">
replicaCount: 2

image:
  repository: <your-account-id>.dkr.ecr.us-east-1.amazonaws.com/todo-app
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: LoadBalancer
  port: 80
  targetPort: 8080
</xaiArtifact>

<xaiArtifact artifact_id="2ee7e1a3-9f20-4949-ba4f-f480d35bd292" artifact_version_id="85fe2d64-20ca-4845-91db-28287c1df7ec" title="deployment.yaml" contentType="text/yaml">
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Chart.Name }}
  labels:
    app: {{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - containerPort: 8080
</xaiArtifact>

<xaiArtifact artifact_id="4cfac9e2-63ad-4934-bc69-36d1726c524f" artifact_version_id="e187760c-bac1-45ad-b3a8-ad163b451752" title="service.yaml" contentType="text/yaml">
apiVersion: v1
kind: Service
metadata:
  name: {{ .Chart.Name }}
  labels:
    app: {{ .Chart.Name }}
spec:
  type: {{ .Values.service.type }}
  ports:
  - port: {{ .Values.service.port }}
    targetPort: {{ .Values.service.targetPort }}
    protocol: TCP
  selector:
    app: {{ .Chart.Name }}
</xaiArtifact>

**Explanation**:
- **Chart.yaml**: Defines the Helm chart metadata.
- **values.yaml**: Configures the image repository, tag, and service settings. Replace `<your-account-id>` with your AWS account ID.
- **deployment.yaml**: Deploys the application with the specified replicas and image.
- **service.yaml**: Exposes the application via a LoadBalancer.
- **Location**: Place these in a `helm/todo-app/` directory in a Git repository (e.g., `helm-charts` repo).

#### 5. **ArgoCD Configuration**
Use ArgoCD for GitOps-based deployment.

<xaiArtifact artifact_id="93dd029e-708e-4489-8689-032e667db751" artifact_version_id="77d9de13-d5e4-4026-9671-99b195be7899" title="argocd-app.yaml" contentType="text/yaml">
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: todo-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-repo>/helm-charts.git
    targetRevision: HEAD
    path: helm/todo-app
    helm:
      valueFiles:
      - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
</xaiArtifact>

**Explanation**:
- **ArgoCD Application**: Defines an application that monitors the Helm chart in the `helm-charts` Git repository and deploys it to the `default` namespace in EKS.
- **Setup**:
  - Install ArgoCD on EKS:
    ```bash
    kubectl create namespace argocd
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    ```
  - Apply the ArgoCD application manifest:
    ```bash
    kubectl apply -f argocd-app.yaml
    ```
  - Update `values.yaml` in the `helm-charts` repo with the latest image tag (`BUILD_NUMBER`) from Jenkins.

#### 6. **Pipeline Flow**
1. **Developer**: Pushes code changes to the GitHub repository (`todo-app`).
2. **Jenkins**:
   - Triggers on code push (via webhook).
   - Builds and tests the Java app with Maven.
   - Runs SonarQube for code quality and vulnerability scanning.
   - Publishes the JAR to Nexus.
   - Builds a Docker image and pushes it to ECR.
   - Updates the `values.yaml` in the `helm-charts` repo with the new image tag.
3. **ArgoCD**:
   - Detects changes in the `helm-charts` repo.
   - Deploys the updated Helm chart to EKS.
4. **EKS**: Runs the application, accessible via the LoadBalancer’s public URL.

#### 7. **Setup Instructions**
- **Nexus**:
  - Deploy Nexus on an EC2 instance or use a hosted service.
  - Create `maven-public`, `maven-releases`, and `maven-snapshots` repositories.
  - Configure Jenkins with Nexus credentials.
- **SonarQube**:
  - Deploy SonarQube on EC2 or a container.
  - Generate a token in SonarQube and add it to Jenkins credentials.
  - Configure the SonarQube server in Jenkins (`Manage Jenkins > Configure System`).
- **Jenkins**:
  - Install on an EC2 instance with Terraform or manually.
  - Install plugins: Git, Maven, SonarQube Scanner, Nexus Artifact Uploader, Docker Pipeline, Amazon ECR.
  - Configure AWS CLI and credentials for ECR access.
- **EKS**:
  - Update `kubeconfig` after Terraform provisioning:
    ```bash
    aws eks update-kubeconfig --region us-east-1 --name todo-eks
    ```
- **ArgoCD**:
  - Access the ArgoCD UI:
    ```bash
    kubectl port-forward svc/argocd-server -n argocd 8080:443
    ```
  - Log in with the default admin password (retrieve with `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`).

#### 8. **Testing and Validation**
- **Access the App**: Get the LoadBalancer URL:
  ```bash
  kubectl get svc -n default
  ```
  Test endpoints:
  ```bash
  curl <loadbalancer-url>/api/todos
  curl -X POST <loadbalancer-url>/api/todos -d "Buy groceries"
  ```
- **SonarQube**: Check the dashboard for code quality reports.
- **ArgoCD**: Verify deployment status in the ArgoCD UI.
- **Nexus**: Confirm the JAR is uploaded to the `maven-releases` repository.
- **ECR**: Verify the Docker image with the correct tag.

#### 9. **Cleanup**
To avoid costs, destroy resources:
```bash
terraform destroy
```
Manually delete ECR images and Nexus artifacts if needed.

---
