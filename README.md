# k8s_questions
# Kubernetes Interview Questions and Answers

This repository contains common scenario-based Kubernetes interview questions and detailed answers to help you prepare for DevOps and SRE interviews.

## Table of Contents

1. [High Traffic and Pod Termination](#1-high-traffic-and-pod-termination)
2. [Node Maintenance Without Disruption](#2-node-maintenance-without-disruption)
3. [Secure Access to Sensitive Data](#3-secure-access-to-sensitive-data)
4. [Data Persistence Across Pod Restarts](#4-data-persistence-across-pod-restarts)
5. [Multi-Environment Deployments](#5-multi-environment-deployments)
6. [Network Connectivity Issues](#6-network-connectivity-issues)
7. [Zero-Downtime Updates](#7-zero-downtime-updates)
8. [Secure External API Communication](#8-secure-external-api-communication)
9. [Troubleshooting Pod Crashes](#9-troubleshooting-pod-crashes)
10. [Restricting Pod Scheduling](#10-restricting-pod-scheduling)
11. [Implementing Canary Deployments](#11-implementing-canary-deployments)
12. [Running Parallel Jobs](#12-running-parallel-jobs)

---

## 1. High Traffic and Pod Termination

**Question:**  
Your application suddenly experiences high traffic and your Kubernetes Pods are getting terminated. What could be the issue and how would you solve it?

**Answer:**  
This is likely a resource constraint issue where Pods are being OOM (Out of Memory) killed.

**Steps to diagnose and fix:**
- Check Pod status and events: `kubectl describe pod <pod-name>`
- Look for OOMKilled or eviction events.
- Review resource metrics: 
```
kubectl top pod
kubectl top nodes
```
- Adjust resource requests and limits in the deployment YAML.
```
resources:
  requests:
    memory: "256Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```
- Consider implementing Horizontal Pod Autoscaler (HPA).
```
kubectl autoscale deployment <deployment-name> --cpu-percent=80 --min=3 --max=10
```
- If node resources are exhausted, scale the cluster by adding more nodes.

---

## 2. Node Maintenance Without Disruption

**Question:**  
You need to perform maintenance on a specific node without disrupting application availability. How would you proceed?

**Answer:**  
I would use node draining to safely evict all Pods from the node before maintenance.

**Steps:**
- Mark the node as unschedulable (cordon).
  ```
  kubectl cordon <node-name>
  ```
- Drain the node (evict existing Pods).
```
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```
- Perform maintenance on the node.
- Make the node schedulable again.
```
kubectl uncordon <node-name>
```

---

## 3. Secure Access to Sensitive Data

**Question:**  
Your application needs to access sensitive data like database credentials. What's the most secure way to handle this in Kubernetes?

**Answer:**  
I would use Kubernetes Secrets combined with RBAC (Role-Based Access Control) for secure credential management.

**Implementation steps:**
- Create a Secret with the sensitive data.
```
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=s3cur3p@ssw0rd
```

- Mount the Secret as environment variables or files in the Pod.
```
spec:
  containers:
  - name: app
    image: my-app:1.0
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
```

- Implement RBAC to restrict access to the Secret.

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["db-credentials"]
  verbs: ["get"]
```
- For production environments, consider using external secret management systems like HashiCorp Vault, AWS Secrets Manager, or GCP Secret Manager with appropriate operators.

---

## 4. Data Persistence Across Pod Restarts

**Question:**  
Your application needs to ensure data persistence across Pod restarts. How would you implement this?

**Answer:**  
I would use Persistent Volumes (PVs) and Persistent Volume Claims (PVCs) to ensure data persistence.

**Implementation:**
- Define a StorageClass (if using dynamic provisioning).
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
```
- Create a PVC.
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast
  resources:
    requests:
      storage: 10Gi
```
- Mount the PVC in the Pod.
```
spec:
  containers:
  - name: app
    image: my-app:1.0
    volumeMounts:
    - mountPath: "/data"
      name: app-data-volume
  volumes:
  - name: app-data-volume
    persistentVolumeClaim:
      claimName: app-data
```
- For stateful applications, consider using StatefulSets instead of Deployments to maintain stable network identities and storage.

---

## 5. Multi-Environment Deployments

**Question:**  
You need to deploy your application across multiple environments (dev, staging, prod) with different configurations. How would you manage this?

**Answer:**  
I would use a combination of Kubernetes namespaces and ConfigMaps along with a templating tool like Helm or Kustomize.

**Using Kustomize approach:**
- Create a base manifest structure.
- Create overlay directories for each environment.
- In each environment's `kustomization.yaml`, specify the base and patches.
- Deploy to specific environment.

**Using Helm approach:**
- Create a Helm chart with templates.
- Use values files for environment-specific configurations.
```
values-dev.yaml
values-staging.yaml
values-prod.yaml
```
- Deploy to specific environment.
```
helm upgrade --install myapp ./mychart -f values-prod.yaml -n prod
```

---

## 6. Network Connectivity Issues

**Question:**  
Your Kubernetes cluster is experiencing network connectivity issues between Pods. How would you troubleshoot and resolve this?

**Answer:**  
I would follow a systematic approach to diagnose and fix the network issues.

**Troubleshooting steps:**
- Verify the cluster's network plugin is functioning.
```
kubectl get pods -n kube-system | grep -E 'calico|flannel|weave|cilium'
```
- Check if CoreDNS is working properly.
```
kubectl get pods -n kube-system | grep coredns
```
- Test basic connectivity using debugging Pods.
```
kubectl run --rm -it network-test --image=nicolaka/netshoot -- /bin/bash
```
- From the debugging Pod, test connectivity.
```
ping <service-name>
nslookup <service-name>
curl <service-name>:<port>
```
- Check if NetworkPolicies are blocking traffic.
```
kubectl get networkpolicies --all-namespaces
```
- Verify service endpoints.
```
kubectl get endpoints <service-name>
```

**Common solutions:**
- Restart the network plugin Pods.
- Check for restrictive NetworkPolicies and adjust them.
- Ensure Services and Pods have matching labels and selectors.
- Verify that cluster CIDR ranges don't overlap with existing network infrastructure.

---

## 7. Zero-Downtime Updates

**Question:**  
You need to perform a zero-downtime update of your application. How would you implement this?

**Answer:**  
I would use a rolling update strategy with readiness probes to ensure zero downtime.

**Implementation:**
- Configure the deployment with a proper update strategy.
```
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```
- Implement a readiness probe to ensure traffic is only sent to ready Pods.
```
readinessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```
- Deploy the updated application.
```
kubectl apply -f deployment.yaml
```

---

## 8. Secure External API Communication

**Question :**  
Your application needs to communicate with an external API that requires authentication credentials. What's the most secure way to handle this in Kubernetes?

**Answer:**  
For secure external API authentication, use Kubernetes Secrets combined with proper Pod security policies.

**Implementation steps:**
- Create a Secret containing the API credentials.
  ```
kubectl create secret generic api-credentials \
  --from-literal=api-key=abc123 \
  --from-literal=api-secret=xyz789
```
- Mount the Secret as environment variables in the deployment.
```
spec:
  containers:
  - name: app
    image: my-app:1.0
    env:
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: api-credentials
          key: api-key
    - name: API_SECRET
      valueFrom:
        secretKeyRef:
          name: api-credentials
          key: api-secret
```
- Implement Pod Security Context to enhance security.

```
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
  containers:
  - name: app
    image: my-app:1.0
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
``
- In production environments, consider using a service mesh like Istio for more advanced authentication mechanisms or external secret management systems.

## 9. Troubleshooting Pod Crashes
**Question :**  
You notice that some of your Pods are frequently crashing and restarting. How would you troubleshoot and fix this issue?

**Answer:**  
I would follow a systematic approach to identify and resolve the Pod crashing issue.

**Troubleshooting steps:**
- Check the Pod status and restart count.
```
kubectl get pods
```
- Examine Pod events and logs.
```
kubectl describe pod <pod-name>
kubectl logs <pod-name> --previous
```
- Look for common issues like OOMKilled, CrashLoopBackOff, Error/ImagePullBackOff.
- Check resource utilization.
```
kubectl top pod <pod-name>
```
- Debug with an ephemeral container if necessary.
```
kubectl debug -it <pod-name> --image=busybox:1.28 --target=<container-name>
```

**Common solutions:**
- Adjust resource limits if OOM killed.
```
resources:
  requests:
    memory: "256Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```
- Add proper liveness and readiness probes.
```
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```
- Fix application code or configuration issues identified in logs.

## 10. Restricting Pod Scheduling

**Question :**  
You need to restrict Pod scheduling to specific nodes in your Kubernetes cluster. How would you achieve this?

**Answer:**  
I would use a combination of node selectors, affinity/anti-affinity rules, and taints/tolerations to control Pod scheduling.

**Using Node Selectors (simplest approach):**
- Label the nodes.
```
kubectl label nodes node1 workload=frontend
```
- Use `nodeSelector` in the Pod spec.
```
spec:
  nodeSelector:
    workload: frontend
```

**Using Node Affinity (more flexible):**
- Define node affinity rules in the Pod spec.
```
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: workload
            operator: In
            values:
            - frontend
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: instance-type
            operator: In
            values:
            - high-memory
```

**Using Taints and Tolerations:**
- Taint the nodes.
```
kubectl taint nodes node1 dedicated=frontend:NoSchedule
```
- Add tolerations to Pods that should be scheduled on those nodes.
```
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "frontend"
    effect: "NoSchedule"
```

## 11. Implementing Canary Deployments
**Question :**  
Your team needs to implement a canary deployment for a new version of your application. How would you set this up in Kubernetes?

**Answer:**  
I would implement a canary deployment using multiple Deployments with different versions and a Service to control traffic distribution.

**Implementation steps:**
- Create a stable deployment with the current version.
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: myapp
      version: stable
  template:
    metadata:
      labels:
        app: myapp
        version: stable
    spec:
      containers:
      - name: app
        image: myapp:1.0
```
- Create a canary deployment with the new version.
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-canary
spec:
  replicas: 1  # 10% of total replicas
  selector:
    matchLabels:
      app: myapp
      version: canary
  template:
    metadata:
      labels:
        app: myapp
        version: canary
    spec:
      containers:
      - name: app
        image: myapp:1.1
```
- Create a Service that selects both deployments.
```
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp  # Selects both versions
  ports:
  - port: 80
    targetPort: 8080
```
- Gradually increase the canary replicas and decrease stable replicas as confidence builds.
- For more advanced canary deployments, use a service mesh like Istio.
```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp
  http:
  - route:
    - destination:
        host: myapp
        subset: stable
      weight: 90
    - destination:
        host: myapp
        subset: canary
      weight: 10
```

## 12. Running Parallel Jobs
**Question :**  
You need to run a job that processes data in parallel across multiple Pods. How would you implement this in Kubernetes?

**Answer:**  
I would use a Kubernetes Job or CronJob with parallelism settings to process data concurrently.

**Implementation for a one-time parallel job:**
```
apiVersion: batch/v1
kind: Job
metadata:
  name: data-processor
spec:
  completions: 10    # Total number of successful Pod completions
  parallelism: 3     # Number of Pods that can run in parallel
  backoffLimit: 4    # Number of retries before job is considered failed
  template:
    spec:
      containers:
      - name: processor
        image: data-processor:1.0
        env:
        - name: BATCH_ID
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
      restartPolicy: Never
```
- Define the Job with parallelism and completions settings.
- Use a template to specify the Pod configuration.

**For recurring parallel jobs, use a CronJob:**
```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: data-processor-cron
spec:
  schedule: "0 2 * * *"  # Run daily at 2 AM
  jobTemplate:
    spec:
      completions: 10
      parallelism: 3
      template:
        spec:
          containers:
          - name: processor
            image: data-processor:1.0
          restartPolicy: Never
```
- Define the CronJob with a schedule and job template.
- Specify parallelism in the job template.
