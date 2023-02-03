## Default Security With EKS Cluster Setup

## EKS Authentication
> As previously mentioned, EKS is AWS’s managed Kubernetes service. As such, it takes the responsibility of deploying and managing the Kubernetes cluster off of you. In addition, EKS is integrated with other AWS services, among them is the Identity and Access Management (IAM), So, EKS uses IAM for cluster authentication, but the authorization still happens on native Kubernetes using RBAC (Role Based Access Control).

> Let's look at EKS’s authentication, and examine every step of the way

> *First, if you don’t have an EKS cluster, you can create one using one of [AWS’s guides](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html). Once our cluster is created, we want to be able to connect to it from our computer.* 

> *to connect to EKS, AWS instructs us to run the following command:*

> `aws eks update-kubeconfig --region REGION --name CLUSTER-NAME`

### the above command generates kube config file as below

```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://18366C4D507373B0179F69195904B314.gr7.us-east-2.eks.amazonaws.com
  name: test.us-east-2.eksctl.io
contexts:
- context:
    cluster: test.us-east-2.eksctl.io
    user: eks-admin@test.us-east-2.eksctl.io
  name: eks-admin@test.us-east-2.eksctl.io
current-context: eks-admin@test.us-east-2.eksctl.io
kind: Config
users:
- name: eks-admin@test.us-east-2.eksctl.io
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      args:
      - eks
      - get-token
      - --cluster-name
      - test
      - --region
      - us-east-2
      command: aws
      env:
      - name: AWS_STS_REGIONAL_ENDPOINTS
        value: regional
      interactiveMode: IfAvailable
      provideClusterInfo: false
```

### Let’s take a closer look at the users’ section in our new kubeconfig file:

```
users:
- name: eks-admin@test.us-east-2.eksctl.io
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      args:
      - eks
      - get-token
      - --cluster-name
      - test
      - --region
      - us-east-2
      command: aws
```

> We can see that the user authentication is different from the usual authentication methods (Bearer token, username and password, etc.). This exec section is specifying a command to run and arguments to use for authenticating the user. The command that is going to be run is:

> `aws --region us-east-2 eks get-token --cluster-name test`

> **with the above command, AWS EKS responds with an access token as below. This access token will be added to the HTTP requests that Kubectl sends to the API server**

```
root@myvm:~# aws --region us-east-2 eks get-token --cluster-name test

{"kind": "ExecCredential", "apiVersion": "client.authentication.k8s.io/v1beta1", "spec": {}, "status": {"expirationTimestamp": "2023-02-02T18:07:21Z", "token": "k8s-aws-v1.aHR0cHM6Ly9zdHMudXMtZWFzdC0yLmFtYXpvbmF3cy5jb20vP0FjdGlvbj1HZXRDYWxsZXJJZGVudGl0eSZWZXJzaW9uPTIwMTEtMDYtMTUmWC1BbXotQWxnb3JpdGhtPUFXUzQtSE1BQy1TSEEyNTYmWC1BbXotQ3JlZGVudGlhbD1BS0lBWjUyUDRRVTZOT0pQNk0yRiUyRjIwMjMwMjAyJTJGdXMtZWFzdC0yJTJGc3RzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyMzAyMDJUMTc1MzIxWiZYLUFtei1FeHBpcmVzPTYwJlgtQW16LVNpZ25lZEhlYWRlcnM9aG9zdCUzQngtazhzLWF3cy1pZCZYLUFtei1TaWduYXR1cmU9ZTliOGRlMTlhN2IwNjczNWNhNmUyMGEyZDIzNWMzMDE2MzNmMjg4YmQ3N2ExYTdiNzJjMWZlMmU3NTA3OTUzMw"}}
```


## Token Exploration 

#### The token is simply a base64 encoded string, if we decode it (without the k8s-aws-v1.), we will get the resulting string below:

> https://sts.amazonaws.com/?Action=GetCallerIdentity&Version=2011-06-15&X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential={ACCESS-KEY-ID}%2F20220201%2Fus-east-1%2Fsts%2Faws4_request&X-Amz-Date=20220201T073249Z&X-Amz-Expires=60&X-Amz-SignedHeaders=host%3Bx-k8s-aws-id&X-Amz-Signature=2de2d0807e2c4caa6dccd03e90faef0d65ba3876e0c35af892bad135e8e659d1

