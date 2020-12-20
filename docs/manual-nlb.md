1. set environment variables for your cluster
```sh
CLUSTER_NAME=tor-k8s
FARGATE_PROFILE_NAME=tor-k8s-fargate-profile
AWS_REGION=ap-northeast-1
AWS_ACCOUNT_ID=$(aws sts get-caller-identity | jq -r '.Account')
```

Note: if your target environment you want to deploy is prod, you have to attach `--profile prod` option to all `aws` ans `eksctl` commands.

2. create an EKS cluster with managed nodegroup
```sh
eksctl create cluster \
    --name $CLUSTER_NAME
    --version 1.18 \
    --region $AWS_REGION
    --nodes 3 \
    --node-type t3.medium \
    --nodes-min 2 \
    --nodes-max 4 \
    --managed
```

3. set credential information to lcoal
```sh
eksctl utils write-kubeconfig --cluster $CLUSTER_NAME
```

4.
```sh
aws eks describe-cluster --name $CLUSTER_NAME --query "cluster.identity.oidc.issuer" --output text
```

5. 
```sh
eksctl utils associate-iam-oidc-provider \
    --region $AWS_REGION \
    --cluster $CLUSTER_NAME \
    --approve
```

6. 
```sh
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.1.0/docs/install/iam_policy.json
```

7. 
```sh
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

8. 
```sh
eksctl create iamserviceaccount \
  --cluster=$CLUSTER_NAME \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::$AWS_ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve
```

9. install ALB LoadBalancer Controller
```sh
curl -o crds.yaml https://raw.githubusercontent.com/aws/eks-charts/master/stable/aws-load-balancer-controller/crds/crds.yaml && \
kubectl apply -f crds.yaml
```

10. 
```sh
helm repo add eks https://aws.github.io/eks-charts
```

11. 
```sh
helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller \
  --set clusterName=$CLUSTER_NAME \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  -n kube-system
```

12. Deploy tor-proxy with NLB
```sh
kubectl apply -f tor-proxy-nlb-full.yaml
```

13. check `EC2` => `target group` => `target`
    Wait until all the status of all targets is healthy
