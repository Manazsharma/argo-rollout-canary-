
# ArgoCD & Argo Rollouts Demo

This readme provides a comprehensive guide for setting up a GitOps pipeline using Argo CD for continuous deployment and implementing  a canary release with Argo Rollouts within a Kubernetes environment.







## Acknowledgements

 - [Argo CD Official Documentation](https://argo-cd.readthedocs.io/en/stable/)
 - [Argo Rollouts Official Documentation](https://argo-rollouts.readthedocs.io/en/stable/)



## Setup & Configuration

### Setting up a Kubernetes Enviornment for Argo CD 

   Contanerization Tool- Docker Desktop -[Setup Guide](https://docs.docker.com/desktop/)

Minikube Configuration and Status
```bash
minikube start
```
```bash
minikube status  
```
```
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

## What is Argo CD?
Argo CD is a declarative, GitOps continuous delivery tool for Kubernetes.
### Install Argo CD 

Here are the steps to install ArgoCD and retrieve the admin password:

1. #### Create Namespace for ArgoCD:
   ```bash
   kubectl create namespace argocd
   ```

2. #### Apply ArgoCD Manifests:
   ```bash
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.4.7/manifests/install.yaml
   ```
3. #### Argo CD services:
    ```bash
     kubectl get svc -n argocd
    ```
![Screenshot 2024-04-03 002453](https://github.com/Manazsharma/Cloud-native-app/assets/91602856/d0d4e8f1-0e83-4848-854a-19d19e810694)

4. #### Patch Service Type to NodePort: 
   *challenge* : As the argocd-server type is 'Cluster IP' by default so we change it to 'NodePort' to access the service from Terminal/Browser.
   ```bash
   kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
   ```

5. #### Retrieve Admin Password:
   ```bash
   kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
   ```

6. #### Starting Tunnel for service argocd-server:
    ```bash
     minikube service argocd-server -n argocd
    ```
![Screenshot 2024-04-01 234114](https://github.com/Manazsharma/Cloud-native-app/assets/91602856/328a0c3c-8833-40f7-b2f0-e4cb0c74aac0)


#### Login to Argo CD Web UI by redirecting to argocd-server URL

![argocd-ui](https://github.com/Manazsharma/Cloud-native-app/assets/91602856/468a51f1-f98f-4c56-be45-2fb79233177e)

Check live demo [here](https://cd.apps.argoproj.io/applications?showFavorites=false&proj=&sync=&autoSync=&health=&namespace=&cluster=&labels=)


## What is Argo Rollouts?
Progressive Delivery for Kubernetes

### Install Argo Rollouts
 ```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

#### Kubectl plugin Installation to monitor the deployment of the new version, ensuring the canary release successfully completes.

For Installation guide click [here](https://argoproj.github.io/argo-rollouts/installation/)


### Canary Deployment Strategy
A canary rollout is a deployment strategy where the operator releases a new version of their application to a small percentage of the production traffic.

Users can define a list of steps the controller uses to manipulate the ReplicaSets when there is a change to the .spec.template. Each step will be evaluated before the new ReplicaSet is promoted to the stable version, and the old version is completely scaled down.

By using the setWeight and the pause fields, a user can declaratively describe how they want to progress to the new version. Below is an example of a canary strategy.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: example-rollout
spec:
  replicas: 10
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.15.4
        ports:
        - containerPort: 80
  minReadySeconds: 30
  revisionHistoryLimit: 3
  strategy:
    canary: #Indicates that the rollout should use the Canary strategy
      maxSurge: "25%"
      maxUnavailable: 0
      steps:
      - setWeight: 10
      - pause:
          duration: 1h # 1 hour
      - setWeight: 20
      - pause: {} # pause indefinitely
```


## Gitops Pipeline 

![architechture](https://github.com/Manazsharma/Cloud-native-app/assets/91602856/cd842a37-5575-4f5c-b5ba-805e027d33fa)




### GitOps Principles and Argo CD Implementation

- Declarative Configuration
- Version Control
- Automation
- Continuous Delivery
- Observability - Rollouts


### Argo CD - *Application Controller*

#### Create An Argo CD Application From A Git Repository

1. After logging in, click the + New App button as shown below
![new-app](https://github.com/Manazsharma/Cloud-native-app/assets/91602856/37a45ca6-12ef-409d-a8a1-d8b46afbe0de)


2. Give your app the name **rollout-canary** use the project default, and leave the sync policy as Manual:

3. Connect the *https://github.com/Manazsharma/argo-rollout-canary-.git* repo to Argo CD by setting repository url to the github repo url, leave revision as HEAD, and set the path to **rollout**:

4. For Destination, set cluster URL to https://kubernetes.default.svc and namespace to default:

5. After filling out the information above, click Create at the top of the UI to create the **rollout-canary** application:

The application status is initially in OutOfSync.To sync (deploy) the application, run:
```bash
 argocd app sync rollout-canary
```
 
### Argo CD to monitor my repository and automatically deploy changes to your Kubernetes cluster.

![Rollout canaray pipeline](https://github.com/Manazsharma/argo-rollout-canary-/assets/91602856/f244e9a5-dae0-450e-a9bd-90cf12161bfd)


#### Listing rollouts in the default namespace
```bash
  kubectl argo rollouts list rollouts
```
![list rollout1](https://github.com/Manazsharma/argo-rollout-canary-/assets/91602856/e43e550f-6d3c-4278-8a74-19b3f2e0ff3b)



#### Rollout description 
 ```bash
  kubectl argo rollouts get rollouts rollout-canary
```

![Rollout description](https://github.com/Manazsharma/argo-rollout-canary-/assets/91602856/50d7784e-e6f4-4b5b-b6b8-1926149d57ba)





### Rollout Strategy [Canary]
 
 - Rollout Manifest files [ro.yaml]()
   
 ```yaml
  strategy:
    canary:
      steps:
      - setWeight: 20
      # The following pause step will pause the rollout indefinitely until manually resumed.
      - pause: {}
      
  ```
1. **setWeight: 20**: This step indicates that initially, only 20% of the traffic or load will be directed to the canary release (i.e., the new version). The remaining 80% will continue to use the stable release. This allows for a gradual introduction of the new version, enabling observation and testing of its performance in a controlled manner.

2. **pause: {}**: This step instructs the deployment process to pause after the initial 20% traffic diversion to the canary release. It effectively stops the rollout process indefinitely until manually resumed. Pausing the rollout at this stage allows operators or developers to inspect the canary release, monitor its behavior, gather metrics, and perform any necessary tests or evaluations before proceeding with further rollout.
  



### Triggering the Rollout

- version change of docker image  
```yaml
 spec:
      containers:
      - name: minikube
        image: nginx:1.24.0-alpine-sl
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
  ``` 

  ```bash
    git push origin main
  ``` 
### Monitoring the Rollout

```bash
  kubectl argo rollouts get rollouts rollout-canary
```

![rollout monitor 2](https://github.com/Manazsharma/argo-rollout-canary-/assets/91602856/613f8fe9-a651-4937-a79c-54c4157e1c19)

### Canary release successfully implemented

As we can monitor that after updateing docker image newer version then the deployment of newer version, as directed by the canary startegy 20% of the traffic that is 2 ReplicaSets out of 10 are of the new verson while rest 8 ReplicaSets still continue to use the stable release.


 -  status: Paused
 -  The rollout will be paused indefinitely. To unpause, use the     argo kubectl plugin promote command.


  ```bash
    kubectl argo rollouts promote rollout-canary
```

![promote 2](https://github.com/Manazsharma/argo-rollout-canary-/assets/91602856/a0d3c9b2-6199-40c2-a4f9-6bbeb4f50c63)






  ```bash
    kubectl argo rollouts get rollout1 rollout-canary
```

![revision scaled down](https://github.com/Manazsharma/argo-rollout-canary-/assets/91602856/4a6606b8-78af-4900-a96e-a1ca4718abc9)






##  Clean up 

To cleanly remove all resources created during this project from the Kubernetes cluster, follow these steps:

1. Delete the ArgoCD application:
  
 ```bash
    argocd app delete rollout-canary
```
2. Wait for all resources to be removed:
```bash
    watch argocd app get rollout-canary
```
3. Delete the ArgoCD namespace:
```bash
    watch argocd app get rollout-canary
```
4. Delete rollout resources:
```bash
    kubectl delete rollout rollout-canary
```
5. Delete deployment and services
```bash
  kubectl delete deployment argocd
  kubectl delete service rollout-svc
```




## Community Blogs & Presentations

1. [Argo CD : Applying Gitops Principles to manage Production Enviornment in Kubernetes](https://youtu.be/vpWQeoaiRM4)

2. [Introduction to Argo CD](https://www.youtube.com/watch?v=aWDIQMbp1cc&feature=youtu.be&t=1m4s)

3. [Gitops Deployment and Kubernetes-using Argo CD](https://medium.com/riskified-technology/gitops-deployment-and-kubernetes-f1ab289efa4b)


## Authors

- [@manassharma](https://github.com/Manazsharma)

