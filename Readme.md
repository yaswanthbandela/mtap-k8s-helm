I'll guide you through setting up the necessary components for your EKS cluster step-by-step. Let's start from scratch to ensure everything is properly configured.

## Step 1: Set up AWS CLI and kubectl

First, make sure you have AWS CLI and kubectl configured:

```bash
# Install AWS CLI (if not already installed)
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Configure AWS CLI
aws configure
# Enter your AWS Access Key, Secret Key, region (us-east-1), and output format

# Install kubectl (if not already installed)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Install eksctl (if not already installed)
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

## Step 2: Connect to your EKS cluster

```bash
# Update your kubeconfig to connect to your EKS cluster
aws eks update-kubeconfig --name YOUR_CLUSTER_NAME --region us-east-1

# Verify you can connect to your cluster
kubectl get nodes
```

## Step 3: Set up IAM OIDC Provider for your EKS cluster

```bash
# Check if an OIDC provider already exists for your cluster
aws eks describe-cluster --name YOUR_CLUSTER_NAME --query "cluster.identity.oidc.issuer" --output text

# Set up OIDC provider if not already configured
eksctl utils associate-iam-oidc-provider --cluster YOUR_CLUSTER_NAME --approve --region us-east-1
```

## Step 4: Install AWS Load Balancer Controller

```bash
# Create IAM policy for the AWS Load Balancer Controller
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json

aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam-policy.json

# Create service account with IAM role for the controller
eksctl create iamserviceaccount \
  --cluster=YOUR_CLUSTER_NAME \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# Install cert-manager (required for the controller)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.0/cert-manager.yaml

# Wait for cert-manager to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=cert-manager -n cert-manager --timeout=60s

# Add the controller helm repository
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# Install the controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=YOUR_CLUSTER_NAME \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

## Step 5: Install AWS Secrets Store CSI Driver and AWS Provider

```bash
# Install the Secrets Store CSI Driver
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm repo update

helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver \
    --namespace kube-system \
    --set syncSecret.enabled=true

# Install the AWS Provider for Secrets Store CSI Driver
helm repo add aws-secrets-manager https://aws.github.io/secrets-store-csi-driver-provider-aws
helm repo update

helm install aws-secrets-manager-provider aws-secrets-manager/secrets-store-csi-driver-provider-aws \
    --namespace kube-system

# Verify the installation
kubectl get pods -n kube-system -l app=secrets-store-csi-driver
kubectl get pods -n kube-system -l app=csi-secrets-store-provider-aws
```

## Step 6: Create IAM Role for Service Account for Secrets Manager access

```bash
# Create IAM policy for Secrets Manager access
cat > ssm-policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue",
                "secretsmanager:DescribeSecret"
            ],
            "Resource": "arn:aws:secretsmanager:us-east-1:$(aws sts get-caller-identity --query Account --output text):secret:mtap/dev/mysql*"
        }
    ]
}
EOF

aws iam create-policy \
    --policy-name mtap-ssm-reader-policy \
    --policy-document file://ssm-policy.json

# Create namespace if it doesn't exist
kubectl create namespace development

# Create service account with IAM role
eksctl create iamserviceaccount \
    --name ssm-reader \
    --namespace development \
    --cluster YOUR_CLUSTER_NAME \
    --attach-policy-arn arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):policy/mtap-ssm-reader-policy \
    --approve \
    --override-existing-serviceaccounts
```

## Step 7: Create your Secret in AWS Secrets Manager

```bash
aws secretsmanager create-secret \
    --name mtap/dev/mysql \
    --secret-string '{
        "MYSQL_DATABASE": "your_database",
        "MYSQL_USERNAME": "your_username",
        "MYSQL_PASSWORD": "your_password",
        "MYSQL_HOST": "your_host.rds.amazonaws.com",
        "MYSQL_PORT": "3306",
        "S3ACCESSKEY": "your_s3_access_key",
        "S3SECRETKEY": "your_s3_secret_key",
        "DRUPAL_ENV": "development"
    }'
```

## Step 8: Apply your Helm Chart

```bash
# Deploy your application using Helm
helm install mtap ./mtap-k8s-helm  -n development
```

## Step 9: Verify Everything is Working

```bash
# Check all deployments, pods, and services
kubectl get all -n development

# Check for any errors in pods
kubectl describe pods -n development

# Check if the secrets are properly mounted
kubectl exec -n development $(kubectl get pods -n development -l app=mtap -o jsonpath="{.items[0].metadata.name}") -- ls -la /mnt/secrets-store

# Check HPA status
kubectl get hpa -n development

# Verify target group binding
kubectl get targetgroupbindings -n development
```

## Troubleshooting Tips

1. **If pods are not starting properly**, check the pod events:
   ```bash
   kubectl describe pod <pod-name> -n development
   ```

2. **If secrets are not mounting correctly**, check the CSI driver logs:
   ```bash
   kubectl logs -n kube-system -l app=secrets-store-csi-driver
   ```

3. **For AWS Load Balancer Controller issues**, check its logs:
   ```bash
   kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
   ```

4. **To verify IRSA is working**, check if the service account has the proper annotations:
   ```bash
   kubectl describe serviceaccount ssm-reader -n development
   ```

This comprehensive guide should help you set up everything needed for your EKS cluster to work with AWS Secrets Manager via the CSI driver and the AWS Load Balancer Controller. Make sure to replace `YOUR_CLUSTER_NAME` with your actual EKS cluster name throughout these instructions.
