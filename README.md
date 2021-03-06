
# The Best Practice for AWS Security Credentials Management
**GitHub Actions Self-Hosted Runner on AWS EKS**

*Brief: Do not use AWS IAM User Access Keys directly in GitHub Actions Self-Hosted Runner on EKS. Instead, define an IAM role with minimum permissions for the Self-Hosted Runner and launch the Self-Hosted Runner with the role.*

The article [Github action with k8s self-hosted runner](https://medium.com/geekculture/github-actions-self-hosted-runner-on-kubernetes-55d077520a31) shows how to deploy [GitHub Actions self-hosted runners](https://docs.github.com/en/actions/hosting-your-own-runners/about-self-hosted-runners) as a container in [Amazon EKS](https://aws.amazon.com/eks/). In the above article, [actions-runner-controller (ARC)](https://github.com/actions-runner-controller/actions-runner-controller) is used to operate self-hosted runners for GitHub Actions.

Sometimes, we need to access AWS resources on GitHub Actions self-hosted runner. For example, after building a Docker image, we need to push the Docker image to AWS ECR. That requires the self-hosted runner running on EKS to have permission to access the ECR. There are usually two ways to do it:
- *IAM User Access Keys*: A general practice when using GitHub-hosted runners to run jobs.
- *IAM Roles for Service Accounts (IRSA)*: A Best practice when using self-hosted runners on EKS to run jobs.

Before we look into the best practice, firstly, let's dive into the general practice.

## General Practice: IAM User access keys
Let's take an example. The following is a GitHub Action workflows sample, including three steps to push the docker image to AWS ECR.
* Configure AWS credentials
* Log in to Amazon ECR
* Push the image to the Amazon ECR repository

*NOTE: The following workflow file only includes job steps to push the docker image and omits other preparative steps for concision, such as checkout code and building the docker image.*
```YAML
jobs:
  build:
    steps:
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
![](/assets/images/accesskey.jpg)

#### 1. Configure AWS credentials
User Access keys are credentials to access AWS resources. In step 1, use the [aws-actions/configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials) action to configure the GitHub Actions environment with environment variables containing user access keys and desired region. The AWS SDKs will detect the environment variables to determine the credentials and region to use for the following AWS API calls.

The access keys are long-term credentials for an IAM user and consist of two parts: an access key ID `AWS_ACCESS_KEY_ID` and a secret access key `AWS_SECRET_ACCESS_KEY`. The `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` are stored on GitHub Actions secrets.
 ![GitHub Actions secrets Demo](/assets/images/secretsDemo.png)
How to create the User Access keys? It includes three sub-steps: 1) Create one IAM user; 2) Attach a policy to the user; 3) [Create access keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html#Using_CreateAccessKey).

In sub-step 2: the policy defines the user's minimum permissions required for pushing images in an ECR repository. The policy called `self_hosted_runner_policy` as flowing:
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
In the policy, `ecr:GetAuthorizationToken` action retrieves an authorization token that is a temporary credential to access the ECR repository and the next Step will use the temporary credential.

#### 2. Log in to Amazon ECR
   Login the local Docker client to an Amazon ECR registry using [amazon-ecr-login](https://github.com/aws-actions/amazon-ecr-login).
  The next Step will use the output(Amazon ECR registry) of the this Step `login-ecr`. The registry format is `aws_account_id`.dkr.ecr.`region`.amazonaws.com. 
#### 3. Push the image to the Amazon ECR repository
There are two sub-steps:
  * Tag your image with the Amazon ECR registry, repository.
  * Push the image using the `docker push` command.

### Why it's not good enough
In Step 2, `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` are AWS IAM user credentials to authenticate requests. They are long-term credentials and need you to create, modify and rotate the access keys. If you leak or lose the access keys, you must delete the access key and create new pair. 
And they are stored on GitHub Actions secrets. So when you manage the access keys on AWS, you must sync it to GitHub Actions secrets.

There is another option to manage identity and access, and it is temporary security credentials (IAM roles). Due to Amazon EKS being integrated with IAM deeply, you can securely control access to AWS resources using the IAM role. Next, let's see how to use AWS Assume Role.
## Best practice: IAM Roles for Service Accounts (IRSA)
Let's look at another example.
```YAML
jobs:
  build:
    steps:
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
Using IRSA, there are only 2 steps on the Github Action workflow. Step 1 `Configure AWS credentials` on `Using AWS IAM User Access Keys` is not needed. Of course, we don't need to store the access keys on Github secrets.

You might be curious about what credentials are used without Step `Configr AWS credentials`. It is working by IRSA. In IRSA, by combining an OpenID Connect (OIDC) identity provider and Kubernetes service account annotations, you can use IAM roles at the pod level, then gain access to AWS services and resources. If you want to know the details, the article [Diving into IAM Roles for Service Accounts](https://aws.amazon.com/cn/blogs/containers/diving-into-iam-roles-for-service-accounts/) helps you understand how the various pieces join together and what really happens behind the scene.  

In this pushing docker image case, briefly, before AWS SDK evaluates `GetAuthorizationToken` to get the authorization token of the ECR registry, AWS SDK will call `sts:AssumeRoleWithWebIdentity` API to get temporary security credentials(short-term credentials) that will be used by `GetAuthorizationToken`. AWS SDK handles the two sub-steps,, and the application developer doesn't care how to get the credentials. If you are intensely interested in the process of how SDK calls `sts:AssumeRoleWithWebIdentity` API before `GetAuthorizationToken`, the `WebIdentityTokenCredentialsProvider` [class](https://github.com/aws/aws-sdk-java/blob/fd409dee8ae23fb8953e0bb4dbde65536a7e0514/aws-java-sdk-core/src/main/java/com/amazonaws/auth/WebIdentityTokenCredentialsProvider.java) on `aws-java-sdk-core` is a good breakthrough.

![](/assets/images/webidentity.jpg)

Next, we will focus on how to enable IRSA. 

Enable IRSA to access AWS resources in [three steps](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html):
* Create an IAM OIDC provider for the cluster
* Create an IAM role and associate it with a service account
* Configure the service account for the runner
#### 1. Create an IAM OIDC provider for the cluster
   To use IAM roles for service accounts in a cluster, we must create an IAM OIDC Identity Provider. 
   We can create an OIDC provider for the cluster using `eksctl`.
  `eksctl utils associate-iam-oidc-provider --cluster YOUR-CLUSTER --approve`
#### 2. Create an IAM role and associate it with a service account
* Create a policy `self_hosted_runner_policy` same with `IAM User access keys` way
* Create the Role
Create a role `self_hosted_runner_role` with policy `self_hosted_runner_policy`.
And add trusted entities as flowing policy. This policy allows your service account to assume the role of using `sts:AssumeRoleWithWebIdentity` action.
The principal for this policy is the created OIDC provider. `sts:AssumeRoleWithWebIdentity` action returns a role's temporary credential. In subsequent AWS API calls to access resources in the account that owns the role.

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
* Associate an IAM role with a service account
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
### 3. Configure the service account for the runner
On [actions-runner-controller (ARC)](https://github.com/actions-runner-controller/actions-runner-controller), there are two ways to use the self-hosted runner:
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
## Conclusion
This article presents two approaches (IAM User Access Keys and IRSA) in which Github action Self-Hosted runners on AWS EKS gain access to AWS resources and services. We don't recommend the User Access Keys on this scene due to the security and management of these credentials' lifecycle. Instead, the IRSA is best practice, we should define an IAM role with minimum permissions for the Self-Hosted Runner and launch the Self-Hosted Runner with the role.

## Future Work
In some scenarios, we use GitHub-Hosted runners or Self-Hosted runners on Other clouds; how to use credentials to access AWS resources and services. Is the only way to use User Access Keys, or is there another way? Let's figure it out in the future.

## Reference:
About Credential: 
* [Best practices for managing AWS access keys](https://docs.aws.amazon.com/general/latest/gr/aws-access-keys-best-practices.html)
* [Working with AWS Credentials](https://aws.amazon.com/cn/blogs/containers/diving-into-iam-roles-for-service-accounts/), [IAM users](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users.html)
* [Temporary security credentials in IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp.html)
  
About IRSA implementation:
* [Introducing fine-grained IAM roles for service accounts](https://aws.amazon.com/cn/blogs/opensource/introducing-fine-grained-iam-roles-service-accounts/)
* [Diving into IAM Roles for Service Accounts](https://aws.amazon.com/cn/blogs/containers/diving-into-iam-roles-for-service-accounts/)

About enable IRSA: 
* [IAM roles for service accounts
](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
* [Introducing OIDC identity provider authentication for Amazon EKS
](https://aws.amazon.com/cn/blogs/containers/introducing-oidc-identity-provider-authentication-amazon-eks/)
* [Authenticating users for your cluster from an OpenID Connect identity provider](https://docs.amazonaws.cn/en_us/eks/latest/userguide/authenticate-oidc-identity-provider.html)
* [Configure Service Accounts for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#service-account-token-volume-projection)