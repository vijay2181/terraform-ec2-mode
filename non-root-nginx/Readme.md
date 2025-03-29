# run nginx as non root user

```

kubectl create secret docker-registry artifactory-secret --docker-server=testproject1234.jfrog.io --docker-username="devopsvijay10@gmail.com" --docker-password=”” --docker-email=”OPTIONAL_EMAIL”

kubectl create secret docker-registry artifactory-secret --docker-server=testproject1234.jfrog.io --docker-username="devopsvijay10@gmail.com" --docker-password=”” --docker-email=”OPTIONAL_EMAIL” -n rapid

docker build -t rapid-nginx:latest .
docker tag rapid-nginx:latest testproject1234.jfrog.io/test-docker-docker-local/rapid-nginx:v1
docker push testproject1234.jfrog.io/test-docker-docker-local/rapid-nginx:v1

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
```
