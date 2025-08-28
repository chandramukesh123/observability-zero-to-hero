### `eks-cluster-setup.md`

````md
# EKS Cluster Setup with Monitoring Stack

## 1. Create EKS Cluster with 2 Nodes of Type t3.medium

```bash
eksctl create cluster \
  --name mukesh-observability \
  --region us-east-1 \
  --nodegroup-name observability-nodes \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed
````


---

## 2. Check Cluster Context

```bash
kubectl config get-contexts
```

---

## 3. Use the Created Context

```bash
kubectl config use-context mukesh-observability.us-east-1.eksctl.io
```

---

## 4. Install Helm

```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null

sudo apt-get install apt-transport-https --yes

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list

sudo apt-get update

sudo apt-get install helm
```

---

## 5. Create IAM Policy for AWS Load Balancer Controller

```bash
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

aws sts get-caller-identity
```

---

## 6. Create IAM Service Account and Attach IAM Policy

```bash
eksctl create iamserviceaccount \
  --cluster=mukesh-observability \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::<aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

---

## 7. Install AWS Load Balancer Controller via Helm

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=mukesh-observability \
  --set serviceAccount.create=false \
  --set region=us-east-1 \
  --set vpcId=vpc-0454546e2b77a935b \
  --set serviceAccount.name=aws-load-balancer-controller
```

---

## 8. Verify AWS Load Balancer Controller Installation

```bash
kubectl get pods -n kube-system | grep aws-load-balancer-controller
kubectl get svc -n monitoring
```

---

## 9. Install Prometheus, Grafana, and Alertmanager

```bash
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f values.yaml

kubectl get pods -n monitoring
kubectl get ingress -n monitoring
kubectl --namespace monitoring get pods -l "release=monitoring"
kubectl get svc -n monitoring
```

---

## 10. Create a CrashLoopBackOff Pod for Testing

```bash
kubectl run busybox-crash --image=busybox -- /bin/sh -c "exit 1"
kubectl get pods
```