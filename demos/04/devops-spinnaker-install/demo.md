## devops-spinnaker-install

#### GIVEN:
  - A developer desktop with docker & git installed (AWS Cloud9)
  - An EKS cluster created via eksctl from demo 03/create-cluster-eksctl-existing-vpc-advanced
  - A Dockerfile for wordpress php frontend

#### WHEN:
  - I download Spinnaker components
  - I set up Spinnaker prerequisites
  - I deploy Spinnaker using Halyard

#### THEN:
  - I will get a Spinnaker environment in my Kubernetes cluster

#### SO THAT:
  - I can build, test, and deploy app components using Spinnaker

#### [Return to Main Readme](https://github.com/bwer432/mglab-share-eks#demos)

---------------------------------------------------------------
---------------------------------------------------------------
### REQUIRES
- 00-setup-cloud9
- 03/create-cluster-eksctl-existing-vpc-advanced

---------------------------------------------------------------
---------------------------------------------------------------
### DEMO

#### 0: Reset Cloud9 Instance environ from previous demo(s).
- Reset your region & AWS account variables in case you launched a new terminal session:
```
cd ~/environment/mglab-share-eks/demos/04/devops-docker-push-ecr/
export C9_REGION=$(curl --silent http://169.254.169.254/latest/dynamic/instance-identity/document |  grep region | awk -F '"' '{print$4}')
export C9_AWS_ACCT=$(curl -s http://169.254.169.254/latest/dynamic/instance-identity/document | grep accountId | awk -F '"' '{print$4}')
export AWS_ACCESS_KEY_ID=$(cat ~/.aws/credentials | grep aws_access_key_id | awk '{print$3}')
export AWS_SECRET_ACCESS_KEY=$(cat ~/.aws/credentials | grep aws_secret_access_key | awk '{print$3}')
clear
echo $C9_REGION
echo $C9_AWS_ACCT
```

### 1: Install Halyard
Note: Halyard is used to raise the sails of spinnaker, as in sailing.
References:
- [Spinnaker Provider for Kubernetes V2](https://spinnaker.io/docs/setup/install/providers/kubernetes-v2/)
- [Set up the Spinnaker Kubernetes provider for Amazon EKS](https://spinnaker.io/docs/setup/install/providers/kubernetes-v2/aws-eks/)
- [Install Halyard @ Spinnaker setup page](https://spinnaker.io/docs/setup/install/providers/kubernetes-v2/aws-eks/#4-install-halyard)
```bash
# Download and configure Halyard
curl -O https://raw.githubusercontent.com/spinnaker/halyard/master/install/debian/InstallHalyard.sh
sudo useradd halyard
sudo bash InstallHalyard.sh
sudo update-halyard
# Verify the installation
hal -v
```

#### 1. (alternative) 
References: [Managing Spinnaker using Spinnaker Operator in Amazon EKS](https://aws.amazon.com/blogs/opensource/managing-spinnaker-using-spinnaker-operator-in-amazon-eks/)

### delete the below fragments from the ECR demo
DELETE everything below this line until the DEPENDENTS and CLEANUP sections

#### 1: Create ECR repository to push wordpress image up to.
- Create ECR repository to push to & share our image:
```
aws ecr create-repository --repository-name eks-demo-devops-docker-push-wordpress-ecr --region $C9_REGION
```

#### 2: Update our kubeconfig to interact with the cluster created in 03-create-advanced-cluster-eksctl-existing-vpc.
- Review your kubeconfig:
```
eksctl utils write-kubeconfig --cluster cluster-eksctl --region $C9_REGION --authenticator-role-arn arn:aws:iam::${C9_AWS_ACCT}:role/cluster-eksctl-creator-role
kubectl config view --minify | grep 'cluster-name' -A 1
kubectl get ns
```

#### 3:  Git clone a local Dockerfile for Wordpress from virtmerlin repo to build.
- Clone the git repo containing the Dockerfile:
```
cd ~/environment
git clone https://github.com/virtmerlin/mglab-wordpress.git
```

#### 4: Build the OCI image on the local C9 Desktop.
- Build the OCI image:
```
cd ~/environment/mglab-wordpress/Dockerfile
docker build -f Dockerfile . -t eks-demo-wordpress:latest -t eks-demo-wordpress:v1.0
docker images
```

#### 5: Login to ECR private registry
- Get ECR auth token:
```
aws ecr get-login-password --region $C9_REGION | docker login --username AWS --password-stdin $C9_AWS_ACCT.dkr.ecr.$C9_REGION.amazonaws.com
```

#### 6: Tag & Push the local OCI image to ECR
- Tag & push image to ECR:
```
docker tag eks-demo-wordpress:latest $C9_AWS_ACCT.dkr.ecr.$C9_REGION.amazonaws.com/eks-demo-devops-docker-push-wordpress-ecr:latest
docker push $C9_AWS_ACCT.dkr.ecr.$C9_REGION.amazonaws.com/eks-demo-devops-docker-push-wordpress-ecr:latest
```
- Open the ECR console [link](https://console.aws.amazon.com/ecr/repositories/?) and scan &/or review your OCI image.

#### 7: Update Wordpress Image in a running deployment.
- Create/Update the wordpress workload deployments & services:
```
cd ~/environment/mglab-share-eks/demos/04/devops-docker-push-ecr
cat ./artifacts/k8s-all-in-one-fargate.yaml  | sed "s/<REGION>/$C9_REGION/" | kubectl apply -f -

kubectl -n wordpress-fargate get deployment.v1.apps/wordpress -o yaml | grep image:
```
- Update the  K8s deployment to use the image you created earlier:
```
kubectl -n wordpress-fargate set image deployment.v1.apps/wordpress wordpress=$C9_AWS_ACCT.dkr.ecr.$C9_REGION.amazonaws.com/eks-demo-devops-docker-push-wordpress-ecr:latest

kubectl -n wordpress-fargate get deployment.v1.apps/wordpress -o yaml | grep image:
```
- Watch the updated image get deployed:
```
watch kubectl get pods -o wide -n wordpress-fargate
```
- Test the updated Wordpress Front End Service
```
echo "http://"$(kubectl get svc wordpress -n wordpress-fargate \
--output jsonpath='{.status.loadBalancer.ingress[0].hostname}')
```

---------------------------------------------------------------
---------------------------------------------------------------
### DEPENDENTS

---------------------------------------------------------------
---------------------------------------------------------------
### CLEANUP
- Do not cleanup if you plan to run any dependent demos
```
export C9_REGION=$(curl --silent http://169.254.169.254/latest/dynamic/instance-identity/document |  grep region | awk -F '"' '{print$4}')
echo $C9_REGION
aws ecr delete-repository --repository-name eks-demo-devops-docker-push-wordpress-ecr --region $C9_REGION --force
```
