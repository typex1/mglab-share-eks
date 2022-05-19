## k8s-dashboard

#### GIVEN:
  - A developer desktop with docker & git installed (AWS Cloud9)
  - An EKS cluster created via eksctl from demo 03/create-cluster-eksctl-existing-vpc-advanced or 03/create-cluster-terraform

#### WHEN:
  - I deploy the Kubernetes Dashboard to my cluster 

#### THEN:
  - I will be able to visualize (and administrate) my EKS cluster using the open-source [Kubernetes Dashboard](https://github.com/kubernetes/dashboard)

#### SO THAT:
  - I can see how to use EKS with open-source administration and observability tooling

#### [Return to Main Readme](https://github.com/virtmerlin/mglab-share-eks#demos)

---------------------------------------------------------------
---------------------------------------------------------------
### REQUIRES
- 00-setup-cloud9
- 03/create-cluster-eksctl-existing-vpc-advanced or 03/create-cluster-terraform

---------------------------------------------------------------
---------------------------------------------------------------
### DEMO

#### 0: Reset Cloud9 Instance environ from previous demo(s).
- Reset your region & AWS account variables in case you launched a new terminal session:
```
cd ~/environment/mglab-share-eks/demos/05/k8s-dashboard
export C9_REGION=$(curl --silent http://169.254.169.254/latest/dynamic/instance-identity/document |  grep region | awk -F '"' '{print$4}')
export C9_AWS_ACCT=$(curl -s http://169.254.169.254/latest/dynamic/instance-identity/document | grep accountId | awk -F '"' '{print$4}')
export AWS_ACCESS_KEY_ID=$(cat ~/.aws/credentials | grep aws_access_key_id | awk '{print$3}')
export AWS_SECRET_ACCESS_KEY=$(cat ~/.aws/credentials | grep aws_secret_access_key | awk '{print$3}')
clear
echo $C9_REGION
echo $C9_AWS_ACCT
```

#### 1: Apply manifests to create the Kubernetes Dashboard core objects.
- Update our kubeconfig to interact with the cluster created in 04-create-advanced-cluster-eksctl-existing-vpc.
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.5/aio/deploy/recommended.yaml
```

#### 2: Create an eks-admin service account and cluster role binding
- This demo will create a service account that grants cluster-admin permissions to the Dashboard.  This is the highest levels of privilege, but will allow you to both visualize and manage the cluster.
```
kubectl apply -f artifacts/esk-admin-service-account.yaml
```

#### 3: Connect to the Dashboard
- Given that the Dashboard is using cluster-admin levels of permissions, you will connect to the Dashboard using kubectl proxy. This ensures that the Dashboard is only available locally, and not via any public facing service.
- However, first we must have an authentication token
```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep eks-admin | awk '{print $1}')
```
- Execute the following command on your local machine (will not work in Cloud9) to expose the REST API on port 8001 (default port)
```
kubectl proxy
```
- Then, open the following URL:
```
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#!/login
```


#### 5: Observe and Administrate
- You can see various k8s objects via the Dashboard
- Using the "+" icon in the upper right you can apply manifests.


---------------------------------------------------------------
---------------------------------------------------------------
### DEPENDENTS

---------------------------------------------------------------
---------------------------------------------------------------
### CLEANUP
- Do not cleanup if you plan to run any dependent demos, otherwise delete the namespace (cascading delete, and the service account)
```
kubectl delete ns kubernetes-dashboard
kubectl delete sa eks-admin -n kube-system
```
