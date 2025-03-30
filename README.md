# Deploying eks cluster using ec2 managed node group

# Steps

- TAKE A MACHINE/OR EXECUTE LOCALLY BELOW STEPS

# Install terraform
```sh
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform
```

# install awscli

```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install --bin-dir /usr/local/bin --install-dir /usr/local/aws-cli --update
```

# aws configure
```
aws configure


root@test-vm1:~/terraform# aws configure
AWS Access Key ID [None]: 
AWS Secret Access Key [None]: 
Default region name [None]: us-west-1
Default output format [None]: json
```

# Kubectl
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
curl -LO https://dl.k8s.io/release/v1.28.0/bin/linux/amd64/kubectl
sudo mv kubectl /usr/local/bin/kubectl
sudo chmod +x /usr/local/bin/kubectl
```

# terraform steps
```
aws configure


export AWS_ACCESS_KEY_ID=""
export AWS_SECRET_ACCESS_KEY=""

terraform init

terraform plan

terraform apply -auto-approve

aws eks --region us-east-1 update-kubeconfig --name demo-eks-QGGshN7P

kubectl get nodes
- you will get timeout error
- after deploying with terraform, our eks cluster will be in private subnet
- in the aws ui, you will see cluster but cant see pods, nodes, etc...because by default you will not have access to new cluster created
```


Since your **EKS cluster is in a private subnet**, the API server is not exposed to the public internet, meaning you **cannot access it directly from your external VM** (outside AWS).

### **How to Fix?**
Since your cluster is in a **private subnet**, you have two main options:

#### **Option 1: Use a Bastion Host (Recommended)**
1. **Launch an EC2 instance inside the same VPC** as your EKS cluster.
2. **SSH into the EC2 instance** from your external VM.
3. **Run `kubectl` commands from the EC2 instance** to interact with EKS.

This method ensures security while allowing you to manage the cluster.

#### **Option 2: Enable Public Access for EKS API (Less Secure)**
If you **must** access the EKS cluster directly from your external VM:
1. Run:
   ```sh
   aws eks update-cluster-config --region us-east-1 --name demo-eks-QGGshN7P --resources-vpc-config endpointPublicAccess=true
   ```
2. Allow your VM’s IP to access the API server by adding an **inbound rule** to your security group:
   ```sh
   aws ec2 authorize-security-group-ingress --group-id sg-0e2b44dc6908dab3d --protocol tcp --port 443 --cidr YOUR_VM_IP/32
   ```
   (Replace `YOUR_VM_IP` with your actual public IP)

However, **this is not recommended** for production due to security risks.



# BASTION/JUMP SERVER
```
- we need to launch jump server on same VPC of eks cluster got deployed
- demo-eks-vpc
- choose public subnet with public ip
- demo-eks-vpc-public-us-east-1-a
- Auto-assign public IP: eable
- name sg: demo-eks-jump-sg

- in prod environments, it is recommended to restrict access to jump servers to a single ip,
so that our entire infra wont be exposed
```

# aws configure on jump server
```
aws configure


root@test-vm1:~/terraform# aws configure
AWS Access Key ID [None]: 
AWS Secret Access Key [None]: 
Default region name [None]: us-east-1
Default output format [None]: json
```

# Kubectl on jump server
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
curl -LO https://dl.k8s.io/release/v1.28.0/bin/linux/amd64/kubectl
sudo mv kubectl /usr/local/bin/kubectl
sudo chmod +x /usr/local/bin/kubectl
```

