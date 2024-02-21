# appworld-nginx-ingress-controller-api

# Windows:
Install Firefox
Install Chrome
Install VS Code
Install Postman
Update localhost file - run notepad as adminstrator

# NGINX External LB
Install NGINX+

adduser user01

usermod -aG sudo user01

su - user01

git clone https://github.com/nginxinc/nginx-loadbalancer-kubernetes.git

Backup existing NGINX .conf files

```
sudo mv /etc/nginx/conf.d/default.conf /etc/nginx/conf.d/default.conf.orig
sudo mv /etc/nginx/nginx.conf /etc/nginx/nginx.conf.orig
```

Copy the NLK files over to the /etc/nginx/


```
sudo mkdir /etc/nginx/stream
sudo cp nginx-loadbalancer-kubernetes/docs/tcp/default-tcp.conf /etc/nginx/conf.d/default-tcp.conf
sudo cp nginx-loadbalancer-kubernetes/docs/tcp/nginx.conf /etc/nginx/nginx.conf
sudo cp nginx-loadbalancer-kubernetes/docs/tcp/dashboard.conf /etc/nginx/conf.d/dashboard.conf 
sudo cp nginx-loadbalancer-kubernetes/docs/tcp/nginxk8slb.conf /etc/nginx/stream/nginxk8slb.conf
```

Validate nginx config and reload

```
nginx -t

systemctl restart nginx
```


# Docker Host


## Add Docker's official GPG key:
sudo apt-get update

