Nebulance: EKS Cluster Deployment
Nebulance is a production-ready deployment of a 3-tier web application on Amazon EKS, showcasing enterprise-grade DevOps practices. It leverages Terraform for infrastructure provisioning, Helm for Kubernetes orchestration, and AWS Secrets Manager for secure credential management. The project demonstrates a modern CI/CD pipeline, containerized workloads, and monitoring-ready infrastructure, making it an excellent showcase for DevOps engineers.
Features

Automated Infrastructure: Provisions an EKS cluster with Terraform, including VPC, subnets, and auto-scaling node groups.
Containerized Application: Deploys a React.js frontend, Node.js/Express backend, and PostgreSQL database using Docker and Kubernetes.
Secure Secrets Management: Integrates AWS Secrets Manager for secure storage and retrieval of database and application credentials.
CI/CD Pipeline: Includes a CircleCI configuration for automated testing and Docker image building.
Scalable Architecture: Uses Horizontal Pod Autoscalers for frontend (2-5 replicas) and backend (3-10 replicas).
Networking: Configures AWS Load Balancer Controller and External Secrets Operator for robust connectivity and secrets synchronization.

Tech Stack

Languages: JavaScript (React, Node.js), Python (scripts), Bash
Infrastructure: Terraform, AWS (EKS, VPC, Secrets Manager)
Containerization: Docker, Kubernetes, Helm
CI/CD: CircleCI, GitHub Actions (optional)
Database: PostgreSQL 15
Security: AWS Secrets Manager, JWT authentication, Helmet, CORS

Architecture
The application follows a 3-tier architecture:

Frontend: React.js with JWT-based authentication, real-time dashboard, and responsive UI.
Backend: Node.js/Express RESTful API with endpoints for user management, posts, and health checks.
Database: PostgreSQL with persistent storage and credentials managed via AWS Secrets Manager.

graph TD
    A[User] -->|HTTP| B[AWS Load Balancer]
    B --> C[Frontend: React.js]
    C -->|ClusterIP| D[Backend: Node.js/Express]
    D -->|JDBC| E[PostgreSQL]
    D -->|Secrets| F[AWS Secrets Manager]
    E -->|Secrets| F
    G[Terraform] --> H[EKS Cluster]
    H --> C
    H --> D
    H --> E
    I[CircleCI] -->|Build & Push| J[Docker Registry]
    J --> H

Prerequisites
Required Tools
Install the following tools:
# AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Terraform
wget https://releases.hashicorp.com/terraform/1.6.0/terraform_1.6.0_linux_amd64.zip
unzip terraform_1.6.0_linux_amd64.zip && sudo mv terraform /usr/local/bin/

AWS Configuration
# Configure AWS CLI
aws configure
# Enter: Access Key ID, Secret Access Key, Region (eu-central-1), Output format (json)

# Verify AWS access
aws sts get-caller-identity

Getting Started
Step 1: Build and Push Docker Images
# Build backend image
cd application/backend/
docker build -t your-registry/nebulance-app:backend-1.0.0 .
docker push your-registry/nebulance-app:backend-1.0.0

# Build frontend image
cd ../frontend/
docker build -t your-registry/nebulance-app:frontend-1.0.0 .
docker push your-registry/nebulance-app:frontend-1.0.0

Step 2: Deploy Infrastructure with Terraform
cd ../../terraform/
terraform init
terraform plan
terraform apply
aws eks update-kubeconfig --region eu-central-1 --name eks-nebulance

Step 3: Create AWS Secrets
# Generate secure secrets
JWT_SECRET=$(openssl rand -base64 32)
API_KEY=$(openssl rand -hex 16)
DB_PASSWORD=$(openssl rand -base64 20)

# Create database secrets
aws secretsmanager create-secret \
  --name "eks-app/database" \
  --description "Database credentials for EKS application" \
  --secret-string "{\"POSTGRES_USER\":\"appuser\",\"POSTGRES_PASSWORD\":\"$DB_PASSWORD\",\"POSTGRES_DB\":\"appdb\"}" \
  --region eu-central-1

# Create application secrets
aws secretsmanager create-secret \
  --name "eks-app/application" \
  --description "Application secrets for EKS application" \
  --secret-string "{\"JWT_SECRET\":\"$JWT_SECRET\",\"API_KEY\":\"$API_KEY\",\"NODE_ENV\":\"production\"}" \
  --region eu-central-1

Step 4: Install AWS Load Balancer Controller
# Download IAM policy
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json

# Create IAM policy
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

# Associate OIDC provider
eksctl utils associate-iam-oidc-provider \
  --region eu-central-1 \
  --cluster eks-nebulance \
  --approve

# Create IAM service account
ACCOUNT_ID="531807594086"  # Replace with your AWS account ID
eksctl create iamserviceaccount \
  --region eu-central-1 \
  --cluster eks-nebulance \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerController \
  --attach-policy-arn arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# Install AWS Load Balancer Controller
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=eks-nebulance \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=eu-central-1 \
  --set vpcId=vpc-04e571ce92ba6626d  # Replace with your VPC ID

Step 5: Install External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
kubectl apply -f https://raw.githubusercontent.com/external-secrets/external-secrets/main/deploy/crds/bundle.yaml
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets \
  --create-namespace \
  --set installCRDs=true

Step 6: Update Helm Chart Configuration
Edit helm-charts/values.yaml:
frontend:
  image:
    repository: your-registry/nebulance-app
    tag: "frontend-1.0.0"
  service:
    type: LoadBalancer

backend:
  image:
    repository: your-registry/nebulance-app
    tag: "backend-1.0.0"
  service:
    type: ClusterIP

Step 7: Deploy Application with Helm
cd ../helm-charts/
helm template nebulance-app . --validate
helm install nebulance-app . \
  --namespace nebulance-app \
  --create-namespace \
  --timeout 10m

Step 8: Verify Deployment
kubectl get pods -n nebulance-app
kubectl get services -n nebulance-app
FRONTEND_URL=$(kubectl get service frontend -n nebulance-app -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "Open browser to: http://$FRONTEND_URL"

Testing
# Test backend health endpoint via port-forward
kubectl port-forward service/backend 3000:3000 -n nebulance-app &
curl http://localhost:3000/health

# Run frontend tests
cd application/frontend/
npm test

# Run backend tests
cd ../backend/
npm test

Troubleshooting

EKS Cluster Issues: Verify IAM permissions (aws iam get-user) and VPC limits (aws ec2 describe-account-attributes).
Pod Failures: Check logs (kubectl logs <pod-name> -n nebulance-app) and secrets sync (kubectl describe externalsecret database-secrets -n nebulance-app).
Load Balancer Issues: Ensure security groups allow traffic (aws ec2 describe-security-groups).

Contributing
Contributions are welcome! Please fork the repository, create a feature branch, and submit a pull request following the PULL_REQUEST_TEMPLATE.
License
This project is licensed under the MIT License - see the LICENSE file for details.
