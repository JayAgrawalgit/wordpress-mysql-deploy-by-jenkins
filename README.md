# WordPress MySQL deployed by Jenkins
Hello, Here we are using `Ubuntu 20.04.3 LTS` OS as a base machine.  

## Overview
* Write Jenkinsfile for jenkins pipeline.
* Install Jenkins in minikube with the help of helm repo.
* Deploy WordPress and MySQL charts in minikube through Jenkins Pipeline.

## Prerequisites
* Install Kubernetes
* Install Helm

## Process
We are doing this deployment in a different namespace called `demo`.   
For creating a new namespace you can use this command.
```bash
kubectl create namespace demo
``` 
> For all below steps we are using the same namespace

### Step 1: Writing Jenkinsfile for jenkins pipeline
1. Adding own jenkins slave image as an agent
> this agent is common for all Jenkinsfiles

```Jenkinsfile
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: shell
    image: jayagrawaldocker/jenkins:0.4
    command:
    - sleep
    args:
    - infinity
'''
            defaultContainer 'shell'
        }
    }
```

here I am adding `jayagrawaldocker/jenkins:0.4` as a slave image. In this image I am taking  `jenkins/inbound-agent:latest` as a base image and `Install helm & kubernetes` is that base image. You can also create your own image with below Dockerfile or continue with mine.

> Dockerfile
```Dockerfile
ARG version=latest
FROM jenkins/inbound-agent:$version
LABEL maintainer='Jay Agrawal'

ARG HELM_VERSION=3.5.2
ENV MAVEN_VERSION=3.5.4

USER root
RUN apt-get update && apt-get install curl

# install helm
RUN mkdir /tmp/helm \
RUN cd /tmp/helm  \
RUN curl -s https://get.helm.sh/helm-v${HELM_VERSION}-linux-amd64.tar.gz  | tar zxvf - \
RUN cp /tmp/helm/linux-amd64/helm /usr/local/bin/helm  \
RUN chmod +x /usr/local/bin/helm  \
RUN rm -rf /tmp/helm

# install Kubernetes CLI
# See https://storage.googleapis.com/kubernetes-release/release/stable.txt
RUN curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.19.0/bin/linux/amd64/kubectl \
  && chmod +x ./kubectl \
  && mv ./kubectl /usr/local/bin/kubectl
USER jenkins
```

### Step 2: Install Jenkins in minikube Jenkins 
1. You can refer to [official documents for Jenkins installation](https://github.com/jenkinsci/helm-charts/blob/main/charts/jenkins/README.md)
and skip `step 2: Install Jenkins in minikube Jenkins` or you can continue with us.
2. Get Repo Info
```bash
helm repo add jenkins https://charts.jenkins.io
helm repo update
```
3. Install Chart
```bash
# Helm 3
helm install -n demo helm-jenkins2 jenkins/jenkins
```
4. Wait for the container to start  
```bash
kubectl get pods -namespace demo
```
> Initial output
```output
NAME              READY   STATUS    RESTARTS       AGE
helm-jenkins-0    0/2     Running   0               1m
```
It will take 5-10 mins according to your machine Configuration, repeat the 4th command, until the addition of all containers in the pool.  

>Final output
```final output
NAME              READY   STATUS    RESTARTS       AGE
helm-jenkins-0    2/2     Running   0              10m
```
5. Generate a password for admin user
```bash
kubectl exec --namespace demo -it svc/helm-jenkins -c jenkins -- /bin/cat /run/secrets/chart-admin-password && echo
```
6. For creating the link between minikube and local machine.
```bash
kubectl --namespace demo port-forward svc/helm-jenkins 8080:8080
```
7. Go to the link [http://127.0.0.1:8080](http://127.0.0.1:8080) and log in with the username -> `admin` and password generated by the second-last command. (5th point)

### Step 3: Creating a new job in Jenkins
#### For running MySQL and WordPress in a single pipeline
1. First go to the new item and create a new pipeline job with any name. (recommended name `demo`)  
2. Then go in the last in the pipeline section there in Definition select `Pipeline script for SCM`.  
3. In SCM select `git` and in `Repository URL` past [https://github.com/JayAgrawalgit/wordpress-mysql-deploy-by-jenkins.git](https://github.com/JayAgrawalgit/wordpress-mysql-deploy-by-jenkins.git) or HTTPS link of this repo which is present in the top green code button.  
4. In `Branches to build` enter `*/main`.  
5. In `Script Path` enter `Jenkinsfile`.  
6. `Apply` and `Save`.  
7. Now Build the Project

#### For running MySQL and WordPress in 2 different pipelines.  
1. First go to the new item and create a new pipeline job with any name. (recommended name `MySQL-pipeline`)  
2. Then go in the last in the pipeline section there in Definition select `Pipeline script for SCM`.  
3. In SCM select `git` and in `Repository URL` past [https://github.com/JayAgrawalgit/wordpress-mysql-deploy-by-jenkins.git](https://github.com/JayAgrawalgit/wordpress-mysql-deploy-by-jenkins.git) or HTTPS link of this repo which is present in the top green code button.  
4. In `Branches to build` enter `*/main`.  
5. In `Script Path` enter `Jenkinsfile1` for MySQL.  
6. `Apply` and `Save`.  
> Note: As e need to perform individual job run for both MySQL and WordPress we need to take care of that especially 
11. For running the job individual In `Script Path` enter `Jenkinsfile2` for WordPress, rest of the procedure as above  
12. `Apply` and `Save`.  
13. Now Build the Project name `MySQL-pipeline` and if It will run `SUCCESS` then build the project with the name `WordPress-pipeline`.

### If you encountered the below error in running the pipeline: 
1. `failed to query with labels: secrets are forbidden: User "system:serviceaccount:demo:default" cannot list resource "secrets" in API group "" in the namespace "demo"`  
then run 
> Given command is `Strictly not recommended in production environment`, This is only a work around.  
```bash
kubectl create clusterrolebinding permissive-binding --clusterrole=cluster-admin --user=admin --user=kubelet --group=system:serviceaccounts:demo
``` 
For more information [kubernetes.io](http://kubernetes.io/) [Jenkins.io](https://jenkins.io/)

#### Keep Learning