# access cluster form jump server
```
aws eks --region us-east-1 update-kubeconfig --name demo-eks-QGGshN7P


ec2-user@ip-10-0-4-172 ~]$ aws eks --region us-east-1 update-kubeconfig --name demo-eks-QGGshN7P
Added new context arn:aws:eks:us-east-1:590183962815:cluster/demo-eks-QGGshN7P to /home/ec2-user/.kube/config
[ec2-user@ip-10-0-4-172 ~]$ kubectl get nodes
E0328 07:42:49.370634    2893 memcache.go:265] couldn't get current server API group list: Get "https://CD9CE8AC4BB2965EA957C164C2DE9971.gr7.us-east-1.eks.amazonaws.com/api?timeout=32s": dial tcp 10.0.1.174:443: i/o timeout

-> https://youtu.be/M3j0mln3jBo?si=bGHv61Hwvek6zzRE

-> still you wont get results, beacuse our eks cluster which is in private subnet ahs to allow jump server sg
- we need to open 443 for eks cluster sg for jump server sg
- goto eks cluster sg and add a inbound rule
- open cluster sg and add 443 rule to target sg-03f55737dac872b45(jump server sg)


[ec2-user@ip-10-0-4-172 ~]$ kubectl get nodes
E0328 07:49:40.784867    3149 memcache.go:265] couldn't get current server API group list: the server has asked for the client to provide credentials

-> https://youtu.be/vG9FNSlBU80?si=dATSSS8fqXXPgRMV


- now you are able to reach cluster but cannot authenticate
- by default eks cluster will restrict all access to it
- do below steps to get access to cluster, that steps can be done by admin who ahs access to add users to cluster

* 		Amazon Elastic Kubernetes Service 
* 		Clusters 
* 		demo-eks-QGGshN7P 
* 		Create access entry

-> Configure IAM access entry
-> select iam user: arn:aws:iam::590183962815:user/vijay
-> type: standard
-> rest keep optional
-> click next
-> Add AmazonEKSAdminPloicy, AmazonEKSClusterAdminPolicy
-> cluster
-> create
-> Access entry was successfully created for user into cluster
-> now got back to cluster and see pods, and nodes on ui


arn:aws:iam::590183962815:user/vijay	Standard	arn:aws:iam::590183962815:user/vijay	-	AmazonEKSAdminPolicy, AmazonEKSClusterAdminPolicy


- come back to jump server

[ec2-user@ip-10-0-4-172 ~]$ kubectl get nodes
NAME                         STATUS   ROLES    AGE    VERSION
ip-10-0-1-248.ec2.internal   Ready    <none>   138m   v1.27.16-eks-aeac579
ip-10-0-2-158.ec2.internal   Ready    <none>   138m   v1.27.16-eks-aeac579

[ec2-user@ip-10-0-4-172 ~]$ kubectl get ns
NAME              STATUS   AGE
default           Active   147m
kube-node-lease   Active   147m
kube-public       Active   147m
kube-system       Active   147m
```

Now that you have access to your EKS cluster from the jump server, follow these steps to create a new namespace, deploy a simple **Hynix** pod, expose it via a ClusterIP service, and test access from the jump server.

---

## **Step 1: Create the `rapid` Namespace**
```sh
kubectl create namespace rapid
```
Verify the namespace:
```sh
kubectl get ns
```

---

## **Step 2: Deploy a Simple Pod (Hynix)**
Create a simple `hynix-pod.yaml` file:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hynix
  namespace: rapid
  labels:
    app: hynix
spec:
  containers:
    - name: hynix
      image: nginx:latest
      ports:
        - containerPort: 80
```
Apply the pod:
```sh
kubectl apply -f hynix-pod.yaml
```

Check if the pod is running:
```sh
kubectl get pods -n rapid
```

---

## **Step 3: Create a ClusterIP Service**
Now, expose the pod inside the cluster using a **ClusterIP** service.

Create a `hynix-service.yaml` file:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: hynix-service
  namespace: rapid
spec:
  selector:
    app: hynix
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```
Apply the service:
```sh
kubectl apply -f hynix-service.yaml
```

Check the service:
```sh
kubectl get svc -n rapid
```
Example output:
```
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
hynix-service  ClusterIP   10.100.200.45   <none>        80/TCP    10s
```

---

## **Step 4: Access the Service from the Jump Server**
Since this is a **ClusterIP** service, it is only accessible inside the cluster. From the **jump server**, you can **test access from within a pod**.

