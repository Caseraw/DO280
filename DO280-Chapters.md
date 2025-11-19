# DO280

Course summary. OpenShift version 4.14.

# Chapters

- Chapter 1, Declarative Resource Management
- Chapter 2, Deploy Packaged Applications
- Chapter 3, Authentication and Authorization
- Chapter 4, Network Security
- Chapter 5, Expose non-HTTP/SNI Applications
- Chapter 6, Enable Developer Self-Service
- Chapter 7, Manage Kubernetes Operators
- Chapter 8, Application Security
- Chapter 9, OpenShift Updates

## Chapter 1, Declarative Resource Management

### Imperative

```shell
[user@host ~]$ kubectl create deployment db-pod --port 3306 \
  --image registry.ocp4.example.com:8443/rhel8/mysql-80


[user@host ~]$ kubectl set env deployment/db-pod \
  MYSQL_USER='user1' \
  MYSQL_PASSWORD='mypa55w0rd' \
  MYSQL_DATABASE='items'
```

### Declarative

```shell
[user@host ~]$ oc create deployment hello-openshift -o yaml \
  --image registry.ocp4.example.com:8443/redhattraining/hello-world-nginx:v1.0 \
  --save-config \
  --dry-run=client \
  > ~/my-app/example-deployment.yaml

[user@host ~]$ tree my-app
my-app
├── example_deployment.yaml
└── service
    └── example_service.yaml

[user@host ~]$ oc create -R -f ~/my-app
deployment.apps/hello-openshift created
service/hello-openshift created
```

#### Update

```shell
[user@host ~]$ oc apply -f ~/my-app/example-deployment.yaml \
  --dry-run=server --validate=true
```

#### Validate

```shell
[user@host ~]$ oc apply -f ~/my-app/example-deployment.yaml \
  --dry-run=server --validate=true
```

#### Compare

```shell
[user@host ~]$ oc diff -f example-deployment.yaml
```

#### Restart

```shell
[user@host ~]$ oc rollout restart deployment deployment-name
```

#### Patch

```shell
[user@host ~]$ oc patch deployment hello -p \
  '{"spec":{"template":{"spec":{"containers":[{"name": \
  "hello-rhel7","resources": {"requests": {"cpu": "100m"}}}]}}}}'

[user@host ~]$ oc patch deployment hello --patch-file ~/volume-mount.yaml
```

#### Kustomize

- https://kustomize.io/
- https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/

```shell
oc apply -k base
oc apply -k overlays/development
oc delete -k overlays/development
```

### Some commands

```shell
oc login -u <USER> -p <PASSWORD> https://api.<cluster>.example.com:6443

oc new-project <PROJECT NAME>

git clone https://git.example.com/repo/project.git
git checkout v1.0
git clone https://git.example.com/repo/project.git --branch v1.1.0
git log --oneline

oc diff -f .
oc apply -f .

oc get pods -w
oc get all
oc get routes -l app=<APP LABEL>
watch oc get deployments,pods
oc rollout restart deployment/<DEPLOYMENT NAME>
```

## Chapter 2, Deploy Packaged Applications

### Templates

```shell
oc get templates -n openshift
oc describe template cache-service -n openshift
oc get template cache-service -o yaml -n openshift

oc process --parameters mysql-persistent -n openshift
oc new-app --template=mysql-persistent \
  -p MYSQL_USER=user1 \
  -p MYSQL_PASSWORD=mypasswd

oc process roster-template \
  -p MYSQL_USER=user1 -p MYSQL_PASSWORD=mypasswd -p INIT_DB=true | oc apply -f -

oc process roster-template \
  --param-file=roster-parameters.env | oc diff -f -

oc process roster-template \
  --param-file=roster-parameters.env | oc apply -f -
```

### HELM charts

```shell
helm repo add openshift-helm-charts https://charts.openshift.io/
helm repo list
helm search repo
helm search repo --versions

helm show chart chart-reference
helm show values chart-reference
helm show values do280-repo/etherpad --version 0.0.6

helm install release-name chart-reference --dry-run --values values.yaml
helm install example-app do280-repo/etherpad -f values.yaml --version 0.0.6
helm upgrade example-app do280-repo/etherpad -f values.yaml --version 0.0.7

helm list
helm history release_name
helm rollback release_name revision
```

