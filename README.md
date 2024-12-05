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
    kubectl port-forward --address 0.0.0.0 service/service 8080:80
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
    kubectl apply -f https://k8s.io/examples/pods/simple-pod.yaml --namespace=managed-nodes
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
## Module 5
### Full AutoScaling 
###     1. Scaling a number of nodes in the cluster with EKS Auto Mode (a new feature of EKS - 1.12.2024) - it's a fully managed Karpenter setup that manages nodes in your cluster
###     2. Scaling a number of pods with Horizontal Pod Autoscaling (HPA) using metrics-server (HPA is already part of Kubernetes)
###    
Note: This won't work in killacoda sandbox environment - metrics-server is not reachable once installed inside the cluster

### 1. To use AutoMode you need to enable it on the cluster or create a new one with AutoMode option set to true.
### This command will create a new cluster with AutoMode enabled and no nodes.
    eksctl create cluster -f custom_cluster_automode.yaml

### 2. Install metrics-server - HPA is using resource usage metrics from metrics-server to make descisions about how many pods do we need at any given time
    kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
    kubectl get deployment metrics-server -n kube-system

### Let's watch a few components, run each command in a separate terminal/window:
### Watch a number of nodes (you will see one node since we're running metrics-server):
    watch kubectl get nodes

### Watch a number of pods (you will see no pods):
    watch kubectl top pods

### Watch the CPU utilization of the deployment (you will see no CPU usage - cpu: 0%/50%):
    watch kubectl get hpa

### Let's create a deployment with HPA enabled:
    kubectl apply -f 5_deployment-with-hpa.yaml

### And simulate some heavy load:
    kubectl get pods
    kubectl exec webapp-deployment-XYZ -ti -- /bin/bash

### Inside one of the pods run:
    apt-get update -y && apt-get install stress -y
    stress --cpu 4 --timeout 60s

### - In a few seconds you should see a spike in CPU usage on hpa.
### - Then a number of pods will increase
### - Then you will see a new node being provisioned
### - Since we're simulating heavy load only for short period of time (120s) and shorten the time that pods should stick around you will see pod number go down after around two minutes and then number of nodes as well
### You can watch what's going on here:
    kubectl get events -w --sort-by '.lastTimestamp' 

## Module 6
### Testing pod-to-pod and ns-to-ns network access
Note: you can use 
https://killercoda.com/playgrounds/course/kubernetes-playgrounds/two-node
to run this demo.

## Clode the repo with examples:
git clone https://github.com/jakamkon/eks-demos

cd eks-demos

## Create a new namespace 
kubectl create namespace new-ns

## Create a pod in default namespace and new-ns namespace
kubectl apply -f 1_simple-pod.yaml

kubectl apply -f 1_simple-pod.yaml -n new-ns

kubectl get pods -A

## Access the first pod from the terminal
kubectl exec webapp1 -ti -- /bin/bash

## Get POD_1_IP
hostname -i

## Open a new tab/terminal and run the second pod
kubectl exec -n new-ns webapp1 -ti -- /bin/bash

## Get POD_2_IP
hostname -i

curl POD_1_IP

## Go back to the first window and access POD 2
curl POD_2_IP

## Even though two pods are in separate namespaces 
## the can all talk to each other

## Apply a network policy to those pods
kubectl apply -f 6_simple_network_policy.yml

## In both windows try curl:
## In POD 1 window:
curl POD_2_IP

## In POD 2 window:
curl POD_1_IP

## Network policy is allowing traffic only from default namespace
## Now in POD 2 window you cannot access POD 1 
## (POD 2 is in different namespace than default)






