1. **Run a temporary busybox pod inside the cluster**
   ```sh
   kubectl run test-pod --rm -it --image=busybox -- /bin/sh
   ```
2. **Inside the pod, test access using `wget` or `curl`**
   ```sh
   wget -qO- http://hynix-service.rapid.svc.cluster.local
   ```
   or
   ```sh
   curl http://hynix-service.rapid.svc.cluster.local
   ```


# testing
```
[ec2-user@ip-10-0-4-172 ~]$ kubectl run test-pod --rm -it --image=busybox -n rapid  /bin/sh 
If you don't see a command prompt, try pressing enter.
/ # wget -qO- http://hynix-service.rapid.svc.cluster.local
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```
If this works, you have successfully deployed and accessed the service within the EKS cluster!

---

## **Alternative: Expose the Service Externally**
If you need to access it **directly from the jump server** (without using a pod), you must change the service type to **NodePort** or **LoadBalancer**.

Would you like to expose it externally using NodePort? 🚀




# Testing with Gatekeeper inside cluster
# k8s-opa-gatekeeper

This project is demoing OPA Gatekeeper with Kubernetes. Gatekeeper is an opensource project supported by CNCF. It aims for creating policies to manage Kubernetes like:

- Enforcing labels on namespaces.
- Allowing only images coming from certain Docker Registries.
- Require all Pods specify resource requests and limits.
- Prevent conflicting Ingress objects from being created.
- Enforcing running containers as non-root.

