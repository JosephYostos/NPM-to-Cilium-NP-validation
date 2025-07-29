# NPM-to-Cilium-NP-validation

# cluster setup 

AKS cluster with cilium (latest)

```bash
az aks create \
    --name Joe-cilium  \
    --resource-group Joe-RG \
    --location westus\
    --network-plugin azure \
    --network-plugin-mode overlay \
    --network-dataplane cilium \
    --generate-ssh-keys
```

AKS cluster with NPM

```bash
az aks create \
    --name Joe-npm \
    --resource-group Joe-RG \
    --location westus\
    --network-plugin azure \
    --network-plugin-mode overlay \
    --network-policy azure \
    --generate-ssh-keys
```

validate cilium version

```bash
kubectl -n kube-system get pods -l k8s-app=cilium -o=jsonpath="{.items[0].spec.containers[0].image}"
```

output: 
```
mcr.microsoft.com/containernetworking/cilium/cilium:v1.17.4-250610
```

# endport 

Deploy the endport-test.yml, this will create the following resources: 

1. Create Namespace "endport-test"
2. Deploy Server Pod "This pod runs a simple Python HTTP server on a range of ports."
3. Deploy Client Pod
4. Apply NetworkPolicy withendPort

```
kubectl apply -f endport-test.yml
```
Test Connectivity

```
curl <IP of test-server>:32000
curl <IP of test-server>:32010
```

# ipBlock 

Deploy the ipblock-test.yml, this will create the following resources: 

1. Create Namespace "ipblock-test"
2. Deploy Server Pod 
3. Deploy Client Pod
4. Apply NetworkPolicy withendPort "This policy allows egress to all IPs in 0.0.0.0/0"

```
kubectl apply -f ipblock-test.yml
```

**Test on NPM**

1. Get the IP of the test-server pod and node IPs :

```
kubectl get pod test-server -n ipblock-test -o wide
kubectl get nodes -o wide
```
2. From the test-client pod, run:

```
curl --max-time 3 <server-pod-ip>:8080
curl --max-time 3 <node-ip>:10250
```
Expected Result (NPM): Traffic is allowed to both Pod IP and node IP.

**Test on Cilium**

Repeat the same steps on a cluster running Cilium.

Expected Result (Cilium): Traffic to the Pod IP and the node IP is blocked, even though it matches the CIDR.

**Remediation Steps:**
- For Pod IPs: before migration to cilium NP, Add namespaceSelector and podSelector to explicitly allow traffic to pods:

```
egress:
  - to:
      - ipBlock:
          cidr: 0.0.0.0/0
      - namespaceSelector: {}
      - podSelector: {}
```
- For Node IPs: You must apply a CiliumNetworkPolicy after migration to allow egress to node IPs.

example of CiliumNetworkPolicy to allow access to local and remote nodes

```
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: allow-node-egress
  namespace: ipblock-test
spec:
  endpointSelector: {}  # Applies to all pods in the namespace
  egress:
     - toEntities:
          - host # host allows traffic from/to the local node’s host network namespace
          - remote-node 
```
# named port 

Deploy the named-port-test.yml, this will create the following resources: 

1. Create Namespace "named-port-test"
2. Deploy Server Pod with Named Port
3. Deploy another Server Pod with the same Named Port and different port
4. Deploy Client Pod
5. Apply NetworkPolicy withendPort "This policy allows egress to all IPs in 0.0.0.0/0"

Test Connectivity
From the test-client pod:
```
curl --max-time 3 http://<Ip of test-server-1>:8080
curl --max-time 3 http://<Ip of test-server-2>:8000
```
# NetworkPolicy with Egress Policies (not Allow All)


==================================
NOTES:
Named ports working on both NPM & cilium ---> https://github.com/cilium/cilium/issues/30003
IPblock for node IPs works with -host & remote nodes
is it ok to make this script public 

NetworkPolicy with Egress Policies (not Allow All)
