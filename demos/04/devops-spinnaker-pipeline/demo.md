## devops-spinnaker-pipeline

#### GIVEN:
  - A developer desktop with docker & git installed (AWS Cloud9)
  - An EKS cluster created via eksctl from demo 03/create-cluster-eksctl-existing-vpc-advanced
  - A Dockerfile for wordpress php frontend

#### WHEN:
  - I use the Spinnaker GUI
  - I deploy a helm chart

#### THEN:
  - I will get a my app deployed to Amazon EKS via Spinnaker

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

- Set up cluster context.
- Adjust accordingly
```
export ACCOUNT_ID=$(curl -s http://169.254.169.254/latest/dynamic/instance-identity/document | grep accountId | awk -F '"' '{print$4}')
export EKS_REGION=us-west-2
aws eks list-clusters --region $EKS_REGION
export EKS_CLUSTER=$(aws eks list-clusters --region $EKS_REGION --query clusters[0] --output text)
export EKS_CLUSTER=dev-cluster
export C9_USER=$(aws sts get-caller-identity --query Arn --output text | sed -e 's/^.*\///')
kubectl config use-context $C9_USER@$EKS_CLUSTER.$EKS_REGION.eksctl.io
```

References: 
- [Managing Spinnaker using Spinnaker Operator in Amazon EKS](https://aws.amazon.com/blogs/opensource/managing-spinnaker-using-spinnaker-operator-in-amazon-eks/)
- [Spinnaker Provider for Kubernetes V2](https://spinnaker.io/docs/setup/install/providers/kubernetes-v2/)
- [Set up the Spinnaker Kubernetes provider for Amazon EKS](https://spinnaker.io/docs/setup/install/providers/kubernetes-v2/aws-eks/)
- [Install Halyard @ Spinnaker setup page](https://spinnaker.io/docs/setup/install/providers/kubernetes-v2/aws-eks/#4-install-halyard)

#### 1. Deploy helm chart

- [Managing Spinnaker using Spinnaker Operator in Amazon EKS](https://aws.amazon.com/blogs/opensource/managing-spinnaker-using-spinnaker-operator-in-amazon-eks/)
Step 11 – Deploy Helm chart

Let’s deploy a Helm-based product catalog application to Amazon EKS using Spinnaker pipeline.

Access Spinnaker UI: Grab the load balancer URL from the previous chapter, or use the below command to get the load balancer URL.
kubectl -n spinnaker get spinsvc spinnaker -w
Open the URL in the browser. You should see the below Spinnaker UI.

Screenshot showing the Spinnaker UI, highlighting the search bar
Create application: Click on Create Application and enter name as product-detail and email as your email. Leave the rest of the fields as default. Then, click on “Create.”
New Application Dialog in the Spinnaker UI
Spinnaker UI showing the successfully added application
Create pipeline: Click on Pipelines under product-detail and click on link Configure a new pipeline and add the name helm-pipeline.
Spinnaker UI showing the Create New Pipeline dialog
Set up trigger: You should now be in the Configuration page.
Now click on Add Trigger under Automated Triggers
Select Type as Docker Registry.
In the Registry Name dropdown you should see the value my-ecr-registry, select that.
In the Organization dropdown you should see the value eks-workshop-demo, select that.
In the Image dropdown you should see the value eks-workshop-demo/test-detail, select that.
Click on Save Changes.
This is the ECR registry we set up in Spinnaker manifest in Step 7 – Configure ECR Artifact.

Spinnaker UI showing the successfully created Automated Trigger
Evaluate variable configuration
Click on Add Stage and select type as Evaluate Variables from the dropdown.
Add the variable name as image_name and value as ${trigger['tag']}.
Add another variable name as repo_name and value as $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/eks-workshop-demo/test-detail. Replace $ACCOUNT_ID and $AWS_REGION based on your setup.
Click on Save Changes. We will be using these variables in the next Bake Stage.
Spinnaker UI showing the Evaluate Variables Configuration settings
Set up bake stage
Click on Add Stage and select Type as Bake Manifest from the dropdown.
Select Template Renderer as Helm3 and enter name as workshop-detail and enter workshop as namespace.
Select Expected Artifact as Define a new artifact
Select your Git account that shows in dropdown. This is the Git account we had setup in Spinnaker manifest in Step 8.
And then enter the below git location in the Content URL and add master as the Commit/Branch. This is to provide the the Helm template for the deployment.
https://api.github.com/repos/aws-containers/eks-app-mesh-polyglot-demo/contents/workshop/productcatalog_workshop-1.0.0.tgz
Keep the branch as “master.”
Under the Overrides section, select Add new artifact.
Select Expected Artifact as Define a new artifact
Select your Git account that shows in dropdown. This is the Git account we had setup in Spinnaker manifest in Step 8.
Enter the below git location in the Content URL. This is to provide the overrides for the Helm template using values.yaml.
https://api.github.com/repos/aws-containers/eks-app-mesh-polyglot-demo/contents/workshop/helm-chart/values.yaml
Keep the branch as “master”
Under the Overrides key/value section, click on “Add override.”
Enter first key as detail.image.repository for repository and value as ${repo_name}.
Enter second key as detail.image.tag for tag and value as ${image_name}.
The keys are based on the Values.yaml from the Helm chart and the values are the variables that we set in previous step “Evaluate Variables.”
Edit the Produces Artifacts and change the name to helm-produced-artifact and click Save Artifact. Then, click Save Changes.
Spinnaker UI showing the Bake Manifest Configuration
Spinnaker UI highlighting the Expected Artifact configuration dialog
Set up deploy stage
Click on Add Stage and select “Type” as Deploy (Manifest) from the dropdown, and give a name as Deploy proddetail
Select Account as spinnaker-workshop from the dropdown. This is the EKS account we had setup in Spinnaker manifest in Step 9 – Add EKS account.
Select Artifact and then select helm-produced-artifact from the dropdown for Manifest Artifact and click Save Changes.
Spinnaker UI showing the customized Manifest Artifact setting
Step 12 – Test deployment

Push the new container image to ECR for testing trigger. To ensure that the Amazon ECR trigger will work in Spinnaker UI:
First, change the code to generate a new docker image digest. Note: The Amazon ECR trigger in Spinnaker does not work for same docker image digest.
Go to ~/environment/eks-app-mesh-polyglot-demo/workshop/apps/catalog_detail/app.js and replace the line "vendors":[ "ABC.com"] with "vendors":[ "ABC.com","XYZ.com"]
Ensure that the image tag (APP_VERSION) you are adding below does not exist in the Amazon ECR repository eks-workshop-demo/test-detail otherwise the trigger will not work. Spinnaker pipeline only triggers when a new version of image is added to ECR.
Then, run the below command.
cd ~/environment/eks-app-mesh-polyglot-demo/workshop
aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
export APP_VERSION=5.0 ## pick a version that is not there in the ECR
export ECR_REPOSITORY=eks-workshop-demo/test-detail
TARGET=$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPOSITORY:$APP_VERSION
docker build -t $TARGET apps/catalog_detail
docker push $TARGET
Building/Pushing Container images for the first time to Amazon ECR may take around 3-5 minutes. You can ignore any warnings you get due to the npm upgrade.

Watch the pipeline getting triggered
You should see the image version 5.0 get triggered.
Spinnaker UI showing the image version 5.0 getting triggered
You will see that Docker push triggers a deployment in the pipeline.
Spinnaker UI showing the deployment in the pipepline
Below are the Execution Details of pipeline:
Spinnaker UI showing the Execution Details of the pipeline
Spinnaker UI showing it at the Bake stage of the pipeline
Spinnaker UI showing it at the Deploy stage of the pipeline
Get deployment details
Click on Clusters and you can see the deployment of frontend, prodcatalog, and proddetail service below.
Spinnaker UI showing the deployment of frontend, prodcatalog, and proddetail
Click on the LoadBalancer icon link for frontend and you should see below information. Click on that Load Balancer link or Paste the link on browser.
Spinnaker UI screen highlighting the LoadBalancer icon for the frontend service
You should see the service up and running as below.
Product Catalog Application dialog showing the finished service architecture
Now add a product. Below, we’ve used “1“ as “id“ and “Table“ as “name.“ And you see that an additional vendor, XYZ.com, is added  from the new container image for proddetail service we pushed into ECR.
Product Catalog Application dialog showing additional vendor XYZ.com
You can also go to the terminal and confirm the deployment details.
kubectl get all -n workshop
You can see the below output

NAME READY STATUS RESTARTS AGE
pod/frontend-7b78bc4cbb-fr2mz 1/1 Running 0 16m
pod/prodcatalog-f6d7bffb5-rjbz2 1/1 Running 0 16m
pod/proddetail-75cd46fb7b-k82lg 1/1 Running 0 16m
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
service/frontend LoadBalancer 10.100.221.87 aa76467f53d53419aa273bf96b8cdd47-XXXXX.us-east-2.elb.amazonaws.com 80:32022/TCP 10h
service/prodcatalog ClusterIP 10.100.213.2 <none> 5000/TCP 10h
service/proddetail ClusterIP 10.100.144.79 <none> 3000/TCP 10h

NAME READY UP-TO-DATE AVAILABLE AGE
deployment.apps/frontend 1/1 1 1 10h
deployment.apps/prodcatalog 1/1 1 1 10h
deployment.apps/proddetail 1/1 1 1 10h

Cleanup

Delete Spinnaker artifacts when finished with this walkthrough.

for i in $(kubectl get crd | grep spinnaker | cut -d" " -f1) ; do
kubectl delete crd $i
done

kubectl delete ns spinnaker-operator

kubectl delete ns spinnaker

cd ~/environment
rm config.yaml

rm -rf spinnaker-tools
rm -rf spinnaker-operator

helm uninstall workshop
kubectl delete ns workshop
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
