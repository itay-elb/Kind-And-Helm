# Kind-And-Helm
What is kind and helm
kind and helm is two appliction that help you create local Kubernetes (k8s) and easy to manega. So, in this post is more like introdection to k8s and see who it work.

## Are steps _-

1.	build a python app with flask.
2.	install docker, build docker file and send it to dockerhub.
3.	install kind and helm for better flexibility and easier installison and modify the yaml file to match the app.

## 1: Build python app
First we need to install python (if you dont have).

```bash
$ sudo dnf install python
```

We create the directory that are project going to be set, so I call it hot-cold like the app we going to build.

```bash
$ mkdir hot-cold
$ cd hot-cold
$ python3 -m venv ./venv
$ source ./venv/bin/activate
```

The venv directory make a close env for the app becouse to give the app only the thing he need. make sure the venv is active, is suppose to be on the side write (venv)
Now for the app secsion we need to create directory for the app.

```bash
$ mkdir app
$ cd app
$ pip install flask
#Flask is python library for web applications.
$ vim app.py
```
```python
from flask import Flask, render_template, request, redirect, url_for
import random

app = Flask(__name__)

@app.route('/')
def home():
    return render_template('index.html')

@app.route('/get_temperature', methods=['POST'])
def get_temperature():
    temperature = random.choice(['hot', 'cold'])
    return render_template('result.html', temperature=temperature)

if __name__ == '__main__':
    app.run(debug=True)
```

Create template directory in app. Ther will be the html files.

```bash
$ mkdir templates
$ cd templates
$ vim index.html
```
```html
index.html

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Hot or Cold</title>
</head>
<body>
    <form action="/get_temperature" method="post">
        <button type="submit">Check Temperature</button>
    </form>
</body>
</html>
$ vim result.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Temperature Result</title>
</head>
<body>
    <p>The temperature is: {{ temperature }}</p>
    <a href="/">Check again</a>
</body>
</html>
```
## 2: The docker step
Install docker

```bash
$ sudo dnf -y install docker
$ sudo systemctl start docker
$ sudo systemctl enable docker
$ sudo usermod -a -G docker your-user
$ newgrp docker
```

Build requirements.txt file for the Dockerfile.

```bash
$ pip freeze > requirements.txt
Create the Dockerfile for build the image.
FROM python
WORKDIR /app
COPY . /app
RUN pip install -r requirements.txt
EXPOSE 8080
CMD ["python3","app.py"]
```
```bash
$ docker build -t hot-cold1 .
```

You can see the image in docker images.

```bash
$ docker images
```

For push an image to dockerhub need account it will be importent for the contuin this project so make sure you have an account.
To push the image need to tag him.

```bash
$ docker tag your-image your-user/your-repo:image
$ docker push your-user/your-repo:image
```

## 3. Kind And Helm
What is kind: is a tool for running local Kubernetes clusters using Docker container.
What is helm: helps you manage Kubernetes applications.
first we install kind.

```bash
$ [ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64
$ chmod 700 ./kind
$ sudo mv ./kind /usr/local/bin/kind
```

After we install kind we create a cluster for this we will make a custom configersion to the cluster. 

```bash
$ vim kind-config.yaml
```

Put the scrip inside the kind-config.yaml.

```yaml
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
nodes:
- role: control-plane
- role: worker
- role: worker
```

Pull image for to create the nodes.

```bash
$ docker pull kindest/node:v1.22.0
$ kind create cluster --config kind-config.yaml
```

## Add metallb
Metallb is a load balancer localy and it will help to get external ip to the website.

```bash
$ kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.3/config/manifests/metallb-native.yaml
$ vim metallb-config.yaml
```

This configerison file will choose the ip for the nodes by the pool you put their.

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.250   # an example of pull ip you need from your ip address 
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
```
Change the ip range to match your avilabole ip.

```bash
$ kubectl get nodes -o wide
```

It will show you internal ip and modify the metallb-config to your internal ip. for my is:

```yaml
# example where to put the ip
spec:
  addresses:
  - 172.18.255.240-172.18.255.250
```

And create the file for know the pool.

```bash
$ kubectl create -f metallb-config.yaml
```

Install helm

```bash
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
$ chmod  700 get_helm.sh
$ ./get_helm.sh
```

Or
```bash
$ sudo dnf install helm   # change the dnf to your installing pakege
```

Now we create are chart that will build the k8s for us.

```bash
$ helm create hot-cold-chart
```

We modify the files to match are app.
values.yaml

```yaml
# Default values for hot-cold.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 2

image:
  name: k8s
  repository: itay5858
  tag: hot-cold

namespace:
  name: hot-cold-app

service:
  type: ClusterIP
  targetport: 8080
  port: 80

ingress:
  enabled: true
  classname: "nginx"
  host: itay.octopus.lab
  path: /
  pathType: Prefix


autoscaling:
  enabled: true
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
```

deploymaent.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dep
  namespace: {{ .Values.namespace.name }}
spec:
  replicas:  {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: hot-cold
  template:
    metadata:
      labels:
        app: hot-cold
    spec:
      containers:
      - name: {{ .Values.image.name }}
        image: "{{ .Values.image.repository }}/{{ .Values.image.name }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: {{ .Values.service.targetport }}
```

service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ser
  namespace: {{ .Values.namespace.name }}
spec:
  selector:
    app: hot-cold
  ports:
    - protocol: TCP
      port:  {{ .Values.service.port }}
      targetPort:   {{ .Values.service.targetport }}
  type: {{ .Values.service.type }}
```

ingress.yaml

```yaml
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hot-cold-ingress
  namespace: {{ .Values.namespace.name }}
  annotations:
    kubernetes.io/ingress.class: {{ .Values.ingress.classname }}
spec:
  rules:
  - host: {{ .Values.ingress.host }}
    http:
      paths:
      - path: {{ .Values.ingress.path }}
        pathType: {{ .Values.ingress.pathType }}
        backend:
          service:
            name: ser
            port:
              number: {{ .Values.service.port }}
{{- end }}
```

namespace.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: {{ .Values.namespace.name }}
```

## Domain in your computer
For the computer will know the domain we put for the hot-cold app we need to add to /etc/hosts file the following command.

```bash
$ sudo vim /etc/hosts
#Add the adress you get for your ingress  #Your domain you put in the ingress
172.18.255.241                                                      itay.octopus.lab         #Example
$ sudo systemctl daemon-reload
```

## Nginx ingress controlar
Nginx ingress controlar is what give you the access to web with the domain you set. go back to the directory you have your app and hot-cold-chart.

```bash
$ cd ~/hot-cold
$ helm pull oci://ghcr.io/nginxinc/charts/nginx-ingress --untar --version 1.1.3
$ kubectl apply -f nginx-ingress/crds/
$ helm install nginx-ingress oci://ghcr.io/nginxinc/charts/nginx-ingress --version 1.1.3
```

The last thing we need to do is to install what k8s.

```bash
$ helm install hot-cold hot-cold-chart/
```

## Summary
We used several different technologies to create this project. python for the application, docker to create and save the image, kind for create a local k8s in the machine, helm for easier way to controll the k8s and his values, neginx-ingress-controller to allow access to the outside and metallb for load balancer.
