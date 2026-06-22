Got it! Let's combine everything beautifully. This single, complete command will write the entire `README.md` file to your folder—including the full explanation, your `scale-test-deployment.yaml` manifest data, the exact patch commands, and the teardown steps.

Run this right inside your terminal:

```bash
cat << 'EOF' > README.md
# Exercise 6 – EKS Cluster Autoscaler Configuration & Max Limits Failure

This directory contains the production-grade simulation artifacts, root cause analysis, and explicit terminal commands used to resolve a critical **Elastic Kubernetes Service (EKS) Node Scale Failure**.

---

## 🚨 Incident Scenario & Diagnosis

### The Symptom
During a high-traffic event, the application's **Horizontal Pod Autoscaler (HPA)** scaled the deployment requirements up to a desired count of **15 replicas**. However, the cluster choked:
* Only **2 pods** successfully entered a `Running` state.
* **13 pods** remained stranded in a `Pending` status.
* Running `kubectl describe pod` revealed the bottleneck: `0/2 nodes are available: 2 Insufficient cpu`.

### The Root Cause Analysis
1. **Missing Control Plane Component:** The EKS cluster was missing the core **Cluster Autoscaler (CA)** deployment engine inside the `kube-system` namespace. The cluster could detect the scheduling issue but had no way to communicate resource demands to AWS.
2. **Infrastructure Glass Ceiling:** After deploying the Cluster Autoscaler, logs showed the engine stalling with `Skipping node group <name> - max size reached`. The underlying AWS Auto Scaling Group (ASG) was hard-capped at an restrictive maximum limit, blocking the cluster from scaling up to accommodate the 15-core load.

---

## 📦 Simulation Artifacts

### Application Test Manifest (`manifests/scale-test-deployment.yaml`)
This deployment demands 1 full CPU core (`1000m`) per pod, forcing resource congestion to trigger the autoscaler failure:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: heavy-scale-app
  namespace: default
spec:
  replicas: 15
  selector:
    matchLabels:
      app: heavy-scale
  template:
    metadata:
      labels:
        app: heavy-scale
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        resources:
          requests:
            cpu: "1000m"
            memory: "128Mi"

```

---

## 🛠️ Step-by-Step Resolution Sequence

### Step 1: Create the Identity & Permissions Bridge (IRSA)

We used `eksctl` to establish an IAM Role for Service Accounts (IRSA). This maps a secure AWS managed policy to the cluster control plane so it can programmatically manipulate EC2 Auto Scaling capacities:

```bash
eksctl create iamserviceaccount \
    --cluster=payment-gitops-cluster \
    --namespace=kube-system \
    --name=cluster-autoscaler \
    --attach-policy-arn=arn:aws:iam::aws:policy/AutoScalingFullAccess \
    --override-existing-serviceaccounts \
    --approve \
    --region=us-east-1

```

### Step 2: Deploy the Cluster Autoscaler Engine

With an authorized identity initialized, the official cloud provider manifest was applied directly to the cluster:

```bash
kubectl apply -f [https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml](https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml)

```

### Step 3: Patch Auto-Discovery Configuration

We patched the deployment to pass your specific EKS cluster name (`payment-gitops-cluster`) into the container arguments so it could actively scan for matching AWS tags:

```bash
kubectl patch deployment cluster-autoscaler \
  -n kube-system \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"cluster-autoscaler","command":["./cluster-autoscaler","--v=4","--stderrthreshold=info","--cloud-provider=aws","--skip-nodes-with-local-storage=false","--expander=least-waste","--node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/payment-gitops-cluster"]}]}}}}'

```

### Step 4: Elevate the AWS Group Capacity

To clear the `max size reached` constraint and provide enough computing space for the remaining pods, we dynamically shifted the node group boundary to a maximum cap of **20 nodes**:

```bash
eksctl scale nodegroup \
    --cluster=payment-gitops-cluster \
    --name=standard-workers \
    --nodes-max=20 \
    --region=us-east-1

```

---

## 🏁 Verification & Infrastructure Verification

As soon as the max limits were raised, the Cluster Autoscaler successfully requested capacity from AWS. The worker instance fleet expanded to **10 healthy nodes** to absorb the compute congestion:

```bash
$ kubectl get nodes
NAME                             STATUS   ROLES    AGE     VERSION
ip-192-168-11-202.ec2.internal   Ready    <none>   2m52s   v1.34.9-eks-93b80c6
ip-192-168-12-124.ec2.internal   Ready    <none>   5h38m   v1.34.9-eks-93b80c6
ip-192-168-17-136.ec2.internal   Ready    <none>   2m53s   v1.34.9-eks-93b80c6
ip-192-168-2-5.ec2.internal      Ready    <none>   2m52s   v1.34.9-eks-93b80c6
ip-192-168-30-151.ec2.internal   Ready    <none>   2m52s   v1.34.9-eks-93b80c6
ip-192-168-39-22.ec2.internal    Ready    <none>   2m53s   v1.34.9-eks-93b80c6
ip-192-168-40-154.ec2.internal   Ready    <none>   5h38m   v1.34.9-eks-93b80c6
ip-192-168-44-32.ec2.internal    Ready    <none>   2m53s   v1.34.9-eks-93b80c6
ip-192-168-54-218.ec2.internal   Ready    <none>   6m42s   v1.34.9-eks-93b80c6
ip-192-168-58-86.ec2.internal    Ready    <none>   2m51s   v1.34.9-eks-93b80c6

```

---

## 📉 Post-Simulation Clean Up & Cool-Down

To prevent unexpected infrastructure billing, the simulation application was removed, and the node group parameters were restored to their standard baseline limits:

```bash
# 1. Clear out the heavy load resource deployment
kubectl delete -f manifests/scale-test-deployment.yaml

# 2. Reset the AWS Nodegroup ceiling safely back to normal parameters
eksctl scale nodegroup \
    --cluster=payment-gitops-cluster \
    --name=standard-workers \
    --nodes-max=4 \
    --region=us-east-1

```

EOF

```

```