sudo apt-get install ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository to Apt sources:
```
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin



adduser user01

usermod -aG sudo user01


sudo adduser user01 docker

newgrp docker


sudo snap install docker-compose

# MICROK8S

## microK8s install
https://ubuntu.com/tutorials/install-a-local-kubernetes-with-microk8s#2-deploying-microk8s


sudo snap install microk8s --classic

sudo ufw allow in on cni0 && sudo ufw allow out on cni0

sudo ufw default allow routed

sudo microk8s enable dns 

sudo microk8s enable dashboard

sudo microk8s enable storage


## helm install
sudo snap install helm --classic

## kubectl install
sudo snap install kubectl --classic

adduser user01

usermod -aG sudo user01

su - user01

sudo microk8s config

mkdir .kube

cd .kube

sudo microk8s config > config


## confirm installation 
kubectl get all --all-namespaces

## nginx+ ingress controller
mkdir /home/user01/nic

cd /home/user01/nic

git clone https://github.com/nginxinc/kubernetes-ingress.git --branch v3.3.1

cd kubernetes-ingress/deployments

kubectl apply -f common/ns-and-sa.yaml

kubectl apply -f rbac/rbac.yaml

kubectl apply -f rbac/ap-rbac.yaml

kubectl apply -f ../examples/shared-examples/default-server-secret/default-server-secret.yaml

kubectl apply -f common/nginx-config.yaml

kubectl apply -f common/nginx-config.yaml

kubectl apply -f common/ingress-class.yaml

kubectl apply -f common/crds/k8s.nginx.org_virtualservers.yaml

kubectl apply -f common/crds/k8s.nginx.org_virtualserverroutes.yaml

kubectl apply -f common/crds/k8s.nginx.org_transportservers.yaml

kubectl apply -f common/crds/k8s.nginx.org_policies.yaml

kubectl apply -f common/crds/k8s.nginx.org_globalconfigurations.yaml

kubectl apply -f common/crds/appprotect.f5.com_aplogconfs.yaml

kubectl apply -f common/crds/appprotect.f5.com_appolicies.yaml

kubectl apply -f common/crds/appprotect.f5.com_apusersigs.yaml

# nginx-plus JWT secret 
- pull N+ JWT token from myf5.com account

kubectl create secret docker-registry regcred --docker-server=private-registry.nginx.com kubectl create secret docker-registry regcred --docker-server=private-registry.nginx.com --docker-username=e.......... --docker-password=none -n nginx-ingress

## deployment ingress
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress
  namespace: nginx-ingress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-ingress
  template:
    metadata:
      labels:
        app: nginx-ingress
        app.kubernetes.io/name: nginx-ingress
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9113"
        prometheus.io/scheme: http
    spec:
      serviceAccountName: nginx-ingress
      imagePullSecrets:
      - name: regcred
      automountServiceAccountToken: true
      securityContext:
        seccompProfile:
          type: RuntimeDefault
#      volumes:
#      - name: nginx-etc
#        emptyDir: {}
#      - name: nginx-cache
#        emptyDir: {}
#      - name: nginx-lib
#        emptyDir: {}
#      - name: nginx-log
#        emptyDir: {}
      containers:
      - image: private-registry.nginx.com/nginx-ic-nap/nginx-plus-ingress:3.3.1
        imagePullPolicy: IfNotPresent
        name: nginx-plus-ingress
        ports:
        - name: http
          containerPort: 80
        - name: https
          containerPort: 443
        - name: readiness-port
          containerPort: 8081
        - name: prometheus
          containerPort: 9113
        - name: service-insight
          containerPort: 9114
        readinessProbe:
          httpGet:
            path: /nginx-ready
            port: readiness-port
          periodSeconds: 1
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "1"
            memory: "1Gi"
        securityContext:
          allowPrivilegeEscalation: false
#          readOnlyRootFilesystem: true
          runAsUser: 101 #nginx
          runAsNonRoot: true
          capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE
#        volumeMounts:
#        - mountPath: /etc/nginx
#          name: nginx-etc
#        - mountPath: /var/cache/nginx
#          name: nginx-cache
#        - mountPath: /var/lib/nginx
#          name: nginx-lib
#        - mountPath: /var/log/nginx
#          name: nginx-log
        env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        args:
          - -nginx-plus
          - -nginx-configmaps=$(POD_NAMESPACE)/nginx-config
         #- -default-server-tls-secret=$(POD_NAMESPACE)/default-server-secret
         #- -include-year
         #- -enable-cert-manager
         #- -enable-external-dns
          - -enable-app-protect
         #- -enable-app-protect-dos
         #- -v=3 # Enables extensive logging. Useful for troubleshooting.
         #- -report-ingress-status
         #- -external-service=nginx-ingress
          - -enable-prometheus-metrics
         #- -enable-service-insight
         #- -global-configuration=$(POD_NAMESPACE)/nginx-configuration
#      initContainers:
#      - image: nginx/nginx-ingress:3.3.1
#        imagePullPolicy: IfNotPresent
#        name: init-nginx-ingress
#        command: ['cp', '-vdR', '/etc/nginx/.', '/mnt/etc']
#        securityContext:
#          allowPrivilegeEscalation: false
#          readOnlyRootFilesystem: true
#          runAsUser: 101 #nginx
#          runAsNonRoot: true
#          capabilities:
#            drop:
#            - ALL
#        volumeMounts:
#        - mountPath: /mnt/etc
#          name: nginx-etc
```
Deploy NGINX+ Ingress Controller with App Protect and Prometheus metrics

kubectl apply -f deployment/nginx-plus-ingress.yaml



## deploy NLK
mkdir /home/user01/nlk

cd /home/user01/nlk

git clone https://github.com/nginxinc/nginx-loadbalancer-kubernetes.git

You will need to modify the following configmap file to update with the IP address of your external NGINX LB.

cd nginx-loadbalancer-kubernetes

nano deployments/deployment/configmap.yaml

``` yaml                                                                        
apiVersion: v1
kind: ConfigMap
data:
  nginx-hosts:
    "http://10.1.10.5:9000/api"
metadata:
  name: nlk-config
  namespace: nlk

```
kubectl apply -f deployments/deployment/namespace.yaml

./deployments/rbac/apply.sh

