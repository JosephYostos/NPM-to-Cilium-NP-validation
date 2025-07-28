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

## endport 

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
