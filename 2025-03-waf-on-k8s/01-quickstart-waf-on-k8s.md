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

```powershell
### assume BASH !!!

### store CPTOKEN secret
# make sure create your own Docker Profile with Single Managed container and use your real CPTOKEN below
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: appsec
type: Opaque
stringData:
  cptoken: cp-bfeae006-5cef-476e-ae20-955ba17d9b88625d1655-2871-473b-ac09-a60a0765a06d 
EOF


kubectl get secret appsec -o jsonpath='{.data.cptoken}' | base64 -d; echo

# create certificate secret for hostname hello.klaud.online with CertManager
cat <<'EOF' | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-cluster-issuer
spec:
  selfSigned: {}

---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: hello-klaud-cert
  namespace: default
spec:
  secretName: hello-klaud-tls
  duration: 2160h # 90 days
  renewBefore: 360h # 15 days before expiry
  subject:
    organizations:
      - Klaud Online
  commonName: hello.klaud.online
  dnsNames:
    - hello.klaud.online
  issuerRef:
    name: selfsigned-cluster-issuer
    kind: ClusterIssuer
EOF

# check the logs of cert-manager
kubectl logs -n cert-manager deploy/cert-manager | grep hello

# check the secret
kubectl get secret hello-klaud-tls -n default -o yaml 


# deploy CloudGuard WAF
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: appsec
spec:
  replicas: 2  # Adjust the number of replicas as needed
  selector:
    matchLabels:
      app: appsec
  template:
    metadata:
      labels:
        app: appsec
    spec:
      containers:
      - name: cloudguard-appsec-standalone
        env:
        - name: CPTOKEN
          valueFrom:
            secretKeyRef:
              name: appsec
              key: cptoken
        # securityContext:
        #   runAsUser: 0
        #   runAsGroup: 0
        image: checkpoint/cloudguard-appsec-standalone:latest
        #image: checkpoint/cloudguard-appsec-standalone:787396
        args:
        - /cloudguard-appsec-standalone
        - --token
        - $(CPTOKEN)
        - --ignore-all
        # env:
        # - name: https_proxy
        #   value: "user:password@Proxy address:port"
        ports:
        - containerPort: 443  # SSL port
        - containerPort: 80   # HTTP port
        - containerPort: 8117 # Health-check port
        volumeMounts:
            - name: tls-secret
              mountPath: "/etc/certs/hello.pem"
              subPath: "tls.crt"
              readOnly: true
            - name: tls-secret
              mountPath: "/etc/certs/hello.key"
              subPath: "tls.key"
        # - name: certs
        #   mountPath: "/etc/certs2/"
        imagePullPolicy: Always
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /
            port: 8117
            scheme: HTTP
          periodSeconds: 20
          successThreshold: 1
          timeoutSeconds: 10
        startupProbe:
          failureThreshold: 90
          httpGet:
            path: /
            port: 8117
            scheme: HTTP
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 10
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      #schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
        - name: tls-secret
          secret:
            secretName: hello-klaud-tls
        - name: certs
          emptyDir: {}
EOF

# ceck the deployment
kubectl get deploy appsec -o wide -n default
kubectl get po -o wide -n default -l app=appsec

# enter WAF container
kubectl exec -it deploy/appsec -n default -- /bin/bash
# or just check it
kubectl exec -it deploy/appsec -n default -- cpnano -s
kubectl exec -it deploy/appsec -n default -- cpnano -s | grep Policy
kubectl exec -it deploy/appsec -n default -- nginx -T
kubectl exec -it deploy/appsec -n default -- nginx -T | grep hello

# expose it as service on Public IP
kubectl expose deploy appsec --type LoadBalancer --name appsec --port 80,443 -n default
# wait for External-IP to be assigned and Ctrl-C
kubectl get svc -o wide -n default -w

# access it via Public IP
PIP=$(kubectl get svc appsec -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "$PIP"
curl https://hello.klaud.online --resolve hello.klaud.online:443:$PIP -k -v
curl "https://hello.klaud.online/?z=cat+/etc/passwd" --resolve hello.klaud.online:443:$PIP -k -v

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