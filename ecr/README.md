## Push Image to Public ecr 
add below IAM Policy to IAM User you using from CLI 

AmazonElasticContainerRegistryPublicFullAccess

## using AWS CLI get login password 
aws ecr-public get-login-password --region us-east-1

copy the token generated

## Login to public ecr as below 
docker login public.ecr.aws --username AWS --password "paste-here-token-genereated-above"

## Create a public repo on Your AWS account by going to ECR page 

get the registry alias generated

ex: public.ecr.aws/j4c4j1j1 --- here j4c4j1j1 is registry alias of you public images on public ecr

## Push an image
docker pull docker.io/library/alpine:latest

docker tag docker.io/library/alpine:latest public.ecr.aws/j4c4j1j1/alpine:v1

docker push public.ecr.aws/j4c4j1j1/alpine:v1
