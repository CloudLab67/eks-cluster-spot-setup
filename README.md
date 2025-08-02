# eks-cluster-spot-setup
🚀 AWS EKS Cluster Setup with Bastion Host & Spot Nodes (Low-Cost, Production-Style)
This guide demonstrates how to create a production-style AWS EKS cluster using:

Bastion Host for secure access

IAM User for CLI management

EKS Cluster with Spot NodeGroup

OIDC Integration

Cluster Autoscaler

Proper Cleanup to avoid costs

Optimized for low cost (~₹20–₹25 per 2-hour session) while maintaining production concepts.

📌 Architecture Diagram
plaintext
Copy
Edit
┌───────────────────────┐
│     IAM User (eks-admin)│
│   (AdministratorAccess)│
└─────────┬──────────────┘
          │AWS CLI
          ▼
┌───────────────────────┐
│   Bastion Host (EC2)  │
│  t3.micro Spot        │
│  SSH Access           │
└─────────┬──────────────┘
          │kubectl/eksctl
          ▼
┌───────────────────────────────────────────────┐
│               EKS Cluster                     │
│   ┌───────────────────────────────────────┐  │
│   │ OIDC Provider                          │  │
│   │ Cluster Autoscaler                     │  │
│   │ NodeGroup (t3.medium Spot)             │  │
│   │ Min=2, Max=4                           │  │
│   └───────────────────────────────────────┘  │
└───────────────────────────────────────────────┘
1️⃣ Create Bastion Host (Jump Server)
Region: ap-south-1
Instance type: t3.micro (Spot)
AMI: Ubuntu 22.04 LTS

Steps:

Go to EC2 Console → Launch Instance

Instance type: t3.micro (Spot enabled)

Key pair: bastion-key-ap-south-1

Security Group: Allow SSH (Port 22) from your IP

Launch the instance.

2️⃣ Create IAM User for CLI
Steps:

IAM Console → Users → Add user

User name: eks-admin

Access type: Programmatic access

Attach policy: AdministratorAccess

Save Access Key ID & Secret Key

3️⃣ Configure AWS CLI on Bastion
SSH to bastion:

bash
Copy
Edit
ssh -i bastion-key-ap-south-1.pem ubuntu@<Bastion-IP>
Install AWS CLI:

bash
Copy
Edit
sudo apt update && sudo apt install -y unzip curl
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
Configure:

bash
Copy
Edit
aws configure
4️⃣ Install kubectl
bash
Copy
Edit
curl -o kubectl https://amazon-eks.s3.ap-south-1.amazonaws.com/1.30.0/2024-04-19/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
5️⃣ Install eksctl
bash
Copy
Edit
curl --silent --location "https://github.com/eksctl-io/eksctl/releases/download/v0.212.0/eksctl_Linux_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
6️⃣ Create EKS Cluster (Without Nodegroup)
bash
Copy
Edit
eksctl create cluster \
  --name=eksdemo \
  --region=ap-south-1 \
  --zones=ap-south-1a,ap-south-1b \
  --without-nodegroup
7️⃣ Associate OIDC Provider
bash
Copy
Edit
eksctl utils associate-iam-oidc-provider \
  --region ap-south-1 \
  --cluster eksdemo \
  --approve
8️⃣ Create Spot NodeGroup
bash
Copy
Edit
eksctl create nodegroup \
  --cluster=eksdemo \
  --region=ap-south-1 \
  --name=eksdemo-ng-spot \
  --node-type=t3.medium \
  --nodes=2 \
  --nodes-min=2 \
  --nodes-max=4 \
  --node-volume-size=8 \
  --spot \
  --ssh-access \
  --ssh-public-key=EKS-Node-AP-S \
  --managed
9️⃣ Install Cluster Autoscaler
bash
Copy
Edit
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/cluster-autoscaler-1.30.0/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
Edit deployment:

bash
Copy
Edit
kubectl -n kube-system edit deployment.apps/cluster-autoscaler
Update:

diff
Copy
Edit
--cluster-name=eksdemo
--balance-similar-node-groups
--skip-nodes-with-system-pods=false
Tag nodegroup:

bash
Copy
Edit
aws eks tag-nodegroup \
  --cluster-name eksdemo \
  --nodegroup-name eksdemo-ng-spot \
  --tags "k8s.io/cluster-autoscaler/enabled=true" "k8s.io/cluster-autoscaler/eksdemo"
🔟 Test Auto Scaling
Terminate a node:

bash
Copy
Edit
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
kubectl delete node <node-name>
✅ New node will be created automatically.

1️⃣1️⃣ Cleanup (Avoid Charges)
bash
Copy
Edit
eksctl delete nodegroup --cluster=eksdemo --region=ap-south-1 --name=eksdemo-ng-spot
eksctl delete cluster --name=eksdemo --region=ap-south-1
Delete unattached EBS:

bash
Copy
Edit
VOLUMES=$(aws ec2 describe-volumes --region ap-south-1 --query "Volumes[?State=='available'].VolumeId" --output text)
for vol in $VOLUMES; do
  aws ec2 delete-volume --volume-id $vol --region ap-south-1
done
