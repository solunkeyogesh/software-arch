# EKS + NodePort + RDS + S3 + Lambda (No Ingress) — End‑to‑End Setup

This guide covers how to set up:
- EKS services exposed via **NodePort** (no Ingress/ALB)
- **Amazon RDS** (PostgreSQL by default; notes for MySQL)
- **Amazon S3** bucket
- **IRSA** for pod-level S3 access
- **AWS Lambda** (in VPC) triggered by S3, writing to RDS

Replace ALL_CAPS placeholders and run section by section.

---

## 0) Assumptions / Variables

```bash
export AWS_REGION=us-east-1
export CLUSTER_NAME=eksdemo1
export NAMESPACE=app
export S3_BUCKET=acme-dev-app-bucket-<yoursuffix>  # must be globally unique
export DB_NAME=appdb
export DB_USER=appuser
export DB_PASS=apppass-ChangeMe!
```

- Region: `us-east-1` (change if needed)
- RDS engine: PostgreSQL (swap to MySQL: `--engine mysql --port 3306` later)

---

## 1) Discover Cluster Networking (VPC, Subnets, Node SG)

```bash
aws eks describe-cluster --name $CLUSTER_NAME --region $AWS_REGION   --query "cluster.resourcesVpcConfig.subnetIds" --output text

NODE_SG=$(aws ec2 describe-network-interfaces   --filters "Name=description,Values=Amazon EKS *" "Name=status,Values=in-use"   --query "NetworkInterfaces[0].Groups[0].GroupId" --output text)

VPC_ID=$(aws ec2 describe-network-interfaces   --filters "Name=group-id,Values=${NODE_SG}"   --query "NetworkInterfaces[0].VpcId" --output text)

echo "NODE_SG=$NODE_SG"
echo "VPC_ID=$VPC_ID"
```

> Use two **private subnets in different AZs** for RDS and Lambda.

---

## 2) Create RDS Subnet Group, SG, and DB Instance

```bash
export RDS_SUBNETS="<subnet-aaaa> <subnet-bbbb>"

aws rds create-db-subnet-group   --db-subnet-group-name ${CLUSTER_NAME}-rds-subnets   --db-subnet-group-description "RDS subnets for ${CLUSTER_NAME}"   --subnet-ids $RDS_SUBNETS --region $AWS_REGION

RDS_SG=$(aws ec2 create-security-group   --group-name ${CLUSTER_NAME}-rds-sg   --description "RDS SG for ${CLUSTER_NAME}"   --vpc-id $VPC_ID --query GroupId --output text)

aws ec2 authorize-security-group-ingress --group-id $RDS_SG   --protocol tcp --port 5432 --source-group $NODE_SG

aws rds create-db-instance   --db-instance-identifier ${CLUSTER_NAME}-pg   --engine postgres --engine-version 16.3   --master-username $DB_USER --master-user-password $DB_PASS   --allocated-storage 20 --db-instance-class db.t4g.micro   --db-subnet-group-name ${CLUSTER_NAME}-rds-subnets   --vpc-security-group-ids $RDS_SG   --backup-retention-period 1   --multi-az   --publicly-accessible false   --storage-encrypted   --region $AWS_REGION

aws rds wait db-instance-available --db-instance-identifier ${CLUSTER_NAME}-pg

DB_ENDPOINT=$(aws rds describe-db-instances   --db-instance-identifier ${CLUSTER_NAME}-pg   --query "DBInstances[0].Endpoint.Address" --output text)
echo "DB_ENDPOINT=$DB_ENDPOINT"
```

---

## 3) Create S3 Bucket (+ optional VPC Endpoint)

```bash
aws s3api create-bucket   --bucket $S3_BUCKET   --create-bucket-configuration LocationConstraint=$AWS_REGION   --region $AWS_REGION

aws ec2 create-vpc-endpoint   --vpc-id $VPC_ID   --service-name com.amazonaws.$AWS_REGION.s3   --route-table-ids $(aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID"      --query "RouteTables[].RouteTableId" --output text)   --vpc-endpoint-type Gateway
```

---

## 4) IRSA for S3 Access from Pods

```bash
cat > s3-app-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "BucketLimitedAccess",
    "Effect": "Allow",
    "Action": ["s3:PutObject","s3:GetObject","s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::${S3_BUCKET}",
      "arn:aws:s3:::${S3_BUCKET}/*"
    ]
  }]
}
EOF

aws iam create-policy --policy-name ${CLUSTER_NAME}-s3-app-policy   --policy-document file://s3-app-policy.json

# OIDC provider assumed already exists for cluster
```

(Create role, trust policy, annotate ServiceAccount as in the `.txt` file)

---

## 5) Deployments + NodePort Services

**API Deployment (NodePort 30080):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: app
spec:
  replicas: 2
  selector: { matchLabels: { app: api } }
  template:
    metadata: { labels: { app: api } }
    spec:
      serviceAccountName: app-sa
      containers:
      - name: api
        image: <your-ecr>/api:latest
        ports: [{ containerPort: 8080 }]
---
apiVersion: v1
kind: Service
metadata:
  name: api-svc
  namespace: app
spec:
  type: NodePort
  selector: { app: api }
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30080
```

**Web Deployment (NodePort 30081):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: app
spec:
  replicas: 2
  selector: { matchLabels: { app: web } }
  template:
    metadata: { labels: { app: web } }
    spec:
      serviceAccountName: app-sa
      containers:
      - name: web
        image: <your-ecr>/web:latest
        ports: [{ containerPort: 8080 }]
        env:
        - name: API_URL
          value: "http://api-svc.app.svc.cluster.local:8080"
---
apiVersion: v1
kind: Service
metadata:
  name: web-svc
  namespace: app
spec:
  type: NodePort
  selector: { app: web }
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30081
```

---

## 6) Lambda Setup

- Create Security Group for Lambda → allow RDS access  
- Create IAM Role for Lambda with `AWSLambdaBasicExecutionRole` and `AmazonS3ReadOnlyAccess`  
- Deploy Lambda inside same VPC/subnets as RDS  
- Connect S3 bucket events to Lambda (ObjectCreated trigger)  

(Steps identical to `.txt` file, trimmed here for brevity)

---

## 7) Test

```bash
# NodePort
kubectl get nodes -o wide
curl http://<NODE_PUBLIC_IP>:30080/health

# S3 → Lambda → RDS
echo "hello" > /tmp/test.txt
aws s3 cp /tmp/test.txt s3://$S3_BUCKET/test.txt
```

---

## Cleanup

Delete K8s objects, Lambda, S3, RDS, IAM roles, and SGs as shown in the `.txt` file.
