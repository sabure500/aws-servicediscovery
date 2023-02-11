# aws-servicediscovery

## 実行コマンド

aws cloudformation deploy --template-file servicediscovery-network.yml --stack-name servicediscovery-network
aws cloudformation deploy --capabilities CAPABILITY_NAMED_IAM --template-file servicediscovery-eks-controlplane.yml --stack-name servicediscovery-eks-controlplane
aws cloudformation deploy --capabilities CAPABILITY_NAMED_IAM --template-file servicediscovery-eks-node.yml --stack-name servicediscovery-eks-node
aws cloudformation deploy --template-file servicediscovery-cloud9.yml --stack-name servicediscovery-cloud9
aws cloudformation deploy --template-file servicediscovery-cloudmap.yml --stack-name servicediscovery-cloudmap

### Cloud9

```bash
# kubectlの取得
## https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/install-kubectl.html
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.24.9/2023-01-11/bin/linux/amd64/kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
kubectl version --short --client

# EKSへの接続と検証
aws eks --region ap-northeast-1 update-kubeconfig --name servicediscovery-test-EKSCluster
kubectl get pods --all-namespaces
## サンプルアプリ作成(https://github.com/kubernetes/examples/tree/master/guestbook-go)
kubectl create namespace sample-app
kubectl apply -f https://raw.githubusercontent.com/kubernetes/examples/master/guestbook-go/redis-master-controller.yaml -n sample-app
kubectl apply -f https://raw.githubusercontent.com/kubernetes/examples/master/guestbook-go/redis-master-service.yaml -n sample-app
kubectl apply -f https://raw.githubusercontent.com/kubernetes/examples/master/guestbook-go/redis-replica-controller.yaml -n sample-app
kubectl apply -f https://raw.githubusercontent.com/kubernetes/examples/master/guestbook-go/redis-replica-service.yaml -n sample-app
kubectl apply -f https://raw.githubusercontent.com/kubernetes/examples/master/guestbook-go/guestbook-controller.yaml -n sample-app
kubectl apply -f https://raw.githubusercontent.com/kubernetes/examples/master/guestbook-go/guestbook-service.yaml -n sample-app
kubectl get all -n sample-app
kubectl get svc -o wide -n sample-app
kubectl delete namespace sample-app
```

```bash
# Appmesh導入
helm repo add eks https://aws.github.io/eks-charts
kubectl create ns appmesh-system
kubectl apply -k "https://github.com/aws/eks-charts/stable/appmesh-controller/crds?ref=master"
set oidc_id $(aws eks describe-cluster --name servicediscovery-test-EKSCluster --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
eksctl utils associate-iam-oidc-provider --cluster servicediscovery-test-EKSCluster --approve

eksctl create iamserviceaccount \
    --cluster servicediscovery-test-EKSCluster \
    --namespace appmesh-system \
    --name appmesh-controller \
    --attach-policy-arn  arn:aws:iam::aws:policy/AWSCloudMapFullAccess,arn:aws:iam::aws:policy/AWSAppMeshFullAccess \
    --override-existing-serviceaccounts \
    --approve

helm upgrade -i appmesh-controller eks/appmesh-controller \
    --namespace appmesh-system \
    --set region=ap-northeast-1 \
    --set serviceAccount.create=false \
    --set serviceAccount.name=appmesh-controller
```

```bash
# Appmeshリソース作成
kubectl apply -f appmesh/namespace.yaml
kubectl apply -f appmesh/mesh.yaml
kubectl apply -f appmesh/virtual-node.yaml
kubectl apply -f appmesh/virtual-router.yaml
kubectl apply -f appmesh/virtual-service.yaml
```

```bash
# appmeshサービス作成
aws iam create-policy --policy-name my-appmesh-policy --policy-document file://tmp/proxy-auth.json
eksctl create iamserviceaccount \
    --cluster servicediscovery-test-EKSCluster \
    --namespace my-apps \
    --name my-service-a \
    --attach-policy-arn arn:aws:iam::277589744906:policy/my-appmesh-policy \
    --override-existing-serviceaccounts \
    --approve
kubectl apply -f appmesh/example-service.yaml
kubectl -n my-apps get pods -o wide
kubectl -n my-apps get svc -o wide
```

## 削除コマンド

```bash
eksctl delete iamserviceaccount --name my-service-a --namespace my-apps --cluster servicediscovery-test-EKSCluster
eksctl delete iamserviceaccount --name appmesh-controller --namespace appmesh-system --cluster servicediscovery-test-EKSCluster
kubectl delete namespace my-apps
# kubectl delete VirtualService/my-service-a -n my-apps
# kubectl delete VirtualRouter/my-service-a-virtual-router -n my-apps
# kubectl delete VirtualNode/my-service-a -n my-apps
kubectl delete mesh my-mesh
helm delete appmesh-controller -n appmesh-system
kubectl delete ns appmesh-system
```

aws cloudformation delete-stack --stack-name servicediscovery-cloudmap
aws cloudformation delete-stack --stack-name servicediscovery-cloud9
aws cloudformation delete-stack --stack-name servicediscovery-eks-node

aws cloudformation delete-stack --stack-name servicediscovery-eks-controlplane
aws cloudformation delete-stack --stack-name servicediscovery-network

## 削除時確認 URL

コードではなく CLI で作成したため、CloudFormation のスタック削除後に確認が必要なもの  
もしくは EKS 経由で作成したため正常に削除されているか念のため確認した方がよいもの

- OIDC プロバイダー()
  https://us-east-1.console.aws.amazon.com/iamv2/home?region=ap-northeast-1#/identity_providers
- IAM ロール(eksctl-servicediscovery-test-EKSCluster-addo-Role\*)
  https://us-east-1.console.aws.amazon.com/iamv2/home?region=ap-northeast-1#/roles
- Mesh
  https://ap-northeast-1.console.aws.amazon.com/appmesh/meshes?region=ap-northeast-1
- IAM ポリシー(my-appmesh-policy)
  https://us-east-1.console.aws.amazon.com/iamv2/home?region=ap-northeast-1#/policies
- ELB
  https://ap-northeast-1.console.aws.amazon.com/ec2/home?region=ap-northeast-1#LoadBalancers:sort=loadBalancerName

```

```
