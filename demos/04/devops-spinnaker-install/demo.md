## devops-spinnaker-install

#### GIVEN:
  - A developer desktop with docker & git installed (AWS Cloud9)
  - An EKS cluster created via eksctl from demo 03/create-cluster-eksctl-existing-vpc-advanced
  - A Dockerfile for wordpress php frontend

#### WHEN:
  - I download Spinnaker components
  - I set up Spinnaker prerequisites
  - I deploy Spinnaker using the Spinnaker Operator

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

#### 1. Download Spinnaker Operator artifacts
References: 
- [Managing Spinnaker using Spinnaker Operator in Amazon EKS](https://aws.amazon.com/blogs/opensource/managing-spinnaker-using-spinnaker-operator-in-amazon-eks/)
- [Spinnaker Provider for Kubernetes V2](https://spinnaker.io/docs/setup/install/providers/kubernetes-v2/)
- [Set up the Spinnaker Kubernetes provider for Amazon EKS](https://spinnaker.io/docs/setup/install/providers/kubernetes-v2/aws-eks/)
- [Install Halyard @ Spinnaker setup page](https://spinnaker.io/docs/setup/install/providers/kubernetes-v2/aws-eks/#4-install-halyard)
```
export VERSION=1.2.5
echo $VERSION
mkdir -p artifacts
cd artifacts
mkdir -p spinnaker-operator && cd spinnaker-operator
bash -c "curl -L https://github.com/armory/spinnaker-operator/releases/download/v${VERSION}/manifests.tgz | tar -xz"
```

#### 2. Install Spinnaker Operator Custom Resource Definitions (CRDs)
```
kubectl api-resources | grep CustomResourceDefinition
# Have to use Kubernetes v1.21 or older due to this:
# Warning: apiextensions.k8s.io/v1beta1 CustomResourceDefinition is deprecated in v1.16+, unavailable in v1.22+; use apiextensions.k8s.io/v1 CustomResourceDefinition
# Attempt to hack version number did not work... unknown properties therein.
# sed -e 's@apiextensions.k8s.io/v1beta1@apiextensions.k8s.io/v1@' <deploy/crds/spinnaker.io_spinnakeraccounts_crd.yaml >spinacct_crd.yaml
# cp spinacct_crd.yaml deploy/crds/spinnaker.io_spinnakeraccounts_crd.yaml 
# sed -e 's@apiextensions.k8s.io/v1beta1@apiextensions.k8s.io/v1@' <deploy/crds/spinnaker.io_spinnakerservices_crd.yaml >spinsvcs_crd.yaml
# cp spinsvcs_crd.yaml deploy/crds/spinnaker.io_spinnakerservices_crd.yaml 
kubectl apply -f deploy/crds/
```

#### 3. Install Spinnaker Operator
- Use `Cluster` mode for use across all namespaces.
```
kubectl create ns spinnaker-operator
kubectl -n spinnaker-operator apply -f deploy/operator/cluster
```

#### 4. Validate that the Spinnaker Operator pod is running.
- This may take about a minute and a half to start both containers in the pod.
```
kubectl get pod -n spinnaker-operator
```

#### 5. Configure Spinnaker release version
- Pick a release from https://spinnaker.io/community/releases/versions/ and export that version. Below, we are using the latest Spinnaker release that was available when this blog was written.
```
export SPINNAKER_VERSION=1.26.6
```

#### 6. Configure Amazon S3 artifacts
```
export S3_BUCKET=spinnaker-workshop-$(cat /dev/urandom | LC_ALL=C tr -dc "[:alpha:]" | tr '[:upper:]' '[:lower:]' | head -c 10)
aws s3 mb s3://$S3_BUCKET --region $EKS_REGION
aws s3api put-public-access-block \
--bucket $S3_BUCKET \
--public-access-block-configuration "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true" \
--region $EKS_REGION
echo $S3_BUCKET
```

#### 7. Associate IAM OIDC provider
- Associate an IAM OIDC provider with your cluster, if you don't already have one.
```
eksctl utils associate-iam-oidc-provider --cluster $EKS_CLUSTER --region $EKS_REGION --approve
```

#### 8. Create IAM Service Account for S3 bucket
```
export S3_SERVICE_ACCOUNT=s3-access-sa
eksctl create iamserviceaccount \
    --name $S3_SERVICE_ACCOUNT \
    --namespace spinnaker \
    --cluster $EKS_CLUSTER \
    --region $EKS_REGION \
    --attach-policy-arn arn\:aws\:iam::aws\:policy/AmazonS3FullAccess \
    --approve \
    --override-existing-serviceaccounts
```

#### 9. Confirm details of IAM service account for S3 access
```
echo $S3_SERVICE_ACCOUNT
kubectl describe sa $S3_SERVICE_ACCOUNT -n spinnaker
```

#### 10. Configure Amazon ECR artifact
```
# cd ~/environment/eks-app-mesh-polyglot-demo
export ECR_REPOSITORY=eks-workshop-demo/test-detail
export APP_VERSION=1.0
aws ecr get-login-password --region $EKS_REGION | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$EKS_REGION.amazonaws.com
aws ecr describe-repositories --repository-name $ECR_REPOSITORY --region $EKS_REGION >/dev/null 2>&1 || \
  aws ecr create-repository --repository-name $ECR_REPOSITORY --region $EKS_REGION >/dev/null
TARGET=$ACCOUNT_ID.dkr.ecr.$EKS_REGION.amazonaws.com/$ECR_REPOSITORY:$APP_VERSION
docker pull nginx
docker tag nginx $TARGET
docker push $TARGET
```

#### 11. Create a ConfigMap for the ECR token
```
cd .. # to artifacts (out of spinnaker-operator)
cat << EOF > config.yaml
interval: 30m # defines refresh interval
registries: # list of registries to refresh
  - registryId: "$ACCOUNT_ID"
    region: "$EKS_REGION"
    passwordFile: "/etc/passwords/my-ecr-registry.pass"
EOF
kubectl -n spinnaker create configmap token-refresh-config --from-file config.yaml
```

#### 12. Confirm ConfigMap for ECR access
```
kubectl describe configmap token-refresh-config -n spinnaker
```

#### 13. Add a GitHub repository
- Set up environment variables to access a GitHub repo as a source of artifacts. If you actually want to use a file from the GitHub commit in your pipeline, you’ll need to configure GitHub as an artifact source in Spinnaker. So we need the GitHub credentials to access the repository from Spinnaker.
```
export GITHUB_USER=<your_github_username>
export GITHUB_TOKEN=<your_github_accesstoken>
```

#### 14. Download Spinnaker Tools
```
git clone https://github.com/armory/spinnaker-tools.git
cd spinnaker-tools
go mod download all
go build
cd ..
```

#### 15. Set up environment variables for Spinnaker
```
export CONTEXT=$(kubectl config current-context)
export SOURCE_KUBECONFIG=${HOME}/.kube/config
export SPINNAKER_NAMESPACE="spinnaker"
export SPINNAKER_SERVICE_ACCOUNT_NAME="spinnaker-ws-sa"
export DEST_KUBECONFIG=${HOME}/Kubeconfig-ws-sa
echo $CONTEXT
echo $SOURCE_KUBECONFIG
echo $SPINNAKER_NAMESPACE
echo $SPINNAKER_SERVICE_ACCOUNT_NAME
echo $DEST_KUBECONFIG
```

#### 14. Create service account for namespace
```
./spinnaker-tools/spinnaker-tools create-service-account \
    --kubeconfig ${SOURCE_KUBECONFIG} \
    --context ${CONTEXT} \
    --output ${DEST_KUBECONFIG} \
    --namespace ${SPINNAKER_NAMESPACE} \
    --service-account-name ${SPINNAKER_SERVICE_ACCOUNT_NAME}
```

#### 15. Set up SpinnakerService account manifest
- Could use spinnaker-service.yml from any of spinnaker-operator/deploy/spinnaker/:
  - basic
  - complete
  - kustomize
- However, here we cook our own.
```
# cd spinnaker-operator
# mv deploy/spinnaker/basic/spinnakerservice.yml ./backup-basic-spinnaker-service.yaml
# cat <<EOF >deploy/spinnaker/basic/spinnakerservice.yml
cat <<EOF >spinnakerservice.yml
apiVersion: spinnaker.io/v1alpha2
kind: SpinnakerService
metadata:
  name: spinnaker
spec:
  spinnakerConfig:
    config:
      version: $SPINNAKER_VERSION   # the version of Spinnaker to be deployed
      persistentStorage:
        persistentStoreType: s3
        s3:
          bucket: $S3_BUCKET
          rootFolder: front50
          region: $EKS_REGION
      deploymentEnvironment:
        sidecars:
          spin-clouddriver:
          - name: token-refresh
            dockerImage: quay.io/skuid/ecr-token-refresh:latest
            mountPath: /etc/passwords
            configMapVolumeMounts:
            - configMapName: token-refresh-config
              mountPath: /opt/config/ecr-token-refresh
      features:
        artifacts: true
      artifacts:
        github:
          enabled: true
          accounts:
          - name: $GITHUB_USER
            token: $GITHUB_TOKEN  # GitHub's personal access token. This fields supports "encrypted" references to secrets.
      providers:
            dockerRegistry:
              enabled: true
            kubernetes:
              enabled: true
              accounts:
              - name: spinnaker-workshop
                requiredGroupMembership: []
                providerVersion: V2
                permissions:
                dockerRegistries:
                  - accountName: my-ecr-registry
                configureImagePullSecrets: true
                cacheThreads: 1
                namespaces: [spinnaker,workshop]
                omitNamespaces: []
                kinds: []
                omitKinds: []
                customResources: []
                cachingPolicies: []
                oAuthScopes: []
                onlySpinnakerManaged: false
                kubeconfigFile: kubeconfig-sp  # File name must match "files" key
              primaryAccount: spinnaker-workshop  # Change to a desired account from the accounts array
    files: 
        kubeconfig-sp: |
$(awk ' {print "            "$0}' <$DEST_KUBECONFIG)
            # <REPLACE_ME_WITH_FILE_CONTENT>
            # ~/Kubeconfig-ws-sa
    profiles:
        clouddriver:
          dockerRegistry:
            enabled: true
            primaryAccount: my-ecr-registry
            accounts:
            - name: my-ecr-registry
              address: https://$ACCOUNT_ID.dkr.ecr.$EKS_REGION.amazonaws.com
              username: AWS
              passwordFile: /etc/passwords/my-ecr-registry.pass
              trackDigests: true
              repositories:
              - $ECR_REPOSITORY
        igor:
          docker-registry:
            enabled: true
    service-settings:
      front50:
        kubernetes:
          serviceAccountName: $S3_SERVICE_ACCOUNT
          securityContext:
            fsGroup: 100
  # spec.expose - This section defines how Spinnaker should be publicly exposed
  expose:
    type: service  # Kubernetes LoadBalancer type (service/ingress), note: only "service" is supported for now
    service:
      type: LoadBalancer
EOF
```

#### 15. Configure SpinnakerService account 
```
# Replace the <REPLACE_ME_WITH_FILE_CONTENT> in the above section of deploy/spinnaker/basic/spinnakerservice.yml with the kubeconfig content from ${HOME}/Kubeconfig-ws-sa.
# From the terminal, Go to ${HOME}/Kubeconfig-ws-sa (in my case it was /home/ec2-user/Kubeconfig-ws-sa) and copy the kubeconfig text starting from “apiVersion…” to the end of file.
# Align the tab of the added file content to look as below
# This should now have been done automatically by the use of awk in the prior shell snippet.
# Verify the Spinnaker manifest
cat $DEST_KUBECONFIG
vi deploy/spinnaker/basic/spinnakerservice.yml
# By now we have completed our configuration for Spinnaker, and the SpinnakerService manifest located at deploy/spinnaker/basic/spinnakerservice.yml should look similar to below.
# Note: “$$$” in the YAML below is just a placeholder. Do not copy the content under “kubeconfig-sp:” in this file. Copy the content from ${HOME}/Kubeconfig-ws-sa to this section.
```

#### 16. Confirm environment variables
```
echo $ACCOUNT_ID
echo $EKS_REGION
echo $SPINNAKER_VERSION
echo $GITHUB_USER
echo $GITHUB_TOKEN
echo $S3_BUCKET
echo $S3_SERVICE_ACCOUNT
echo $ECR_REPOSITORY
```

#### 17. Install Spinnaker
- We have already substituted variables and files by using cat earlier,
- so we don't need to use envsubst here. Just apply the custom manifest.
```
# envsubst < deploy/spinnaker/basic/spinnakerservice.yml | kubectl -n spinnaker apply -f -
kubectl -n spinnaker apply -f spinnakerservice.yml 
```

#### 18. Wait for services to be created
```
# Get all the resources created
kubectl get svc,pod -n spinnaker
# Watch the install progress.
kubectl -n spinnaker get spinsvc spinnaker -w
kubectl logs deploy/spinnaker-operator -n spinnaker-operator -c halyard
kubectl logs deploy/spinnaker-operator -n spinnaker-operator -c spinnaker-operator
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
