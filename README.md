# The Credentials for GitHub Actions Self-Hosted Runner on AWS EKS
*Brief: Do not use AWS IAM User Access Keys directly in GitHub Actions Self-Hosted Runner on EKS. Instead, define an IAM role with minimum permissions for the Self-Hosted Runner and launch the Self-Hosted Runner with the role.*

The article [Github action with k8s self-hosted runner](https://medium.com/geekculture/github-actions-self-hosted-runner-on-kubernetes-55d077520a31) shows how to deploy [GitHub Actions self-hosted runners](https://docs.github.com/en/actions/hosting-your-own-runners/about-self-hosted-runners) as a container in [Amazon EKS](https://aws.amazon.com/eks/). In this article, [actions-runner-controller (ARC)](https://github.com/actions-runner-controller/actions-runner-controller) operates self-hosted runners for GitHub Actions on EKS.

Sometimes, we need to use AWS resources on GitHub Actions self-hosted runner. For example, after building a Docker image, we need to push the Docker image to AWS ECR. That requires that the self-hosted runner on EKS has permission to ECR. There are two ways to do it:
- Using AWS IAM User Access Keys
- Using AWS Assume IAM Role

## Using AWS IAM User Access Keys
There are 4 steps to push the image using access keys on Github Action.

*Note: The following workflow file only includes job steps and omits other configurations for concision.*
```YAML
env:
  CACHE_IAMGE_TAG: ${{ github.sha }}
jobs:
  build:
    steps:
      - name: Build the Docker image
        run: docker build . --file Dockerfile --tag  ${{ env.CACHE_IAMGE_TAG }} 
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: YOUR_AWS_REGION
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1
      - name: Push image to Amazon ECR
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: YOUR_ECR_REPOSITORY
        run: |
          docker tag ${{ env.CACHE_IAMGE_TAG }} $ECR_REGISTRY/$ECR_REPOSITORY:$CACHE_IAMGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$CACHE_IAMGE_TAG
```
1. Build the Docker image
2. Configure AWS credentials with user access keys
Use the [aws-actions/configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials) action to configure the GitHub Actions environment with environment variables containing user access keys and desired region.
The access keys are long-term credentials for an IAM user and consist of two parts: an access key ID `AWS_ACCESS_KEY_ID` and a secret access key `AWS_SECRET_ACCESS_KEY`.
The `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` are stored on GitHub Actions secrets.
![GitHub Actions secrets Demo](/assets/images/secretsDemo.png)
1. Log in to Amazon ECR
   Login the local Docker client to an Amazon ECR registry using [amazon-ecr-login](https://github.com/aws-actions/amazon-ecr-login).
  The output of step `login-ecr` that is the Amazon ECR registry will be used in the next step. The registry format is `aws_account_id`.dkr.ecr.`region`.amazonaws.com. 
4. Push the image to Amazon ECR repository
  There are two sub-steps:
    * Tag your image with the Amazon ECR registry, repository.
    * Push the image using the `docker push` command.

### Cron: Managing the access keys
In Step2, `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` are AWS IAM user credentials to authenticate requests. They are long-term credentials and need you to create, modify and rotate the access keys. If you leak or lose the access keys, you must delete the access key and create new pair. 
And they are stored on GitHub Actions secrets. So when you change the access keys on AWS, you must sync it to GitHub Actions secrets.

There is another option to manage identity and access, and it is temporary security credentials (IAM roles). Due to Amazon EKS being integrated with IAM deeply, you can securely control access to AWS resources using IAM role. Next, let's see how to use AWS Assume Role.
## Using AWS Assume Role
There are only 3 steps to push the image using assume Role on Github Action.
```YAML
env:
  CACHE_IAMGE_TAG: ${{ github.sha }}
jobs:
  build:
    steps:
      - name: Build the Docker image
        run: docker build . --file Dockerfile --tag  ${{ env.CACHE_IAMGE_TAG }} 
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1
      - name: Push image to Amazon ECR
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: YOUR_ECR_REPOSITORY
        run: |
          docker tag ${{ env.CACHE_IAMGE_TAG }} $ECR_REGISTRY/$ECR_REPOSITORY:$CACHE_IAMGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$CACHE_IAMGE_TAG
```
For now, we don't need step 2 `Configure AWS credentials` on Using AWS IAM User Access Keys. Also, we don't need to store the access keys on Github secrets.
We use AWS assume role to grant permission for the self-hosted runner on EKS to access ECR.
**Enable it in two steps:**  
* IAM roles for service accounts
* Configure the service account for the runner

#### IAM roles for service accounts
We can associate an IAM role with a Kubernetes service account. This service account can then provide AWS permissions to the containers in any pod that uses that service account.
1. [Create an IAM OIDC provider for the cluster](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html)
   To use IAM roles for service accounts in cluster, we must create an IAM OIDC Identity Provider. We can create an OIDC provider for cluster using `eksctl` or the AWS Management Console.
  `eksctl utils associate-iam-oidc-provider --cluster YOUR-CLUSTER --approve`
2. Create an IAM role and attach an IAM policy to it with the permissions that your service accounts need.
   
    2.1 Create a policy
    Create a policy `self_hosted_runner_policy` and add the following minimum permissions required for pushing images in an ECR repository.
    
    2.2 Create the Role
    Create a role `self_hosted_runner_role` with policy `self_hosted_runner_policy`.
    Add trusted entities that can assume the above role under specified conditions.


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

3. Associate an IAM role with a service account
We can associate an IAM role with a Kubernetes service account. This service account can then provide AWS permissions to the containers in any pod that uses that service account.
  Create a service account `self-hosted-runner-service-account.yaml` with the role.
```YAML
apiVersion: v1
kind: ServiceAccount
metadata:
  name: self-hosted-runner-service-account
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::YOUR_ACCOUNT_ID:role/self_hosted_runner_role
```
### Configure the service account for the runner
On [actions-runner-controller (ARC)](https://github.com/actions-runner-controller/actions-runner-controller), there are two ways to use self-hosted runner:
* Manage runners one by one with `Runner`.
* Manage a set of runners with `RunnerDeployment`.
  
`Runner` and `RunnerDeployment` kinds on ARC respectively correspond to the `Pod` and `Deployment` kinds on K8S. The following code takes `RunnerDeployment` for example.

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
## Feature work
In the above, We show how self-hosted runners on AWS EKS use credentials to access AWS resources.
Suppose we use GitHub-hosted runners or self-hosted runners on Other clouds; how to use credentials to access AWS resources. Is the only way to use User Access Key, or is there another way? Let's figure it out in feature.
## Reference:
[Best practices for managing AWS access keys](https://docs.aws.amazon.com/general/latest/gr/aws-access-keys-best-practices.html)

[IAM users](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users.html)

[Temporary security credentials in IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp.html)

[IAM roles for service accounts
](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)

[Introducing OIDC identity provider authentication for Amazon EKS
](https://aws.amazon.com/cn/blogs/containers/introducing-oidc-identity-provider-authentication-amazon-eks/)

[Authenticating users for your cluster from an OpenID Connect identity provider](https://docs.amazonaws.cn/en_us/eks/latest/userguide/authenticate-oidc-identity-provider.html)