You can enforce this policy using OPA Gatekeeper by creating a **ConstraintTemplate** that ensures all Pods are not running as the root user.

### **Step 1: Create the Constraint Template**
This template will check the `securityContext.runAsNonRoot` field in Pod specifications.

Create a file named **`k8sno-root-pods_template.yaml`**:

```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8snorootpods
spec:
  crd:
    spec:
      names:
        kind: K8sNoRootPods
      validation:
        openAPIV3Schema:
          properties: {}
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8snorootpods
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.securityContext.runAsNonRoot
          msg := sprintf("Pod must not run as root: %v", [input.review.object.metadata.name])
        }
```

Apply the ConstraintTemplate:

```sh
kubectl apply -f k8sno-root-pods_template.yaml
```

---

### **Step 2: Create the Constraint**
Now, create a **Constraint** that enforces this rule.

Create a file named **`no-root-pods.yaml`**:

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sNoRootPods
metadata:
  name: enforce-no-root-pods
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
```

Apply the Constraint:

```sh
kubectl apply -f no-root-pods.yaml
```

---

### **Step 3: Test the Policy**
#### **Test a Pod that violates the policy**
Create a file named **`bad-pod.yaml`** (this will be denied):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: bad-pod
spec:
  containers:
    - name: nginx
      image: nginx
```

Try deploying it:

```sh
kubectl apply -f bad-pod.yaml
```

🚨 **This should be denied with an error message:**
```
Error from server (Forbidden): admission webhook "validation.gatekeeper.sh" denied the request: Pod must not run as root: bad-pod
```

---

#### **Test a Pod that complies with the policy**
Create a file named **`good-pod.yaml`**:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: good-pod
spec:
  containers:
    - name: nginx
      image: nginx
      securityContext:
        runAsNonRoot: true
```

Deploy it:

```sh
kubectl apply -f good-pod.yaml
```

✅ **This should be allowed.**

---

### **Summary**
- We created a ConstraintTemplate (`k8sno-root-pods_template.yaml`) that checks if `runAsNonRoot` is set.
- We enforced this with a Constraint (`no-root-pods.yaml`).
- We tested the policy with a bad and a good Pod.

This ensures that no Pods run as the root user! 🚀


# TEST 2

your **"no-root-pods"** policy will apply to Pods deployed in the `rapid` namespace **unless** the namespace is explicitly excluded in the Constraint.  

---

### **How Constraints Apply to Namespaces**
- The **Constraint** applies to all Pods across all namespaces by default.
- If you want to **exclude** certain namespaces (e.g., `rapid`), you must explicitly mention them in the `excludedNamespaces` field.

---

### **Check Your Current Constraint**
Your current **Constraint (`no-root-pods.yaml`)** applies globally:

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sNoRootPods
metadata:
  name: enforce-no-root-pods
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
```

🔹 **Since no namespace exclusions are defined, the policy applies to all namespaces, including `rapid`.**

---

### **Modify Constraint to Exclude `rapid` Namespace**
If you **do not** want this policy to apply in the `rapid` namespace, update the Constraint (`no-root-pods.yaml`):

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sNoRootPods
metadata:
  name: enforce-no-root-pods
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces:
      - rapid
```

Apply the updated constraint:

```sh
kubectl apply -f no-root-pods.yaml
```

🚀 **Now, the policy will not be enforced in the `rapid` namespace.**

---

### **Test in the `rapid` Namespace**
#### **Deploy a Root Pod in `rapid` (should be allowed)**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: root-pod
  namespace: rapid
spec:
  containers:
    - name: nginx
      image: nginx
```

```sh
kubectl apply -f root-pod.yaml
```

✅ **This should be allowed because `rapid` is excluded.**

---

#### **Deploy a Root Pod in another namespace (should be denied)**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: root-pod
  namespace: default
spec:
  containers:
    - name: nginx
      image: nginx
```

```sh
kubectl apply -f root-pod.yaml
```

🚨 **This should be denied because the default namespace is not excluded.**

---

### **Conclusion**
- If you deploy your Pod in the `rapid` namespace **with the current constraint**, it **will be affected** by the policy.
- If you **want to exclude `rapid`**, add it to `excludedNamespaces` in `no-root-pods.yaml`.

Let me know if you need any modifications! 🚀
