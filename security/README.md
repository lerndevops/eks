## Security With EKS

## EKS Authentication
> As previously mentioned, EKS is AWS’s managed Kubernetes service. As such, it takes the responsibility of deploying and managing the Kubernetes cluster off of you. In addition, EKS is integrated with other AWS services, among them is the Identity and Access Management (IAM), So, EKS uses IAM for cluster authentication, but the authorization still happens on native Kubernetes using RBAC (Role Based Access Control).

> Let's look at EKS’s authentication, and examine every step of the way

### First We use below Command to get Auth data for user 

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

```
root@myvm:~# aws --region us-east-2 eks get-token --cluster-name test

{"kind": "ExecCredential", "apiVersion": "client.authentication.k8s.io/v1beta1", "spec": {}, "status": {"expirationTimestamp": "2023-02-02T18:07:21Z", "token": "k8s-aws-v1.aHR0cHM6Ly9zdHMudXMtZWFzdC0yLmFtYXpvbmF3cy5jb20vP0FjdGlvbj1HZXRDYWxsZXJJZGVudGl0eSZWZXJzaW9uPTIwMTEtMDYtMTUmWC1BbXotQWxnb3JpdGhtPUFXUzQtSE1BQy1TSEEyNTYmWC1BbXotQ3JlZGVudGlhbD1BS0lBWjUyUDRRVTZOT0pQNk0yRiUyRjIwMjMwMjAyJTJGdXMtZWFzdC0yJTJGc3RzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyMzAyMDJUMTc1MzIxWiZYLUFtei1FeHBpcmVzPTYwJlgtQW16LVNpZ25lZEhlYWRlcnM9aG9zdCUzQngtazhzLWF3cy1pZCZYLUFtei1TaWduYXR1cmU9ZTliOGRlMTlhN2IwNjczNWNhNmUyMGEyZDIzNWMzMDE2MzNmMjg4YmQ3N2ExYTdiNzJjMWZlMmU3NTA3OTUzMw"}}
```
