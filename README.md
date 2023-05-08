# Kubernetes_ALB_SSL_Enabled
Deploying the Kubernetes cluster in Private and expose the POD via SSL enabled ALB


# Architect view
1) Create Cluster with nodegroup in private subnet
2) Create IAM role with policy to enable the necessary access for cluster to taken action for ELB
3) This role to be mapped with Cluster-ServiceAccount
4) Install the load balancer controller in the cluster
5) Register the public domain for SSL
6) Basis the domain name, generate the SSL certificate that to be used in ALB Annotation for mapping with service

# Once SSL Certificate is generated then proceed with pod deployments
1) Create the default ingress class
2) Deploy the pod and expose it via NodePort service
3) Create ingress service and map it with the exposed NodePort service

# SSL mapping and Context based routing
1) In ingress.annotation :-
    a) Add the ARN of SSL certificate // to map the certificate with ALB for said registered domain
    b) Redirect the traffic from 80 to 443
2) Define the path for respective application in the "ingress" service
3) Basis this path loadbalancer will be updated with lister rules
4) Use the default backend concept as default path for unmatch routing
4) Basis the listener rules, traffic will be routed to respective service accordingly

# What will be deployed
1) Cluster + NodeGroup
2) Role for Cluster
3) LoadBalancer controller in cluster
4) Public Domain registeration via Route53
5) SSL certificate generation for said domain in ACM
6) Ingress Class
7) App POD + NodePort Service
8) Ingress Service
  a) Application Load Balancer
  b) NodePort service mapping with ALB
  c) Listener rules in ALB for context based routing
  d) Deployment sequence of "- path" matter as first match will be preferred

#####################################


# Execution Sequence to expose the POD in private subnet via ALB in Public Subnet

# Step-01: Create Cluster
sh create_cluster.sh
Create key-pair "veru-eks"
Enable inbound rule in security-group for port 80 on EC2-NodeGroup remote access facing security group


# Step-02: Create IAM Policy
Download  latest IAM Policy

curl -o iam_policy_latest.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

Download specific version //substitute to above command to get the speicfic version
curl -o iam_policy_v2.3.1.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.3.1/docs/install/iam_policy.json


Create IAM Policy using policy downloaded 
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy_latest.json
ensure ec2:DescribeAvailabilityZones is allowed in above policy
output "Arn": "arn:aws:iam::809411733541:policy/AWSLoadBalancerControllerIAMPolicy"


# Step-03: Create an IAM role for the AWS LoadBalancer Controller and attach the role to the Kubernetes service account
  eksctl create iamserviceaccount --name aws-load-balancer-controller --namespace kube-system --cluster chorus \
    --attach-policy-arn arn:aws:iam::809411733541:policy/AWSLoadBalancerControllerIAMPolicy --approve



Verify service account creation
eksctl  get iamserviceaccount --cluster acoustic
Verify if any existing service account
kubectl get sa -n kube-system
kubectl get sa aws-load-balancer-controller -n kube-system
kubectl describe sa aws-load-balancer-controller -n kube-system


# Step-04: Install the AWS Load Balancer Controller using Helm V3

brew install Helm
Add the eks-charts repository:
helm repo add eks https://aws.github.io/eks-charts

Update your local repo to make sure that you have the most recent charts.
helm repo update

Install the AWS Load Balancer Controller:
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=chorus \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=vpc-009dc7739895aea55

Use 'upgrade' in place of 'install' for next-time upgrade
helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=<k8s-cluster-name> --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller

---

Verify that the controller is installed.
kubectl -n kube-system get deployment 
kubectl -n kube-system get deployment aws-load-balancer-controller
kubectl -n kube-system describe deployment aws-load-balancer-controller

Verify AWS Load Balancer Controller Webhook service created
kubectl -n kube-system get svc 
kubectl -n kube-system get svc aws-load-balancer-webhook-service
kubectl -n kube-system describe svc aws-load-balancer-webhook-service

Verify Labels in Service and Selector Labels in Deployment
kubectl -n kube-system get svc aws-load-balancer-webhook-service -o yaml
kubectl -n kube-system get deployment aws-load-balancer-controller -o yaml

List Service Account and its secret
kubectl -n kube-system get sa aws-load-balancer-controller
kubectl -n kube-system get sa aws-load-balancer-controller -o yaml
kubectl -n kube-system get secret <GET_FROM_PREVIOUS_COMMAND - secrets.name> -o yaml
kubectl -n kube-system get secret aws-load-balancer-controller-token-5w8th 
kubectl -n kube-system get secret aws-load-balancer-controller-token-5w8th -o yaml
## Decoce ca.crt using below two websites
https://www.base64decode.org/
https://www.sslchecker.com/certdecoder
# List aws-load-balancer-tls secret 
kubectl -n kube-system get secret aws-load-balancer-tls -o yaml

# Uninstall AWS Load Balancer Controller
helm uninstall aws-load-balancer-controller -n kube-system




