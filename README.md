# GitHub Actions Self-Hosted Runner on EKS
I'm inspired while reading [Github action with k8s self-hosted runner](https://medium.com/geekculture/github-actions-self-hosted-runner-on-kubernetes-55d077520a31), I explore how to deploy GitHub Actions Self-Hosted Runner on [Amazon EKS](https://aws.amazon.com/eks/) which is a managed service and certified Kubernetes conformant to run Kubernetes on AWS and on-premises.

## What is it difference between them?
It's easy to deploy the GitHub Actions Self-Hosted Runner on EKS as the steps in the above article. However sometimes we need to use AWS resources on GitHub Actions, for example, we need to push the Docker image built by GitHub Action to ECR. In the following paragraphs, I will use the case that pushes the docker image to ECR to show how to configure K8S and write the workflow of GitHub Actions.

Firstly, we just take a look at how to push the Docker image on K8S.
### Push the image on K8S:
There are 4 steps to push the image on K8S.
1. Build the Docker image
```YAML
- name: Build the Docker image
  run: docker build . --file Dockerfile --tag  ${{ env.CACHE_IAMGE_TAG }}
```
2. Configure AWS credentials
Use the [aws-actions/configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials) action to configure the GitHub Actions environment with environment variables containing AWS credentials and your desired region.
```YAML
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v1
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: YOUR_AWS_REGION
```
  The `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` is stored on GitHub Actions secrets.
  ![GitHub Actions secrets Demo](/assets/images/secretsDemo.png)
  
3. Log in to Amazon ECR
   Login the local Docker client to an Amazon ECR registry using [amazon-ecr-login](https://github.com/aws-actions/amazon-ecr-login).
```YAML
- name: Login to Amazon ECR
  id: login-ecr
  uses: aws-actions/amazon-ecr-login@v1
```
  The step named `login-ecr` will be used in the next step. And the step output is the Amazon ECR registry. The registry format is `aws_account_id`.dkr.ecr.`region`.amazonaws.com. 

4. Push the image to Amazon ECR repository
  ```YAML
- name: Push image to Amazon ECR
  env:
    ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
    ECR_REPOSITORY: YOUR_ECR_REPOSITORY
  run: |
    docker tag ${{ env.CACHE_IAMGE_TAG }} $ECR_REGISTRY/$ECR_REPOSITORY:$CACHE_IAMGE_TAG
    docker push $ECR_REGISTRY/$ECR_REPOSITORY:$CACHE_IAMGE_TAG
  ```
  There are two sub-steps
 *  Tag your image with the Amazon ECR registry, repository.
 * Push the image using the `docker push` command.

In the Step2, we use `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`stored on GitHub Actions secrets.
Due to Amazon EKS working with IAM, you can securely control access to AWS resources using IAM. You don't need to store the AWS credential on Github secrets. 

Next, let's see how to push the image on EKS.

## Push the image on EKS
We can follow the above article to deploy GitHub Actions on Kubernetes. In this section, I just focus on AWS Service Account when pushing the docker image to ECR.

### IAM role for the service account
We can associate an IAM role with a Kubernetes service account. This service account can then provide AWS permissions to the containers in any pod that uses that service account.
1. [Create an IAM OIDC provider for the cluster](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html)
  `eksctl utils associate-iam-oidc-provider --cluster YOUR-CLUSTER --approve`
2. Create an IAM role and attach an IAM policy to it with the permissions that your service accounts need
  *2.1 Create a policy*
  Create a policy `self_hosted_runner_policy` and add the following minimum permissions required for pushing images in an ECR repository.

```YAML
{
  "Version": "2012-10-17",
  "Statement": [
      {
          "Sid": "GetAuthorizationToken",
          "Effect": "Allow",
          "Action": [
              "ecr:GetAuthorizationToken"
          ],
          "Resource": "*"
      },
      {
          "Sid": "AllowPush",
          "Effect": "Allow",
          "Action": [
              "ecr:GetDownloadUrlForLayer",
              "ecr:BatchGetImage",
              "ecr:BatchCheckLayerAvailability",
              "ecr:PutImage",
              "ecr:InitiateLayerUpload",
              "ecr:UploadLayerPart",
              "ecr:CompleteLayerUpload"
          ],
          "Resource": "arn:aws:ecr:YOUR_REGION:YOUR_ACCOUNT_ID:repository/YOUR_ECR_REPOSITORY"
    }
  ]
}
``` 
  
  *2.2 Create the Role*
    Create a role `self_hosted_runner_role` with policy `self_hosted_runner_policy`.
    Add trusted entities that can assume the above role under specified conditions:

```YAML
{
"Version": "2012-10-17",
"Statement": [
    {
        "Effect": "Allow",
        "Principal": {
            "Federated": "arn:aws:iam::YOUR_ACCOUNT_ID:oidc-provider/oidc.eks.YOUR_REGION.amazonaws.com/id/YOUR_OIDC_PROVIDER_ID"
        },
        "Action": "sts:AssumeRoleWithWebIdentity",
        "Condition": {
            "StringEquals": {
                "oidc.eks.YOUR_REGION.amazonaws.com/id/YOUR_OIDC_ID:sub": "system:serviceaccount:YOUR_NAMESPACE:YOUR_SERVICE_ACCOUNT"
            }
        }
    }
  ]
}
```

### Associate an IAM role with a service account
  Create a service account `self-hosted-runner-service-account.yaml` with the role.
```YAML
apiVersion: v1
kind: ServiceAccount
metadata:
  name: self-hosted-runner-service-account
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::YOUR_ACCOUNT_ID:role/self_hosted_runner_role
```
### Configure the service account for the deployment
```YAML
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: runner-deployment
spec:
  template:
    spec:
      serviceAccountName: self-hosted-runner-service-account
      repository: YOUR_REPOSITORY
```
### Update workflow steps
The steps are almost the same as steps on K8S, except for now we don't need the step 2 `Configure AWS credentials` on K8S. We use service account with IAM role to automatically configure the AWS credentials.
```YAML
env:
  ECR_REGISTRY: YOUR_ACCOUNT_ID.dkr.ecr.YOUR_REGION.amazonaws.com
  ECR_REPOSITORY: YOUR_ECR_REPOSITORY
  CACHE_IAMGE_TAG: ${{ github.sha }}
jobs:
  build:
    steps:
      - name: Build the Docker image
        run: docker build . --file Dockerfile --tag $ECR_REGISTRY/$ECR_REPOSITORY:$CACHE_IAMGE_TAG
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1
      - name: Push image to Amazon ECR repository
        run: docker push $ECR_REGISTRY/$ECR_REPOSITORY:$CACHE_IAMGE_TAG
```
