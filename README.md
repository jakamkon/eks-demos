Sign in for free into 
https://killercoda.com/playgrounds/course/kubernetes-playgrounds/two-node
to get free 1h sessions of kubernetes env to run the following examples.

Url shortcut for the repo:
## https://t.ly/eks-demos
(doesn't work with clone repo)

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
configuration

    kubectl replace --force -f 1_simple-pod-configmap-env-v2.yaml
    kubectl exec pod-configmap-env -- env | grep -i SPECIAL

### 20. Or use a Deployment to manage our pods
    kubectl apply -f 1_deployment-configmap.yaml
    kubectl get deployment 
    kubectl get pods -l app=webapp-d

Check newly created env var from ConfigMap
    kubectl exec webapp-deployment-UUID -- env | grep -i special

### You can simply do a new deployment and new old pods will be
### replaces with a pods with a new configuration
    kubectl apply -f 1_deployment-configmap-v2.yaml
    kubectl get pods -l app=webapp-d

This time the UUID will be different since we've got a few totally new pods
    kubectl exec webapp-deployment-UUID -- env | grep -i special








