## Hosting React Applications on Microsoft Azure with GitHub Actions

![](img/poster.png)

[YouTube Demonstration](https://www.youtube.com/watch?v=73tq15DVeSE&feature=youtu.be)

### Create a repository and add secrets

#### Generate Microsoft Azure Secret

* Generate Secret

```
az account list
az ad sp create-for-rbac --name "myApp" --role contributor --scopes /subscriptions/<"">/resourceGroups/<""> --sdk-auth
```

* Store Secret

```
{
  "clientId": "",
  "clientSecret": "",
  "subscriptionId": "",
  "tenantId": "",
  "activeDirectoryEndpointUrl": "",
  "resourceManagerEndpointUrl": "",
  "activeDirectoryGraphResourceId": "",
  "sqlManagementEndpointUrl": "",
  "galleryEndpointUrl": "",
  "managementEndpointUrl": ""
}
```

### Create a React Application

```
$ npx create-react-app demo
$ cd demo
$ npm run start
```

### Push the React Application on GitHub

```
$ git remote add origin "<URL>"
$ git add .
$ git commit -m "<Commit Message>"
$ git push -u origin master
```

### Dockerize the React Application

```Dockerfile
FROM    node:10-alpine 
WORKDIR /usr/src/app
COPY    . .
RUN     npm install
RUN     npm run build
EXPOSE  3000
CMD     [ "npm", "start" ]
```

### Setup GitHub Actions for Docker Build and Push

```yaml
name: Deploy React Applications to AKS using GitHub Actions
on:
  push
  
jobs:
  push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Create Docker Image for Development Environment
        run: docker build -t [username/repo] .

      - name: Login to DockerHub
        uses: docker/login-action@v1 
        with:
          username: ${{ secrets.username }}
          password: ${{ secrets.password }}
        
      - name: Docker Push
        run: docker push [username/repo]
```

https://labs.play-with-docker.com/#

### Create a Kubernetes Service on Azure

* Setup an Ingress and Map to FQDN

```
helm repo add stable https://charts.helm.sh/stable --force-update
helm install ingress stable/nginx-ingress
kubectl get service
```

#### Writing Kubernetes Deployment and Service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - name: testaks
        image: [username/repo]
        ports:
        - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: demo
spec:
  type: NodePort
  ports:
  - port: 3000
  selector:
    app: demo
```

#### Writing Kubernetes Ingress

```yaml
apiVersion : extensions/v1beta1
kind : Ingress
metadata :
  name : demo
  annotations :
    kubernetes.io/ingress.class : nginx
    nginx.ingress.kubernetes.io/rewrite-target : /
spec :
  rules :
  - host : <URL> 
    http :
      paths :
      - path : /
        backend :
          serviceName : demo
          servicePort : 3000
```

### Setting up GitHub Actions to Deploy React Docker Image on AKS

```yaml
      - uses: azure/aks-set-context@v1
        with:
          creds: '${{ secrets.AZURE }}'
          cluster-name: sib
          resource-group: jioexchange
        
      - uses: azure/k8s-deploy@v1
        with:
          manifests: |
            k8s.yaml
            ingress.yaml
          namespace: default
```