```
- deploy gatekeeper
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml


# Scenario: Enforce Having A Specific Label In Any New Namespace
# Deploy the Contraint Template

kubectl apply -f k8srequiredlabels_template.yaml


# Deploy the Constraints

kubectl apply -f all_ns_must_have_gatekeeper.yaml


# Deploy a namespace denied by Policy

kubectl apply -f bad-namespace.yaml


# Deploy a namespace allowed by Policy

kubectl apply -f good-namespace.yaml


# implementation

kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml

[ec2-user@ip-10-0-4-172 k8s-opa-gatekeeper-demo]$ kubectl get ns
NAME                STATUS   AGE
default             Active   4h6m
gatekeeper-system   Active   16s
kube-node-lease     Active   4h6m
kube-public         Active   4h6m
kube-system         Active   4h6m
rapid               Active   92m


[ec2-user@ip-10-0-4-172 k8s-opa-gatekeeper-demo]$ kubectl get pods -n gatekeeper-system
NAME                                             READY   STATUS    RESTARTS      AGE
gatekeeper-audit-648c95c965-cflxz                1/1     Running   1 (28s ago)   32s
gatekeeper-controller-manager-7578784dd9-4tbqv   1/1     Running   0             32s
gatekeeper-controller-manager-7578784dd9-4vw4b   1/1     Running   0             32s
gatekeeper-controller-manager-7578784dd9-lhf7m   1/1     Running   0             32s
[ec2-user@ip-10-0-4-172 k8s-opa-gatekeeper-demo]$ 

[ec2-user@ip-10-0-4-172 k8s-opa-gatekeeper-demo]$ kubectl apply -f all_ns_must_have_gatekeeper.yaml
k8srequiredlabels.constraints.gatekeeper.sh/ns-must-have-gk created

[ec2-user@ip-10-0-4-172 k8s-opa-gatekeeper-demo]$ kubectl apply -f bad-namespace.yaml
Error from server (Forbidden): error when creating "bad-namespace.yaml": admission webhook "validation.gatekeeper.sh" denied the request:
 [ns-must-have-gk] you must provide labels: {"gatekeeper"}
 
[ec2-user@ip-10-0-4-172 k8s-opa-gatekeeper-demo]$ kubectl apply -f good-namespace.yaml
namespace/mynamespace created


- non-root-pod:
----------------
kubectl apply -f gatekeeper/non-root-pod/k8sno-root-pods_template.yaml
kubectl apply -f gatekeeper/non-root-pod/no-root-pods.yaml
kubectl apply -f gatekeeper/non-root-pod/bad-pod.yaml
kubectl apply -f gatekeeper/non-root-pod/good-pod.yaml

[ec2-user@ip-10-0-4-172 ~]$ kubectl get constraints
NAME                                                           ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
k8snorootpods.constraints.gatekeeper.sh/enforce-no-root-pods   deny                 6

NAME                                                          ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
k8srequiredlabels.constraints.gatekeeper.sh/ns-must-have-gk   warn                 2

- currently we have two policies applied on our cluster


kubectl create ns rapid

[ec2-user@ip-10-0-4-172 ~]$ kubectl get ns
NAME                STATUS   AGE
default             Active   29h
gatekeeper-system   Active   25h
kube-node-lease     Active   29h
kube-public         Active   29h
kube-system         Active   29h
mynamespace         Active   25h
rapid               Active   27h



test:
-----
cd non-root-nginx
docker build -t rapid-nginx:latest .
docker tag rapid-nginx:latest testproject1234.jfrog.io/test-docker-docker-local/rapid-nginx:v1
docker push testproject1234.jfrog.io/test-docker-docker-local/rapid-nginx:v1

kubectl create secret docker-registry artifactory-secret --docker-server=testproject1234.jfrog.io --docker-username="devopsvijay10@gmail.com" --docker-password=”” --docker-email=”OPTIONAL_EMAIL”

kubectl create secret docker-registry artifactory-secret --docker-server=testproject1234.jfrog.io --docker-username="devopsvijay10@gmail.com" --docker-password=”” --docker-email=”OPTIONAL_EMAIL” -n rapid


kubectl apply -f rapid-ngnix.yaml -n rapid
kubectl get pods -n rapid

kubectl delete -f rapid-ngnix.yaml 

kubectl get events -n rapid

troubleshoots:
--------------
kubectl get deployment nginx-rapid -n rapid
kubectl describe deployment nginx-rapid -n rapid
kubectl get rs -n rapid
kubectl describe rs -n rapid


Your current Deployment YAML already has runAsNonRoot: true and runAsUser: 1000, but it's likely that the base NGINX image is still running as root by default.
Admission webhook policies still detecting root privileges incorrectly
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000



	🔥 Issue Identified: Permission Denied on Port 80
	Your Nginx container cannot bind to port 80 due to insufficient permissions. This happens because:

	Ports below 1024 require root privileges.
	You are running Nginx as appuser (UID 1000), which does not have permission to bind to port 80.


kubectl expose deployment nginx-rapid --type=NodePort --port=8080 -n rapid

[ec2-user@ip-10-0-4-172 ~]$ kubectl get svc -n rapid
NAME          TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
nginx-rapid   NodePort   172.20.10.49   <none>        8080:32453/TCP   5m21s


[ec2-user@ip-10-0-4-172 ~]$ kubectl get nodes -o wide
NAME                         STATUS   ROLES    AGE   VERSION                INTERNAL-IP   EXTERNAL-IP   OS-IMAGE         KERNEL-VERSION                  CONTAINER-RUNTIME
ip-10-0-1-248.ec2.internal   Ready    <none>   29h   v1.27.16-eks-aeac579   10.0.1.248    <none>        Amazon Linux 2   5.10.234-225.910.amzn2.x86_64   containerd://1.7.25
ip-10-0-2-158.ec2.internal   Ready    <none>   29h   v1.27.16-eks-aeac579   10.0.2.158    <none>        Amazon Linux 2   5.10.234-225.910.amzn2.x86_64   containerd://1.7.25

- access from jump server:
[ec2-user@ip-10-0-4-172 ~]$ curl http://10.0.1.248:32453
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>


with service:
------------
kubectl delete -f rapid-ngnix.yaml 
kubectl apply -f rapid-ngnix.yaml 
kubectl apply -f rapid-service.yaml 


implementation:
--------------
[ec2-user@ip-10-0-4-172 ~]$ kubectl apply -f rapid-ngnix.yaml -n rapid
deployment.apps/nginx-rapid created
[ec2-user@ip-10-0-4-172 ~]$ kubectl apply -f rapid-service.yaml -n rapid
service/nginx-rapid-service created


[ec2-user@ip-10-0-4-172 ~]$ kubectl get pods -n rapid
NAME                           READY   STATUS    RESTARTS   AGE
nginx-rapid-6d5788ff99-q9plm   1/1     Running   0          34s
nginx-rapid-6d5788ff99-x7kbr   1/1     Running   0          34s
[ec2-user@ip-10-0-4-172 ~]$ kubectl get svc -n rapid
NAME                  TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
nginx-rapid-service   NodePort   172.20.147.123   <none>        80:30080/TCP   28s

[ec2-user@ip-10-0-4-172 ~]$ kubectl get nodes -n rapid -o wide
NAME                         STATUS   ROLES    AGE   VERSION                INTERNAL-IP   EXTERNAL-IP   OS-IMAGE         KERNEL-VERSION                  CONTAINER-RUNTIME
ip-10-0-1-248.ec2.internal   Ready    <none>   29h   v1.27.16-eks-aeac579   10.0.1.248    <none>        Amazon Linux 2   5.10.234-225.910.amzn2.x86_64   containerd://1.7.25
ip-10-0-2-158.ec2.internal   Ready    <none>   29h   v1.27.16-eks-aeac579   10.0.2.158    <none>        Amazon Linux 2   5.10.234-225.910.amzn2.x86_64   containerd://1.7.25

-> GOTO JUMP SERVER AND ACCESS SERVICE

[ec2-user@ip-10-0-4-172 ~]$ curl http://10.0.1.248:30080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>


```