## Chapter 3, Authentication and Authorization

### Passwd

```shell
htpasswd -c -B -b /tmp/htpasswd student redhat123
htpasswd -b /tmp/htpasswd student redhat1234
htpasswd -D /tmp/htpasswd student

oc create secret generic htpasswd-secret \
  --from-file htpasswd=/tmp/htpasswd -n openshift-config

oc extract secret/htpasswd-secret -n openshift-config --to -

oc extract secret/htpasswd-secret -n openshift-config --to /tmp/ --confirm

oc set data secret/htpasswd-secret \
  --from-file htpasswd=/tmp/htpasswd -n openshift-config
```

### OAuth

```shell
oc get oauth cluster -o yaml > oauth.yaml
oc replace -f oauth.yaml

watch oc get pods -n openshift-authentication
```

```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - htpasswd:
      fileData:
        name: htpasswd-secret
    mappingMethod: claim
    name: my-local-users
    type: HTPasswd
```

### Users

```
oc delete user manager
oc get identities
oc delete identity my_htpasswd_provider:manager
```

### RBAC

```shell
oc adm policy add-cluster-role-to-user cluster-role username
oc adm policy add-cluster-role-to-user cluster-admin student
oc adm policy remove-cluster-role-from-user cluster-role username
oc adm policy who-can delete user

oc policy add-role-to-user role-name username -n project
oc policy add-role-to-user admin dev -n wordpress
```

### Self provisioner

```
oc describe clusterrolebindings self-provisioners
oc adm policy remove-cluster-role-from-group self-provisioner system:authenticated:oauth

oc adm policy add-cluster-role-to-group self-provisioner managers
```

### Groups

```shell
oc adm groups new lead-developers
oc adm groups add-users lead-developers user1
```

## Chapter 4, Network Security

### Routes

```shell
oc expose svc todo-http --hostname todo-http.apps.ocp4.example.com
oc get routes

oc create route edge todo-https \
    --service todo-http \
    --hostname todo-https.apps.ocp4.example.com
```

### Certs

```shell
openssl genrsa -out training.key 4096

openssl req -new \
    -key training.key -out training.csr \
    -subj "/C=US/ST=North Carolina/L=Raleigh/O=Red Hat/\
    CN=todo-https.apps.ocp4.example.com"

openssl x509 -req -in training.csr \
    -passin file:passphrase.txt \
    -CA training-CA.pem -CAkey training-CA.key -CAcreateserial \
    -out training.crt -days 1825 -sha256 -extfile training.ext

oc create secret tls todo-certs \
    --cert certs/training.crt --key certs/training.key

oc create route passthrough todo-https \
    --service todo-https --port 8443 \
    --hostname todo-https.apps.ocp4.example.com
```

### NetworkPolicies

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: deny-all
  namespace: my-namespace
spec:
  podSelector: {}
---
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: allow-specific
  namespace: my-namespace
spec:
  podSelector:
    matchLabels:
      deployment: hello
  ingress:
    - from:
      - namespaceSelector:
          matchLabels:
            network: different-namespace
        podSelector:
          matchLabels:
            deployment: sample-app
      ports:
      - port: 8080
        protocol: TCP
---
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: allow-from-openshift-ingress
  namespace: my-namespace
spec:
  podSelector:
    matchLabels:
      deployment: 'hello'
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          policy-group.network.openshift.io/ingress: ""
```

```shell
oc annotate service server \
    service.beta.openshift.io/serving-cert-secret-name=server-secret

oc get secret server-secret
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: server
  namespace: my-namespace
  annotations:
    service.beta.openshift.io/serving-cert-secret-name: server-secret
...
...
...
```

```shell
oc create configmap ca-bundle
oc annotate configmap ca-bundle service.beta.openshift.io/inject-cabundle=true
```

## Chapter 5, Expose non-HTTP/SNI Applications
## Chapter 6, Enable Developer Self-Service
## Chapter 7, Manage Kubernetes Operators
## Chapter 8, Application Security
## Chapter 9, OpenShift Updates

