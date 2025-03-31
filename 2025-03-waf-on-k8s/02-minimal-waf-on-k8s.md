# QuickStart - CloudGuard WAF as deployment in Kubernetes (AKS)

### Small AKS cluster

```bash
# verify you are logged in to Azure CLI
az account show -o table

# start small cluster
az group create --location northeurope -g aks
az aks create --resource-group aks --name aks1 --node-count 1 --node-vm-size Standard_B2s --enable-managed-identity --generate-ssh-keys

# fetch credentials
az aks get-credentials --resource-group aks --name aks1

# check our new cluster
kubectl get nodes -o wide
kubectl get po -o wide -A

# install CertManager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.17.0/cert-manager.yaml
```

### WAF setup

Consider reading HELM chart building blocks at https://github.com/mkol5222/appsec-chart/tree/main/charts/simple-waf/templates
in following order:
- secret.yaml - where WAF agent token is stored
- issuer.yaml and cert.yaml - where certificate is requested
- deployment.yaml - where single managed WAF containers are deployed
- service.yaml - how it is exposed to Internet


Progress with setup using commands below:

```shell
# WAF profile for our agents deployed (Docker single managed profile)
# create profile and paste value here
export CPTOKEN="cp-bfeae006-5cef-476e-ae20-955ba17d9b88625d1655-2871-473b-ac09-a60a0765a06d"
# host exposed - keep for demo
export WAF_HOST=hello.klaud.online

# reminder - this is how cert manager WAS installed above
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.17.0/cert-manager.yaml
# check it - all running, Ctrl-C
kubectl get po -n cert-manager -w

# deploy simple WAF setup
helm install waf https://github.com/mkol5222/appsec-chart/releases/download/simple-waf-0.0.1/simple-waf-0.0.1.tgz --set cptoken=$CPTOKEN --set hostname=$WAF_HOST 
 
# verify WAF deployment exposed on load balancer with Public IP address:
kubectl get svc appsec -o wide -n default -w

# access it via Public IP
pip=$(kubectl get svc appsec -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo $pip

curl https://hello.klaud.online --resolve hello.klaud.online:443:$pip -k -v
curl "https://hello.klaud.online/?z=cat+/etc/passwd" --resolve hello.klaud.online:443:$pip -k -v

```

### CLEANUP
```shell
az aks list -o table -g aks
# delete RG with the cluster and all resources
az aks delete --resource-group aks --name aks1 -y --no-wait
az group delete --resource-group aks
# check it is gone
az aks list -o table  aks
az group list -o table | sls aks
```