# Access Application using ALB Ingress, even though your pods are in private subnet:

```
- you can access the application by creating loadbalancer service, but your pods are in private subnet
- to access your pods which are in private subnet you can use ingress controller
- access application which is inside private subnet eks cluster using alb ingreess 
1.create a oidc provider
2.create iam policy and servcie account
3.install controller using helm
4.deploy application
4.access the application using ALB


cluster name: demo-eks-QGGshN7P

Step 1: Create an OIDC Provider for EKS
---------------------------------------
AWS Load Balancer Controller requires an IAM role, and EKS uses OIDC to authenticate.

Get your EKS OIDC provider:
aws eks describe-cluster --name demo-eks-QGGshN7P --query "cluster.identity.oidc.issuer" --output text

[ec2-user@ip-10-0-4-172 ~]$ aws eks describe-cluster --name demo-eks-QGGshN7P --query "cluster.identity.oidc.issuer" --output text
https://oidc.eks.us-east-1.amazonaws.com/id/CD9CE8AC4BB2965EA957C164C2DE9971


INSTALL EKSCTL ON JUMP SERVER:
------------------------------
sudo curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | sudo tar xz -C /usr/local/bin
[ec2-user@ip-10-0-4-172 ~]$ eksctl version
0.206.0


Next Step: Associate OIDC Provider with IAM
Now, you need to associate this OIDC provider with AWS IAM so that the ALB Ingress Controller can authenticate securely.
Create the OIDC provider:
eksctl utils associate-iam-oidc-provider \
  --region us-east-1 \
  --cluster demo-eks-QGGshN7P \
  --approve
  
  2025-03-30 19:36:30 [ℹ]  IAM Open ID Connect provider is already associated with cluster "demo-eks-QGGshN7P" in "us-east-1"

Your OIDC provider is already associated with the EKS cluster!


 Step 2: Create IAM Policy for ALB Ingress Controller
-------------------------------------------------------
The AWS Load Balancer Controller needs an IAM policy to manage ALB resources.
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json


aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
  
  
[ec2-user@ip-10-0-4-172 ~]$ aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
{
    "Policy": {
        "PolicyName": "AWSLoadBalancerControllerIAMPolicy",
        "PolicyId": "ANPAYS2NUTC7Q3UHNZ6E2",
        "Arn": "arn:aws:iam::590183962815:policy/AWSLoadBalancerControllerIAMPolicy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2025-03-30T19:42:31+00:00",
        "UpdateDate": "2025-03-30T19:42:31+00:00"
    }
}


 Copy the IAM Policy ARN, as we will use it in the next step.
"Arn": "arn:aws:iam::590183962815:policy/AWSLoadBalancerControllerIAMPolicy",



 Step 3: Create IAM Role & Service Account for ALB Controller
-------------------------------------------------------------
Now, we will attach the IAM policy to an IAM role and Kubernetes service account.

eksctl create iamserviceaccount \
  --cluster demo-eks-QGGshN7P \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn arn:aws:iam::590183962815:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve



- it will craete a role with that policy name
  The AWS Load Balancer Controller runs in the kube-system namespace, but it can manage ALBs for applications in any namespace, including your rapid namespace.
  So yes, your setup will work because:
  The controller pod runs in kube-system.
  It watches Ingress resources across all namespaces.
  Your application pods stay in rapid, but the ALB will still route traffic to them.


  [ec2-user@ip-10-0-4-172 ~]$ eksctl create iamserviceaccount \
    --cluster demo-eks-QGGshN7P \
    --namespace kube-system \
    --name aws-load-balancer-controller \
    --role-name AmazonEKSLoadBalancerControllerRole \
    --attach-policy-arn arn:aws:iam::590183962815:policy/AWSLoadBalancerControllerIAMPolicy \
    --approve
  2025-03-30 19:55:51 [ℹ]  1 iamserviceaccount (kube-system/aws-load-balancer-controller) was included (based on the include/exclude rules)
  2025-03-30 19:55:51 [!]  serviceaccounts that exist in Kubernetes will be excluded, use --override-existing-serviceaccounts to override
  2025-03-30 19:55:51 [ℹ]  1 task: { 
      2 sequential sub-tasks: { 
          create IAM role for serviceaccount "kube-system/aws-load-balancer-controller",
          create serviceaccount "kube-system/aws-load-balancer-controller",
      } }2025-03-30 19:55:51 [ℹ]  building iamserviceaccount stack "eksctl-demo-eks-QGGshN7P-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
  2025-03-30 19:55:52 [ℹ]  deploying stack "eksctl-demo-eks-QGGshN7P-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
  2025-03-30 19:55:52 [ℹ]  waiting for CloudFormation stack "eksctl-demo-eks-QGGshN7P-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
  2025-03-30 19:56:22 [ℹ]  waiting for CloudFormation stack "eksctl-demo-eks-QGGshN7P-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
  2025-03-30 19:56:22 [ℹ]  created serviceaccount "kube-system/aws-load-balancer-controller"


If successful, verify the IAM Service Account:
  [ec2-user@ip-10-0-4-172 ~]$ kubectl get sa -n kube-system | grep aws-load-balancer-controller
  aws-load-balancer-controller         0         51s
  [ec2-user@ip-10-0-4-172 ~]$ 




Step 4: Install AWS Load Balancer Controller Using Helm
--------------------------------------------------------
INSTALL HELM:
-------------
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh


Add the Helm repo & update:
helm repo add eks https://aws.github.io/eks-charts
helm repo update


[ec2-user@ip-10-0-4-172 ~]$ helm repo add eks https://aws.github.io/eks-charts
"eks" has been added to your repositories

[ec2-user@ip-10-0-4-172 ~]$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "eks" chart repository
Update Complete. ⎈Happy Helming!⎈
[ec2-user@ip-10-0-4-172 ~]$ 




Install the ALB Controller:
---------------------------
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=demo-eks-QGGshN7P \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcID=vpc-0774623cf3a8aad1e
  
- IN THIS VPC vpc-0774623cf3a8aad1e OUR CLUSTER IS CREATED

[ec2-user@ip-10-0-4-172 ~]$ helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=demo-eks-QGGshN7P \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
  --set region=us-east-1 \
  --set vpcID=vpc-0774623cf3a8aad1e
NAME: aws-load-balancer-controller
LAST DEPLOYED: Sun Mar 30 20:05:32 2025
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
AWS Load Balancer controller installed!

[ec2-user@ip-10-0-4-172 ~]$ kubectl get pods -n kube-system | grep aws-load-balancer-controller
aws-load-balancer-controller-8448cc5b89-tzb4x   1/1     Running   0          36s
aws-load-balancer-controller-8448cc5b89-x6fb2   1/1     Running   0          36s


[ec2-user@ip-10-0-4-172 ~]$ kubectl logs -n kube-system deployment/aws-load-balancer-controller
Found 2 pods, using pod/aws-load-balancer-controller-8448cc5b89-tzb4x
{"level":"info","ts":"2025-03-30T20:05:36Z","msg":"version","GitVersion":"v2.12.0","GitCommit":"ab69d951f7f42ce91f5d7880118b8fc7937794f1","BuildDate":"2025-03-10T17:20:40+0000"}
{"level":"info","ts":"2025-03-30T20:05:36Z","logger":"setup","msg":"adding health check for controller"}
{"level":"info","ts":"2025-03-30T20:05:36Z","logger":"setup","msg":"adding readiness check for webhook"}
{"level":"info","ts":"2025-03-30T20:05:36Z","logger":"controller-runtime.webhook","msg":"Registering webhook","path":"/mutate-v1-pod"}
{"level":"info","ts":"2025-03-30T20:05:36Z","logger":"controller-runtime.webhook","msg":"Registering webhook","path":"/mutate-v1-service"}
{"level":"info","ts":"2025-03-30T20:05:36Z","logger":"controller-runtime.webhook","msg":"Registering webhook","path":"/validate-elbv2-k8s-aws-v1beta1-ingressclassparams"}
{"level":"info","ts":"2025-03-30T20:05:36Z","logger":"controller-runtime.webhook","msg":"Registering webhook","path":"/mutate-elbv2-k8s-aws-v1beta1-targetgroupbinding"}
{"level":"info","ts":"2025-03-30T20:05:36Z","logger":"controller-runtime.webhook","msg":"Registering webhook","path":"/validate-elbv2-k8s-aws-v1beta1-targetgroupbinding"}
{"level":"info","ts":"2025-03-30T20:05:36Z","logger":"controller-runtime.webhook","msg":"Registering webhook","path":"/validate-networking-v1-ingress"}
{"level":"info","ts":"2025-03-30T20:05:36Z","logger":"setup","msg":"starting podInfo repo"}
{"level":"info","ts":"2025-03-30T20:05:36Z","logger":"setup","msg":"starting deferred tgb reconciler"}
{"level":"info","ts":"2025-03-30T20:05:38Z","logger":"controller-runtime.metrics","msg":"Starting metrics server"}
{"level":"info","ts":"2025-03-30T20:05:38Z","logger":"controller-runtime.webhook","msg":"Starting webhook server"}
{"level":"info","ts":"2025-03-30T20:05:38Z","logger":"controller-runtime.metrics","msg":"Serving metrics server","bindAddress":":8080","secure":false}
{"level":"info","ts":"2025-03-30T20:05:38Z","msg":"starting server","name":"health probe","addr":"[::]:61779"}
{"level":"info","ts":"2025-03-30T20:05:38Z","logger":"controller-runtime.certwatcher","msg":"Updated current TLS certificate"}
{"level":"info","ts":"2025-03-30T20:05:38Z","logger":"controller-runtime.webhook","msg":"Serving webhook server","host":"","port":9443}
{"level":"info","ts":"2025-03-30T20:05:38Z","logger":"controller-runtime.certwatcher","msg":"Starting certificate watcher"}
{"level":"info","ts":"2025-03-30T20:05:38Z","msg":"attempting to acquire leader lease kube-system/aws-load-balancer-controller-leader..."}



step 5: Deploy sample application:
----------------------------------
[ec2-user@ip-10-0-4-172 test]$ cat non-root-deployment.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-rapid
  namespace: rapid
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-rapid
  template:
    metadata:
      labels:
        app: nginx-rapid
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000  # Ensures correct file system permissions
      containers:
        - name: nginx-rapid
          image: testproject1234.jfrog.io/test-docker-docker-local/rapid-nginx:v2  # Use 'latest' tag
          imagePullPolicy: Always  # Ensures the latest image is always pulled
          ports:
            - containerPort: 8080
          securityContext:
            runAsNonRoot: true
            runAsUser: 1000
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
      imagePullSecrets:
        - name: artifactory-secret  # Ensure this secret exists in the "rapid" namespace

---------------------

[ec2-user@ip-10-0-4-172 test]$ cat service.yaml 
apiVersion: v1
kind: Service
metadata:
  name: nginx-rapid-service
  namespace: rapid
spec:
  selector:
    app: nginx-rapid
  ports:
    - protocol: TCP
      port: 80        # Exposed service port
      targetPort: 8080  # Container port
  type: NodePort
[ec2-user@ip-10-0-4-172 test]$ 

-----------------------

[ec2-user@ip-10-0-4-172 test]$ cat ingress.yaml 
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-rapid-ingress
  namespace: rapid
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
    alb.ingress.kubernetes.io/group.name: "nginx-ingress-group"
    alb.ingress.kubernetes.io/healthcheck-path: "/"
    alb.ingress.kubernetes.io/subnets: subnet-094ec44e1d74a96ae, subnet-089d86a458321b72c #these are public subnets of cluster vpc where it got deployed, son in this subnets iam deploying load balancer
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-rapid-service
                port:
                  number: 80

-------------------------------

kubectl apply -f non-root-deployment.yaml -n rapid

kubectl get pods -n rapid

[ec2-user@ip-10-0-4-172 test]$ kubectl apply -f service.yaml -n rapid
service/nginx-rapid-service created

[ec2-user@ip-10-0-4-172 test]$ kubectl get svc -n rapid
NAME                  TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx-rapid-service   NodePort   172.20.66.216   <none>        80:31400/TCP   5s

[ec2-user@ip-10-0-4-172 test]$ kubectl get svc -n kube-system
NAME                                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
aws-load-balancer-webhook-service   ClusterIP   172.20.211.51   <none>        443/TCP         33m
kube-dns                            ClusterIP   172.20.0.10     <none>        53/UDP,53/TCP   2d14h


[ec2-user@ip-10-0-4-172 test]$ kubectl apply -f ingress.yaml 
ingress.networking.k8s.io/nginx-rapid-ingress created

[ec2-user@ip-10-0-4-172 test]$ kubectl get ingress -A

[ec2-user@ip-10-0-4-172 test]$ kubectl get ingress -n rapid
NAME                  CLASS   HOSTS   ADDRESS                                                                  PORTS   AGE
nginx-rapid-ingress   alb     *       k8s-nginxingressgroup-db8a477944-434554598.us-east-1.elb.amazonaws.com   80      35s
[ec2-user@ip-10-0-4-172 test]$ 


- wait for sometime, it will craete loadbalancer, you can check there
- check rules 
- default port of load balancer is 80
- check target groups and health checks
- in lb, we can use annotations to route traffic from 80 to 443 -> https and attach certifcates
- when the loadbalancer state is active, we can use dns name of load balancer to access that

- once lb is available, access it 
k8s-nginxingressgroup-db8a477944-434554598.us-east-1.elb.amazonaws.com

if you open still it will redirect to https
manually chnage to http
```



