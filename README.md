Url shortcut for this repo:
## t.ly/eks-demos
(doesn't work with clone repo)

Sign in for free into 
https://killercoda.com/playgrounds/course/kubernetes-playgrounds/two-node
to get free 1h sessions of kubernetes env to run the following examples.

Once you're in the console, clone this repository:
git clone https://github.com/jakamkon/eks-demos
cd eks-demos

## Module 1:
### 1. Create some pods
    kubectl apply -f 1_simple-pod.yaml -f 1_simple-pod2.yaml

### 2. Check if pods are running
    kubectl get pods

## How do we access a pod?

### 3. Get into the first pod
    kubectl exec webapp1 -ti -- /bin/bash

### 4. Inside the container, access running server:
    curl localhost

### 5. Forward Pod port to your local port local-port:pod-port
    kubectl port-forward  --address 0.0.0.0 webapp1 8080:80
### 6. Click on top right on triple bar and choose Traffic/Ports then Host1 port 8080

## How do you access all pods?
### 7. Create a service that will be a single endpoint for pods
    kubectl apply -f 1_simple-service.yaml
    kubectl get svc service -o wide

### 8. You can test the new service from inside one of the pods
    kubectl exec webapp1 -ti -- /bin/bash
    for i in $(seq 10); do curl service;done

### 9.Forward service port to a localport
    kubectl port-forward service/service 8080:80
### 10. Click on top right on triple bar and choose Traffic/Ports then Host1 port 8080

### 11. Service will create an endpoint for each pod
    kubectl get endpoints service -o yaml

### 12. Where are those pods running?
    kubectl get nodes
    kubectl describe nodes

### 13. Check which node is used to run this pod
    kubectl describe pods webapp1

### 14. Check control plane and data plane
    kubectl get pods -n kube-system

### 15. Check available contexts and current context
### Context is a combination of cluster, namespace and user
    kubectl config get-contexts
    kubectl config current-context
    kubectl get pods

### 16. You can switch to a different context if you have one
    kubectl config set current-context new-context
    kubectl config current-context
    kubectl get pods

### And then switch back
    kubectl config set current-context default-context
    kubectl get pod

### 17. Create a new Namespace and run a pod in it
    kubectl create namespace new-ns
    kubectl apply -n new-ns -f 1_simple-pod.yaml
    kubectl get pods -n new-ns 

### 18. Using ConfigMaps.
    kubectl apply -f 1_simple-pod-configmap-env.yaml

### 19. But, when we try to update it with a different config...it fails
    kubectl apply -f 1_simple-pod-configmap-env-v2.yaml
    kubectl exec pod-configmap-env -- env | grep -i SPECIAL

There are some parts of a pod that we cannot change once it's been created, in our case it's `spec.containers[*].image`

We have to delete the current pod and create a new one with a new 
configuration:

    kubectl replace --force -f 1_simple-pod-configmap-env-v2.yaml
    kubectl exec pod-configmap-env -- env | grep -i SPECIAL

### 20. Or use a Deployment to manage our pods
    kubectl apply -f 1_deployment-configmap.yaml
    kubectl get deployment 
    kubectl get pods -l app=webapp-d

Check newly created env var from ConfigMap:

    kubectl exec webapp-deployment-UUID -- env | grep -i special

### You can simply do a new deployment and new old pods will be
### replaced with pods with a new configuration
    kubectl apply -f 1_deployment-configmap-v2.yaml
    kubectl get pods -l app=webapp-d

This time the UUID will be different since we've got a few totally new pods:

    kubectl exec webapp-deployment-UUID -- env | grep -i special

### Now you scale in and out your pods
    kubectl scale --replicas=10 deployment/webapp-deployment
    kubectl get deployment
    kubectl get pods
    kubectl scale --replicas=1 deployment/webapp-deployment
    kubectl get deployment
    kubectl get pods

### To access all of the pods you also need to create a service
### for this deployment
    kubectl scale --replicas=5 deployment/webapp-deployment
    kubectl apply -f 1_service-for-deployment.yaml
    kubectl get svc

You can also test out the service with all those new pods:

    kubectl get pods -l app=webapp-d
    kubectl exec webapp-deployment-UUID -ti -- /bin/bash
    for i in $(seq 10); do curl webapp-d-service;done

## Module 2,3
### 1. Create a custom EKS cluster with one managed group and fargate profile.
We will run some pods on both and see how each of them will scale.

Note: This won't work in killacoda sandbox environment from previous module since
you've already have a cluster.

    eksctl create cluster -f 2_custom-cluster-fargate.yaml
    eksctl get cluster --name=small-fargate

### 2. Let's check what nodegroups we have:

    eksctl get nodegroup --cluster=small-fargate

Before we use kubectl, let's check if our new cluster (small-fargate) is the current one (after using eksctl create cluster usually it is):
    kubectl config current-context

If not, let's switch:

    kubectl config get-contexts
    kubectl config set current-context small-farage-context
    kubectl config current-context

and how many nodes we have:

    kubectl get nodes

### 3. Create two namespaces, one for a managed node group and one for fargate:
    kubectl create namespace fargate-ns
    kubectl create namespace managed-nodes

### 4. Run some pods in each:
    kubectl apply -f https://k8s.io/examples/pods/simple-pod.yaml --namespace=fargate-ns 
    kubectl apply -f https://k8s.io/examples/pods/simple-pod.yaml \ 
								--namespace=managed-nodes
    kubectl get pods -A

### 5. Fargate won't work - namespace is fine, but also needs a label:
    kubectl describe pods nginx -n fargate-ns
    kubectl apply -f 2_fargate-pod.yaml --namespace=fargate-ns
    kubectl describe pods webapp1 -n fargate-ns

### 6. Since we don't have any nodes in a managed group no pods will be created there:
    kubectl describe pods --namespace=managed-nodes

But Fargate namespace will scale automaticly:

    kubectl get pods --namespace=fargate-ns

### 6. To run some pods on a managed group we need to excplicilty scale it:
m - min, M - max, N - nodes (desired number of nodes)

    eksctl scale nodegroup --name=ng1 --cluster=small-fargate -m=1 -M=1 -N=1
### 7. Let's check if we have scaled:
    kubectl get nodes
    kubectl get pods --namespace=managed-nodes

## Module 4
### Using ECR
### Build an image:
    docker build  --platform="linux/amd64" -t hello-world .
    docker images
### Create ECR repo
    aws ecr create-repository --repository-name demo-repo
    aws ecr describe-repositories | grep repositoryName | grep demo-repo
### Login to ECR
    aws ecr get-login-password --region eu-central-1 | docker login --username AWS --password-stdin 848718099117.dkr.ecr.eu-central-1.amazonaws.com
### Tag our image for ECR
    docker tag hello-world:latest 848718099117.dkr.ecr.eu-central-1.amazonaws.com/demo-repo
### Push our image to our ECR repository
    docker push 848718099117.dkr.ecr.eu-central-1.amazonaws.com/demo-repo
### Check if our image is there:
    aws ecr list-images --repository-name demo-repo
### Check for scan findings:
    aws ecr describe-image-scan-findings --repository-name demo-repo --image-id imageTag=latest
### Deploy to cluster (our nodes need to have permissions to read ECR repo):
    kubectl apply -f 4_pod-using-ecr.yaml























