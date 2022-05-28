## devops-helm-for-ecsdemo

#### GIVEN:
  - A developer desktop with docker & git installed (AWS Cloud9)
  - An EKS cluster created via eksctl from demo: 03/create-cluster-eksctl-existing-vpc-advanced
  - Container images for 3 ecsdemo components in Amazon ECR repos.

#### WHEN:
  - I install the helm cli
  - I refer to some existing ECR repositories
  - I build a hierarchy of 3 helm charts on my C9 desktop

#### THEN:
  - I will get a local helm chart with subcharts

#### SO THAT:
  - I can build a helm chart
  - I can deploy all three deployments and services together

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
cd ~/environment/mglab-share-eks/demos/04/devops-helm-chart-build-push-ecr/
export C9_REGION=$(curl --silent http://169.254.169.254/latest/dynamic/instance-identity/document |  grep region | awk -F '"' '{print$4}')
export C9_AWS_ACCT=$(curl -s http://169.254.169.254/latest/dynamic/instance-identity/document | grep accountId | awk -F '"' '{print$4}')
export AWS_ACCESS_KEY_ID=$(cat ~/.aws/credentials | grep aws_access_key_id | awk '{print$3}')
export AWS_SECRET_ACCESS_KEY=$(cat ~/.aws/credentials | grep aws_secret_access_key | awk '{print$3}')
clear
echo $C9_REGION
echo $C9_AWS_ACCT
```

#### 1: Install the helm cli.
- Install helm v3:
```
curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get-helm-3 > get_helm.sh
chmod 700 get_helm.sh
./get_helm.sh
```

#### 2: Create Helm Chart & store locally on C9 instance.
- Build Chart:
```
cd artifacts
helm create ecsdemo
```

#### 3: Query the Amazon Elastic Contaner Registry (Amazon ECR) for a container image.
- Look for an image called ecsdemo/frontend
```
region=us-east-2
repo=$(aws ecr describe-repositories --region $region --repository-names ecsdemo/frontend --query repositories[].repositoryUri --output text)
echo $repo
```
- Copy the repository value from the previous command.
- ***You copied that repo value, right?***

#### 4: Edit the `values.yaml` in this ecsdemo helm chart.
- Open the values.yaml in an editor.
```
c9 open ecsdemo/values.yaml
```
- Add another comment line toward the top after the first line (a new line 2):
```
# This is for the ecsdemo-frontend component.
```
- Edit the image in the chart's `values.yaml` file.
- Change the `.image.repository` value from "nginx" 
- to now be the value you copied from Amazon ECR earlier.
- e.g. (your account ID will be different than this)
```
repository: 042617493216.dkr.ecr.us-east-2.amazonaws.com/ecsdemo/frontend
```
- Change the `tag` value to latest. You can keep the quotation marks here if you like.
```
tag: "latest"
```
- Change the value of `fullnameOverride` to ecsdemo-frontend. You can keep the quotes.
- We are overriding this name because our chart is called "ecsdemo" but 
- we want the actual component being deployed to be called ecsdemo-frontend.
```
fullnameOverride: "ecsdemo-frontend"
```
- Keep the editor open. We'll make a few more changes in the next step.

#### 5: Make more edits to the `values.yaml` in this ecsdemo helm chart.
- Scroll down to the `service` section. This may be around line 40.
- Change the `.service.type` value from "ClusterIP" to "LoadBalancer".
- You do not need quotation marks. 
- Be sure to keep two spaces at the beginning of the line.
```
  type: LoadBalancer
```
- Leave the port number as it is, with a value of `80`.
- Add another line after the `port`. 
- Indent this, call it `targetPort`, and give it a value of `3000`.
- (this value matches the TCP port the `ecsdemo-frontend` Ruby-on-Rails app listens on.)
```
  targetPort: 3000
```
- Save your changes to `values.yaml`. You can close the editor if you want.

#### 6: Edit the `deployment.yaml` file in the `templates` folder of the chart.
- Open the `ecsdemo/templates/deployment.yaml`. This file needs a few changes.
```
c9 open ecsdemo/templates/deployment.yaml
```
- Notice that most of this `deployment.yaml` file is standard YAML
- as a Kubernetes manifest for creating or updating a `Deployment` object.
- However, as a part of a Helm chart, there are substitutions in double curly braces.
- Scroll down to the `ports` section. This may be around line 36.
- Change the `containerPort` value from `80` to `{{ .Values.service.targetPort }}`.
- The edited line should look similar to this:
```
              containerPort: {{ .Values.service.targetPort }}
```
- Note that this is a more manageable value than putting `3000` here directly.
- Although in this case, your value of `3000` in the `values.yaml` file will
- be injected in here when Helm evaluates this template, with this technique
- you can simply change the port number to another value in the `values.yaml`
- if your needs change in the future (or for other similar charts).
 
#### 7. Change the `livenessProbe` value as well.
- Scroll down to the `livenessProbe` section. This may be around line 40.
- Change the `port` value from `http` to `{{ .Values.service.targetPort }}`.
- The edited line should look similar to this:
```
              port: {{ .Values.service.targetPort }}
```

#### 8. Change the `readinessProbe` value as well.
- Scroll down to the `readinessProbe` section. This may be around line 44.
- Change the `port` value from `http` to `{{ .Values.service.targetPort }}`.
- The edited line should look similar to this:
```
              port: {{ .Values.service.targetPort }}
```

#### 9: Save the `deployment.yaml` file.
- Save the `deployment.yaml` file.
- You may close the editor if you wish.

#### 10: Edit the `service.yaml` file in the `templates` folder of the chart.
- Open the `ecsdemo/templates/service.yaml`. This file needs one little change.
```
c9 open ecsdemo/templates/service.yaml
```
- Find the `ports` section. This may be around line 9.
- Change the `targetPort` value from `http` to `{{ .Values.service.targetPort }}`.
- The edited line should look similar to this:
```
      targetPort: {{ .Values.service.targetPort }}
```
- Save the `service.yaml` file.
- You may close the editor as you wish.

#### 11: Deploy this version of your `ecsdemo` helm chart.
- Use `helm install` to deploy your `Deployment` and `Service` using this chart.
```
helm install ecsdemo ./ecsdemo
```
- Note that the first use of the name `ecsdemo` is the name of the installation.
- The second use of the name, as the relative path `./ecsdemo` is a reference to
- the helm chart you just created and edited.

#### 12: Connect to your app.
- Copy the values from the app installation instructions generated by `helm install`.
- Paste them in your shell and run them. They may resemble he following:
```
export SERVICE_IP=$(kubectl get svc --namespace default ecsdemo-frontend --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
echo http://$SERVICE_IP:80
```
- Copy the resulting URL.
- Paste it in a new tab of your web browser.
- NOTE: You may have to wait for the `LoadBalancer` type `Service` resource to spin up.
- Retry as needed.
- Close the browser tab when satisifed.

%% DRAFT below this line

#### Later
- Change the `replicaCount` value from 1 to 3. This line should now be:
```
replicaCount: 1
```
- Scroll down to the `.autoscaling` section. This may be around line 73.
- Change the value of `.autoscaling.enabled` from "false" to "true".
- You don't need quotes.
```
  enabled: true
```
- Leave `minReplicas` as it is, with a value of 1.
- Change `maxReplicas` from 100 to 4.
```
  maxReplicas: 4
```

%% event DRAFTIER vestigal stuff from another demo below this line

```
- Show chart possible values to override:
```
helm show chart demo-nginx-helm
helm show values demo-nginx-helm
```
- Deploy our nginx chart with default values:
```
helm install my-nginx demo-nginx-helm
helm status my-nginx -n default
helm ls -A
```
- Show what the helm Chart created in the cluster:
```
kubectl get deploy -n default -o yaml
```
- Show the passed paramters that were given when deploying the helm chart:
```
helm get values my-nginx -n default
helm get values my-nginx -n default --all
```

#### 7: Update the Deployed Helm Chart.
- Show current image for the deployment:
```
kubectl get deploy my-nginx-demo-nginx-helm -n default -o yaml | grep image
```
- Override chart values for image with public ECR image:
```
helm upgrade \
      --set image.repository=public.ecr.aws/u3e9a9s8/nginx \
      --set image.tag=latest \
      my-nginx ./demo-nginx-helm  -n default
```
- Show updated image for the deployment:
```
kubectl get deploy my-nginx-demo-nginx-helm -n default -o yaml | grep image
```
- See how helm captures state with default secrets backend:
```
kubectl get secret -n default | grep nginx
```
- Use helm cli to 'Rollback' revision:
```
helm history my-nginx -n default
sleep 3
helm rollback my-nginx 1 -n default
```
- Show original image for the deployment:
```
kubectl get deploy my-nginx-demo-nginx-helm -n default -o yaml | grep image
helm history my-nginx -n default
```

#### 8: Interact with well known public Helm Chart Repos.
- Add public repos & Show External Chart Values:
```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```
- Search for a chart in a public repo:
```
helm search repo eks
helm search repo eks --version ^1.0.0
```
- Inspect chart in a public repo:
```
helm show values bitnami/wordpress
```
- Pull a well known public chart down to inspect it locally:
```
helm pull bitnami/wordpress
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
export C9_AWS_ACCT=$(curl -s http://169.254.169.254/latest/dynamic/instance-identity/document | grep accountId | awk -F '"' '{print$4}')
aws ecr delete-repository --repository-name demo-nginx-helm --region $C9_REGION --force
eksctl utils write-kubeconfig --cluster cluster-eksctl --region $C9_REGION --authenticator-role-arn arn:aws:iam::${C9_AWS_ACCT}:role/cluster-eksctl-creator-role
helm delete my-nginx -n default
```
