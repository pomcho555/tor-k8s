1. set environment variables for your cluster
```sh
CLUSTER_NAME=tor-k8s
FARGATE_PROFILE_NAME=tor-k8s-fargate-profile
AWS_REGION=ap-northeast-1
AWS_ACCOUNT_ID=$(aws sts get-caller-identity | jq -r '.Account')
```

2. create an EKS cluster with fargate
```sh
eksctl create cluster --name $CLUSTER_NAME --version 1.18 --fargate --profile stg
```

3. set fragate profile    
```sh
eksctl create fargateprofile --cluster $CLUSTER_NAME --name $FARGATE_PROFILE_NAME --namespace tor-k8s --profile stg
```

## INSTALL ALB INGRESS CONTROLER

See also:https://aws.amazon.com/jp/blogs/news/using-alb-ingress-controller-with-amazon-eks-on-fargate/

1. Determine whether you have an existing IAM OpenID Connect (OIDC) provider created for your cluster.

```sh
aws eks describe-cluster --name $CLUSTER_NAME --query "cluster.identity.oidc.issuer" --output text
```

Output is like:    
```
https://oidc.eks.<region-code>.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E
```

2. Download an IAM policy
```sh
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.1.0/docs/install/iam_policy.json
```

3. Create an IAM policy
```sh
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

4. Create an IAM role
```sh
eksctl create iamserviceaccount \
  --cluster=$CLUSTER_NAME \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::$AWS_ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve
```

4.ex. if you appear to be error, run below command
```sh
eksctl utils associate-iam-oidc-provider --region=ap-northeast-1 --cluster=tor-k8s --approve
```

5. Install the AWS Load Balancer Controller using Helm.
```sh
kubectl apply -k "https://raw.githubusercontent.com/aws/eks-charts/master/stable/aws-load-balancer-controller/crds/crds.yaml"
```

Or
```sh
curl -o crds.yaml https://raw.githubusercontent.com/aws/eks-charts/master/stable/aws-load-balancer-controller/crds/crds.yaml
kubectl apply -f crds.yaml
```

6. Add chart
```sh
helm repo add eks https://aws.github.io/eks-charts
```


7. 

```sh
STACK_NAME=eksctl-$CLUSTER_NAME-cluster
```

7. Fetch and set a VPC ID
```sh
VPC_ID=$(aws cloudformation describe-stacks --stack-name "$STACK_NAME" | jq -r '[.Stacks[0].Outputs[] | {key: .OutputKey, value: .OutputValue}] | from_entries' | jq -r '.VPC')
```

**Important**    
You should do an tags to the above VPC. Here is a tag.
```
kubernetes.io/cluster/<cluster-name>: shared
```




```sh
helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller \
  --set clusterName=$CLUSTER_NAME \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=$AWS_REGION --set vpcId=$VPC_ID \
  -n kube-system
```

8. Verify if the controller is installed
```sh
kubectl get deployment -n kube-system aws-load-balancer-controller
```

**For you application**    
You have to attach below `annotations` into your `ingress resource`
```yaml
annotations:
    kubernetes.io/ingress.class: alb
```

**EX** Deploy tor-k8s    
```sh
eksctl create fargateprofile --cluster $CLUSTER_NAME --region $AWS_REGION --name test-tor-k8s --namespace tor-proxy
```

```sh
kubectl apply -f tor_proxy_full.yaml
```

## (Option) Deoploy a sample application

```sh
eksctl create fargateprofile --cluster $CLUSTER_NAME --region $AWS_REGION --name alb-sample-app --namespace game-2048
```

```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.1.0/docs/examples/2048/2048_full.yaml
```

confirm
```sh
kubectl get ingress/ingress-2048 -n game-2048
```

delete sample app
```sh
kubectl delete -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.1.0/docs/examples/2048/2048_full.yaml
```