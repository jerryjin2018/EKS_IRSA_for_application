## EKS_IRSA_for_application
assign the service account to the pod or deployment, the pod will be with specific IAM Role

### 1. setup the environment parameters
``` 
export CLUSTER_NAME=eksgo108
export AWS_REGION=cn-northwest-1
export EKS_NAMESPACE=cal-test
``` 

### 2. check  IAM OIDC provider url
``` 
aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.identity.oidc.issuer" --output text
``` 
``` 
[ec2-user@ip-172-31-1-111 ekslab]$ aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.identity.oidc.issuer" --output text
https://oidc.eks.cn-northwest-1.amazonaws.com.cn/id/5602B729FBF0E7DB1200AC53ADBXXXXX
``` 

### 3. create IAM policy

iam_policy_cal.json

``` 
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:*",
                "s3-object-lambda:*"
            ],
            "Resource": "*"
        }
    ]
}
``` 
``` 
aws iam create-policy \
    --policy-name AWSCalIAMPolicy \
    --policy-document file://iam_policy_cal.json
``` 
```
[ec2-user@ip-172-31-1-111 ekslab]$ aws iam create-policy \
>     --policy-name AWSCalIAMPolicy \
>     --policy-document file://iam_policy_cal.json
{
    "Policy": {
        "PolicyName": "AWSCalIAMPolicy",
        "PolicyId": "ANPAZTGKYWNBMYQFXRYLZ",
        "Arn": "arn:aws-cn:iam::xxxxxxxxxxxxx:policy/AWSCalIAMPolicy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2022-01-11T07:41:06+00:00",
        "UpdateDate": "2022-01-11T07:41:06+00:00"
    }
}
``` 

### 4. IAM policy ARN
``` 
POLICY_NAME=$(aws iam list-policies --query 'Policies[?PolicyName==`AWSCalIAMPolicy`].Arn' --output text --region ${AWS_REGION})
``` 

### 5. create a service account
``` 
eksctl create iamserviceaccount \
--cluster=${CLUSTER_NAME} \
--namespace=${EKS_NAMESPACE} \
--name=aws-cal-sa \
--attach-policy-arn=${POLICY_NAME} \
--override-existing-serviceaccounts \
--approve
``` 
``` 
[ec2-user@ip-172-31-1-111 ekslab]$ eksctl create iamserviceaccount \
> --cluster=${CLUSTER_NAME} \
> --namespace=${EKS_NAMESPACE} \
> --name=aws-cal-sa \
> --attach-policy-arn=${POLICY_NAME} \
> --override-existing-serviceaccounts \
> --approve
2022-01-11 07:55:36 [ℹ]  eksctl version 0.76.0
2022-01-11 07:55:36 [ℹ]  using region cn-northwest-1
2022-01-11 07:55:36 [ℹ]  1 existing iamserviceaccount(s) (kube-system/aws-load-balancer-controller) will be excluded
2022-01-11 07:55:36 [ℹ]  1 iamserviceaccount (cal-test/aws-cal-sa) was included (based on the include/exclude rules)
2022-01-11 07:55:36 [!]  metadata of serviceaccounts that exist in Kubernetes will be updated, as --override-existing-serviceaccounts was set
2022-01-11 07:55:36 [ℹ]  1 task: { 
    2 sequential sub-tasks: { 
        create IAM role for serviceaccount "cal-test/aws-cal-sa",
        create serviceaccount "cal-test/aws-cal-sa",
    } }2022-01-11 07:55:36 [ℹ]  building iamserviceaccount stack "eksctl-eksgo108-addon-iamserviceaccount-cal-test-aws-cal-sa"
2022-01-11 07:55:36 [ℹ]  deploying stack "eksctl-eksgo108-addon-iamserviceaccount-cal-test-aws-cal-sa"
2022-01-11 07:55:36 [ℹ]  waiting for CloudFormation stack "eksctl-eksgo108-addon-iamserviceaccount-cal-test-aws-cal-sa"
2022-01-11 07:55:53 [ℹ]  waiting for CloudFormation stack "eksctl-eksgo108-addon-iamserviceaccount-cal-test-aws-cal-sa"
2022-01-11 07:55:53 [ℹ]  created serviceaccount "cal-test/aws-cal-sa"
```  

### 6.check serviceaccount
``` 
eksctl get iamserviceaccount --cluster ${CLUSTER_NAME} --name aws-cal-sa --namespace ${EKS_NAMESPACE}
``` 
``` 
[ec2-user@ip-172-31-1-111 ekslab]$ eksctl get iamserviceaccount --cluster ${CLUSTER_NAME} --name aws-cal-sa --namespace ${EKS_NAMESPACE}
2022-01-11 07:57:12 [ℹ]  eksctl version 0.76.0
2022-01-11 07:57:12 [ℹ]  using region cn-northwest-1
NAMESPACE       NAME            ROLE ARN
cal-test        aws-cal-sa      arn:aws-cn:iam::xxxxxxxxxxxxx:role/eksctl-eksgo108-addon-iamserviceaccount-cal-Role1-1FHUVNHFBQRBY
``` 

### 7. update deployment with serviceAccount, and create deployment/pod and ingress 
serviceAccount: aws-cal-sa   
serviceAccountName: aws-cal-sa   

``` 
#---
#apiVersion: v1
#kind: Namespace
#metadata:
#  name: cal-test
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cal-deployment
  namespace: cal-test
  labels:
    app: cal-test
spec:
  replicas: 3
  selector:
    matchLabels:
      app: cal-test
  template:
    metadata:
      labels:
        app: cal-test
    spec:
      # add following serviceAccount info
      serviceAccount: aws-cal-sa
      serviceAccountName: aws-cal-sa
      containers:
      - name: cal
        image: xxxxxxxxxxxxx.dkr.ecr.cn-northwest-1.amazonaws.com.cn/jlr/cal-api:v1
        ports:
        - containerPort: 5000
---
apiVersion: v1
kind: Service
metadata:
  name: cal-service-clusterip
  namespace: cal-test
spec:
  type: ClusterIP
  selector:
    app: cal-test
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 5000
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  namespace: cal-test
  name: cal-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
    - http:
        paths:
          - path: /*
            backend:
              serviceName: cal-service-clusterip
              servicePort: 8080
``` 
