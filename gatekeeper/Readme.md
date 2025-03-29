# k8s-opa-gatekeeper

This project is demoing OPA Gatekeeper with Kubernetes. Gatekeeper is an opensource project supported by CNCF. It aims for creating policies to manage Kubernetes like:

- Enforcing labels on namespaces.
- Allowing only images coming from certain Docker Registries.
- Require all Pods specify resource requests and limits.
- Prevent conflicting Ingress objects from being created.
- Enforcing running containers as non-root.

# Deploy OPA Gatekeeper
```
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml
```

# Scenario: Enforce Having A Specific Label In Any New Namespace
# Deploy the Contraint Template
```
kubectl apply -f k8srequiredlabels_template.yaml
```

# Deploy the Constraints
```
kubectl apply -f all_ns_must_have_gatekeeper.yaml
```

# Deploy a namespace denied by Policy
```
kubectl apply -f bad-namespace.yaml
```

# Deploy a namespace allowed by Policy
```
kubectl apply -f good-namespace.yaml
```

# implementation
```
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

```