> This is an HTTP request to AWS STS (Security Token Service), and to be specific, it is a “GetCallerIdentity” request [GetCallerIdentity request documentation](https://docs.aws.amazon.com/STS/latest/APIReference/API_GetCallerIdentity.html).

1. This request is sent to the EKS cluster encoded as a token used for authentication. 
2. The EKS cluster receives the request, extracts the token, decodes it, and sends the same STS “GetCallerIdentity” request to the AWS STS service. 
3. The AWS STS response details provide our EKS cluster with the exact identity which is trying to perform the action. 
4. If the cluster gets a successful response for this request, it knows that AWS authenticated the user, and it gets the user’s identity.

#### So, what exactly is the response that the cluster gets?

> Since the cluster access the AWS STS “GetCallerIdentity” API, we can run a similar command in our local CLI.

```
nareshwar@mbpro ~ % aws sts get-caller-identity
{
    "UserId": "AIDAVVWXZIT7VDSXOHKCK",
    "Account": "8375905",
    "Arn": "arn:aws:iam::8375905:user/eks06"
}
```

> cluster gets same response as above, when we ran the “sts get-caller-identity” from the CLI

> **there is one last step to complete the authentication process, and that is to translate the AWS identity into a Kubernetes identity**

## “aws-auth” ConfigMap

> In general, a ConfigMap in Kubernetes is an API object that lets you store data in key-value pairs. Pods can consume ConfigMaps as environment variables, command-line arguments, or as configuration files in a volume. 

#### The main point of integration between Kubernetes and AWS IAM is the “aws-auth” ConfigMap

1. This ConfigMap object maps AWS identities to Kubernetes identities. 

2. It is not enough that the "GetCallerIdentity" request is successful, because the AWS identity has no meaning in native Kubernetes. 

3. You cannot assign access permissions without having a parallel Kubernetes identity that works with RBAC

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - username: system:node:{{EC2PrivateDNSName}}
      rolearn: arn:aws:iam::111122223333:role/my-role
      groups:
        - system:bootstrappers
        - system:nodes
    - username: my-console-viewer-role
      rolearn: arn:aws:iam::111122223333:role/my-console-viewer-role
      groups:
        - eks-console-dashboard-full-access-group    
  mapUsers: |
    - username: admin
      userarn: arn:aws:iam::111122223333:user/admin
      groups:
        - system:masters
    - username: my-user
      userarn: arn:aws:iam::444455556666:user/my-user
      groups:
        - eks-console-dashboard-restricted-access-group      
      
```

> **There are two main parts in the ”aws-auth” ConfigMap:**

1. **mapRoles** – Allows you to map an IAM role, using its ARN, to a Kubernetes identity or group. By that, each IAM identity that can assume that role, will have access to your cluster based on the permissions of the group/username it is mapped to.

2. **mapUsers** – Allows you to map an IAM user, using its ARN, to a Kubernetes user. Then, this user will have access to your cluster based on the roles/cluster roles that are bound to its Kubernetes user. You can also map IAM users to be a part of a Kubernetes group and have its permissions.

> Users that have permissions to edit the “aws-auth” ConfigMap, can become cluster admins by mapping themselves to a group that is already bound to a cluster-admin (“system:masters” group for instance).

> *Note that if you created your cluster using AWS CLI, the ”aws-auth” ConfigMap is not created automatically. Follow this guide to apply it to your cluster.*

> **If an AWS identity is mapped in your “aws-auth” ConfigMap to a Kubernetes identity, this identity will be able to access your cluster. The scope of access will be determined by the roles/cluster roles that are bound to this identity.**

## Summary 

> a successful flow of running a kubectl command with the help of a schema from [AWS’s documentation](https://docs.aws.amazon.com/eks/latest/userguide/cluster-auth.html)

<p align="center">
  <img src="https://docs.aws.amazon.com/images/eks/latest/userguide/images/eks-iam.png" alt="Cluster Auth"/>
</p>

1. **Pass Amazon Web Services Identity** - The “aws eks get-token” command run in the background while using the kubectl tool, and the token is attached to the Kubernetes API request.
2. **Verify Amazon Web Services Identity** - Kubernetes cluster uses the token and sends the decoded string to AWS STS to get the user’s identity.
3. **Role Based access control** - Kubernetes translates the AWS identity to a Kubernetes identity using the "aws-auth" ConfigMap and checks if this identity is authorized to perform the required action according to the roles/cluster roles that are bound to this identity.
4. **Kubernetes action allowed** – The action is executed, and the user gets its output.


> It is important to understand that managed Kubernetes services, though very fast and easy to use, come at a cost. You will not have complete control over your cluster, and parts of the deployment might be hidden. Before using EKS, or any other managed service, try wrapping your head around what AWS (or other cloud providers) is doing for you. The more knowledge you have on how things work behind the scenes, the more secure your cluster will be, especially when it comes to authentication.
