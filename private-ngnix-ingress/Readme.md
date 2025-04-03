# deploy application in private eks cluster without internet gateway



- nginx ingress controller is deployed, it is internal, we need to use existing one
- your vpc is private and doesn't have internet access to it

```
kubectl get ingress -A
NAMESPACE   NAME                  CLASS   HOSTS                                                                           ADDRESS                                                                         PORTS     AGE
rapid       nginx-ingress         nginx   k8s-ingressn-ingressn-861aa874d6-877187e88114cfc9.elb.eu-west-1.amazonaws.com   k8s-ingressn-ingressn-861aa874d6-877187e88114cfc9.elb.eu-west-1.amazonaws.com   80        24h
rapid       nginx-rapid-ingress   alb     *                                                                                                                                                               80        24h
```

- we need to use this nlb: k8s-ingressn-ingressn-861aa874d6-877187e88114cfc9.elb.eu-west-1.amazonaws.com

```
# Create DNS record
ELB_DNS="k8s-ingressn-ingressn-861aa874d6-877187e88114cfc9.elb.eu-west-1.amazonaws.com"
HOSTED_ZONE_ID=$(aws route53 list-hosted-zones-by-name --dns-name rapid.com --query 'HostedZones[0].Id' --output text)

cat > dns-record.json <<EOF
{
  "Changes": [{
    "Action": "CREATE",
    "ResourceRecordSet": {
      "Name": "demo.rapid.com",
      "Type": "CNAME",
      "TTL": 300,
      "ResourceRecords": [{ "Value": "$ELB_DNS" }]
    }
  }]
}
EOF

aws route53 change-resource-record-sets --hosted-zone-id $HOSTED_ZONE_ID --change-batch file://dns-record.json
```

- add jump server security group inside nlb security group on 443, ICMP ports
- ✅ Jump server security group allows 443 to Ingress.

```
[aws_install@ip-10-121-33-176 bin]$ curl -vk https://demo.rapid.com
*   Trying 10.121.33.133:443...
* Connected to demo.rapid.com (10.121.33.133) port 443
* ALPN: curl offers h2,http/1.1
* Cipher selection: ALL:!EXPORT:!EXPORT40:!EXPORT56:!aNULL:!LOW:!RC4:@STRENGTH
* TLSv1.2 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-AES128-GCM-SHA256
* ALPN: server accepted h2
* Server certificate:
*  subject: O=Acme Co; CN=Kubernetes Ingress Controller Fake Certificate
*  start date: Mar  6 20:20:59 2025 GMT
*  expire date: Mar  6 20:20:59 2026 GMT
*  issuer: O=Acme Co; CN=Kubernetes Ingress Controller Fake Certificate
*  SSL certificate verify result: unable to get local issuer certificate (20), continuing anyway.
* using HTTP/2
* [HTTP/2] [1] OPENED stream for https://demo.rapid.com/
* [HTTP/2] [1] [:method: GET]
* [HTTP/2] [1] [:scheme: https]
* [HTTP/2] [1] [:authority: demo.rapid.com]
* [HTTP/2] [1] [:path: /]
* [HTTP/2] [1] [user-agent: curl/8.3.0]
* [HTTP/2] [1] [accept: */*]
> GET / HTTP/2
> Host: demo.rapid.com
> User-Agent: curl/8.3.0
> Accept: */*
>
< HTTP/2 404
< date: Thu, 03 Apr 2025 18:00:13 GMT
< content-type: text/html
< content-length: 146
< strict-transport-security: max-age=31536000; includeSubDomains
<
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
* Connection #0 to host demo.rapid.com left intact
```

```
Your Load Balancer and Ingress are working, but your request is getting a 404 Not Found error. This means:

Traffic is reaching your Load Balancer (ELB) ✅

We see the SSL handshake, HTTP/2 communication, and response headers.

The request is reaching Nginx Ingress, but it’s not routing correctly ❌

The 404 Not Found is coming from Nginx, meaning no backend service is handling this request.

- since we have not craeted a backend service yet
```

```
Load Balancer only has a TCP 443 rule, which means:

It does not handle SSL termination.

Traffic is passed as-is to the Kubernetes cluster (Layer 4 - TCP).

Your Ingress Controller must handle TLS termination.
```

- your traffic should like this
  
```
Client → (HTTPS) → Nginx Ingress ✅

Nginx Ingress → (HTTP) → Backend Service ✅
```


# Generate a Self-Signed TLS Certificate (If Required)
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=demo.rapid.com/O=demo"

kubectl create secret tls demo-rapid-tls \
  --cert=tls.crt --key=tls.key

kubectl get secret demo-rapid-tls -n rapid

```

```
kubectl apply -f deployment.yaml
kubectl apply -f servcie.yaml
kubectl apply -f ingress.yaml
```

# Ensure the Jump Server Can Reach the Nginx Ingress
```
curl -vk https://demo.rapid.com
```






