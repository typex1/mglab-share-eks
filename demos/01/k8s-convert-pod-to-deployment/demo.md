## k8s-convert-pod-to-deployment

#### GIVEN:
  - A developer desktop with docker & git installed (AWS Cloud9)
  - A container image for a web app workload (ECS demo frontend) in Amazon ECR
  - A Kubernetes cluster (e.g. Amazon EKS)
  - A running pod

#### WHEN:
  - I inspect the Kubernetes manifest for the running Pod in YAML format

#### THEN:
  - I can convert that Pod spec to a *reproducible* Kubernetes manifest
  - I can convert that Pod manifest into a multi-pod Deployment manifest

#### SO THAT:
  - I can reuse that Pod manifest and no longer use `kubectl run`
  - I can use the Deployment to deployment multiple replicas of the Pod

#### [Return to Main Readme](https://github.com/bwer432/mglab-share-eks#demos)

---------------------------------------------------------------
---------------------------------------------------------------
### REQUIRES

- 00-setup-cloud9
- Either:
  - 03/create-cluster-eksctl-existing-vpc-advanced
  - 03/create-cluster-terraform
  - or similar
- 01/k8s-run-ecsdemo-frontend 
  - (or similar)

---------------------------------------------------------------
---------------------------------------------------------------
### DEMO

#### 0: Reset Cloud9 Instance environ from previous demo(s).
- Reset your region & AWS account variables in case you launched a new terminal session:
```
cd ~/environment/mglab-share-eks/demos/01/docker-build-wordpress/
export C9_REGION=$(curl --silent http://169.254.169.254/latest/dynamic/instance-identity/document |  grep region | awk -F '"' '{print$4}')
export C9_AWS_ACCT=$(curl -s http://169.254.169.254/latest/dynamic/instance-identity/document | grep accountId | awk -F '"' '{print$4}')
clear
echo $C9_REGION
echo $C9_AWS_ACCT
```

#### 1. Check for a running container to use as a model.
- Get a list of pods running in the default Kubernetes namespace.
- Use `api-resources` to show the `apiVersion` and `kind` values for Pod and Deployment.
- Generate the Kubernetes manifest for that Pod in YAML format.
```
kubectl get pods
kubectl api-resources -o wide >artifacts/k8s-api.txt
c9 open artifacts/k8s-api.txt
kubectl get pod frontend -o yaml >artifacts/frontend-pod.yaml 
c9 open artifacts/frontend-pod.yaml
```

#### 2. Prune out the pod runtime information to leave a small Pod spec.
- Delete the annotations, creationTimestamp, namespace, resourceVersion, and uid from metadata.
- You should just have labels and name in the metadata section.
- Change the key of the one label from "run" to "app". This is a more customary label.
- Delete everything from the container spec except its `image` and `name`.
- Remove all other fields beside the containers list from the Pod `spec`.
- Remove the whole `status` section of the Pod.
- You should be left with something like this (your image URI will be different):
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: frontend
  name: frontend
spec:
  containers:
  - image: 042617493216.dkr.ecr.us-east-2.amazonaws.com/ecsdemo/frontend
    name: frontend
```
- Save your changes.
- Close the editor tab if you like.
 
#### 5. Replace the current pod with a new one created from your manifest.
- Delete the existing pod.
```
kubectl delete pod frontend
```
- Now create a replacment pod using your manifest.
- Note: `kubectl create` could be used instead of `kubectl apply`
```
kubectl apply -f artifacts/frontend-pod.yaml
```

#### 6. Copy that Pod spec to be a starting point for a deployment.
- Copy the Pod spec to a Deployment spec, 
- however instead of using `cp pod.yaml deploy.yaml`
- the following illustrates the hard way.
- You could choose to create the manifest from scratch.
```
manifest=artifacts/frontend-pod.yaml
deploymanifest=artifacts/frontend-deploy.yaml
head -2 $manifest | sed -e 's@v1@apps/v1@' -e 's/Pod/Deployment/' >$deploymanifest
head -6 $manifest | tail -4 >>$deploymanifest
cat <<EOF >>$deploymanifest
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
EOF
head -9 $manifest | tail -7 | sed -e 's/^/    /' >>$deploymanifest
c9 open $deploymanifest
```

#### 7. Replace the solo pod with your deployment.
- Delete the existing pod.
```
kubectl delete pod frontend
```
- Now create a deployment using your manifest.
- Note: `kubectl create` could be used instead of `kubectl apply`
```
kubectl apply -f artifacts/frontend-deploy.yaml
```

#### 8. Check what the `Deployment` created.
- List the deployments.
```
kubectl get deployments   # or deployment or deploy
```
- List the replica sets.
```
kubectl get replicasets   # or replicaset or rs
```
- List the pods.
```
kubectl get pods          # or pod or po 
```

#### 9. Now create a manifest the easy way.
- Find the image URI used earlier for the pod.
- Get the usage information for `kubectl create deployment`.
- Use `kubectl create` to make a Deployment spec.
```
repo=$(grep image: artifacts/frontend-pod.yaml | sed -e 's/.*image: //')
kubectl create deployment --help | grep ^Usage: -A 1
kubectl create deployment frontend --image=$repo --replicas 3 --dry-run=client -o yaml >artifacts/frontend-create.yaml
c9 open artifacts/frontend-create.yaml
kubectl apply -f artifacts/frontend-create.yaml
```

---------------------------------------------------------------
---------------------------------------------------------------
### DEPENDENTS
- None

---------------------------------------------------------------
---------------------------------------------------------------
### CLEANUP
- Do not cleanup if you plan to run any dependent demos
```
export C9_REGION=$(curl --silent http://169.254.169.254/latest/dynamic/instance-identity/document |  grep region | awk -F '"' '{print$4}')
aws ecr delete-repository --region $C9_REGION --repository-name eks-demo-wordpress --force
```