kubectl apply -f deployments/deployment/configmap.yaml

kubectl apply -f deployments/deployment/deployment.yaml

You will then need to update the following file.  The ExternalIP section will need to be updated in order to match your IP of external NGINX LB

nano docs/tcp/loadbalancer-nlk.yaml

``` yaml
# NLK LoadBalancer Service file
# Spec -ports name must be in the format of
# nlk-<upstream-block-name>
# The nginxinc.io Annotation must be added
# Chris Akker, Apr 2023
#
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress
  namespace: nginx-ingress
  annotations:
    nginxinc.io/nlk-nginx-lb-http: "stream"    # Must be added
    nginxinc.io/nlk-nginx-lb-https: "stream"   # Must be added
spec:
  type: LoadBalancer
  externalIPs:
  - 10.1.10.5
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: nlk-nginx-lb-http      # Must be changed
  - port: 443
    targetPort: 443
    protocol: TCP
    name: nlk-nginx-lb-https     # Must be changed
  selector:
    app: nginx-ingress
```

kubectl apply -f docs/tcp/loadbalancer-nlk.yaml


You will then need to get the PORTS for HTTP stream and HTTPS Stream

kubectl get svc nginx-ingress -n nginx-ingress


## troubleshooting NLK
In the environment, the NLK is currently not working.  Checking the logs below, there's an error message indicating that NLK cannot reach external N+ LB.  Will revisit in the future.

```
kubectl -n nlk get pods | grep deployment | cut -f1 -d" "  | xargs kubectl logs -n nlk --follow $1
```

From the microK8s host, I ran the following commands to get external LB working:

To confirm connectivitity:
```
curl -X GET -s 'http://10.1.10.5:9000/api/9/stream/upstreams/'
```
I order to get the external LB working, utilize the N+ API to update stream servers. Be sure to match it with the ports identified in the nginx-ingress service type load balancer.

```
curl -X POST -d '{"server": "10.1.10.6:30564"}' -s 'http://10.1.10.5:9000/api/9/stream/upstreams/nginx-lb-http/servers'
```
```
curl -X POST -d '{"server": "10.1.10.6:31674"}' -s 'http://10.1.10.5:9000/api/9/stream/upstreams/nginx-lb-https/servers'
```



# Install Base infrastructure

mkdir /home/user01/base-infrastructure 

cd /home/user01/base-infrastructure

These manifiests from https://github.com/thnguyenf5/appworld-nginx-ingress-controller-api/tree/main/base-infrastructure

## install cafe app
kubectl apply -f cafe.yaml

kubectl apply -f cafe-vs.yaml

## install dadjokes api
kubectl apply -f dadjokes.yaml

kubectl apply -f dadjokes-vs.yaml

## install keycloak
kubectl apply -f keycloak.yaml

kubectl apply -f keycloak-vs.yaml

## configure monitoring persistant storage
kubectl apply -f monitoring.yaml
## create nginx prometheus service and service monitor
kubectl apply -f nginx-ingress-metrics-prometheus.yaml
## install kube prometheus stack

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm repo update

```
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring --set=alertmanager.persistentVolume.existingClaim=kube-prometheus-stack-pvc,server.persistentVolume.existingClaim=kube-prometheus-stack-pvc,grafana.persistentVolume.existingClaim=kube-prometheus-stack-pvc,serviceMonitorSelectorNilUsesHelmValues=false,podMonitorSelectorNilUsesHelmValues=false
```

kubectl apply -f prometheus-vs.yaml 

kubectl apply -f grafana-vs.yaml


kubectl exec --namespace monitoring -it kube-prometheus-stack-grafana-855dcd954b-8ft57 grafana-cli admin reset-admin-password appworld





## crapi
helm install --create-namespace --namespace crapi crapi . --values values.yaml


## misc commands
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout dev.local.key -out dev.local.crt

kubectl exec -it <pod> -n <namepace> -- /bin/sh
