Sign in for free into 
https://killercoda.com/playgrounds/course/kubernetes-playgrounds/two-node
to get free 1h sessions of kubernetes env to run the following examples.

Once you're in the console, clone this repository:
git clone https://github.com/jakamkon/eks-demos.git
cd eks-demos

## Module 1:
### Create some pods
    kubectl apply -f 1_simple-pod.yaml -f 1_simple-pod2.yaml

### Check if pods are running
    kubectl get pods

### Get into the first pod
    kubectl exec webapp1 -ti -- /bin/bash

### Forward Pod port to your local port local-port:pod-port
    kubectl port-forward webapp1 8080:80