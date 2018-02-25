# jenkins-kubernetes

This project builds a war and deploys it into a Kubernetes Cluster using the [Pipeline Multibranch Plugin](https://wiki.jenkins.io/display/JENKINS/Pipeline+Multibranch+Plugin).

Deploys and updates a canary, production and development environment when pushing code through the Pipeline Multibranch job.

## Setup
Start a Jenkins instance using [Helm](https://github.com/kubernetes/helm):
```
helm install --name jenkins --values=jenkins-override-values.yml --namespace jenkins stable/jenkins
```

In your Kubernetes cluster create a namespace called "production":
```
kubectl create ns production
```

Create a Pipeline Multibranch job and link it to your repository.

Add your Docker Hub credentials to Jenkins via [Jenkins credentials](https://jenkins.io/doc/book/using/using-credentials/).
The credential ID must be `"docker-hub-credentials"` - this is for pushing newly built Docker images.

## Usage notes
If the repository has branches called "canary" or "master", Jenkins will deploy any newly pushed code into their respective
environments. if a branch is named any other, Jenkins will create a new namespace and environment for that branch.

You can apply resource limits for a namespace by applying the `dev_namespace_quota.yml` resource quota:
```
kubectl create -f dev/dev-namespace-quota--namespace=myspace
```
