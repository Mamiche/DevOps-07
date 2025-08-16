# Déployer une application E-Commerce en architecture 3 tiers sur AWS EKS avec Helm

## Introduction

Dans le paysage dynamique du développement logiciel, les architectes et développeurs recherchent des modèles robustes assurant scalabilité, maintenabilité et efficacité. L’architecture 3 tiers divise une application en trois couches interconnectées, facilitant des applications évolutives et résilientes.

### Bases de l'architecture 3 tiers

1. **Couche de Présentation** :

   - Interaction avec les utilisateurs (UI, applications web/mobile).
   - Fournit une expérience utilisateur fluide.

2. **Couche Application (Logique Métier)** :

   - Traite les requêtes, applique les règles métier.
   - Sépare la logique métier de la présentation.

3. **Couche Données** :

   - Stocke et gère les données (bases de données, entrepôts).
   - Assure intégrité, sécurité et accès efficace aux données.

### Avantages

- **Scalabilité** : Chaque couche peut évoluer indépendamment.
- **Maintenabilité** : Modifications possibles sans impacter les autres couches.
- **Flexibilité** : Intégration de nouvelles technologies possible par couche.

## Prérequis

- **kubectl** : CLI pour Kubernetes. [Guide officiel](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)
- **eksctl** : CLI pour EKS. [Guide officiel](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html)
- **AWS CLI** : CLI pour AWS. [Guide officiel](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)
- Configurer AWS CLI avec `aws configure`. [Configuration rapide](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-config)

## Étapes

### Étape 1 : Créer un cluster EKS

```bash
eksctl create cluster \
  --name mon-cluster \
  --region ma-region \
  --nodegroup-name mon-nodegroup \
  --node-type t3.small \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3
```
![](https://imgur.com/Z1apZSY.png)

### Étape 2 : Configurer IAM OIDC

```bash
export cluster_name=mon-cluster
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
```

### Étape 3 : Configurer ALB Add-On

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json
eksctl create iamserviceaccount --cluster=mon-cluster --namespace=kube-system --name=aws-load-balancer-controller --role-name AmazonEKSLoadBalancerControllerRole --attach-policy-arn=arn:aws:iam::<AWS-ID>:policy/AWSLoadBalancerControllerIAMPolicy --approve
```
![](https://imgur.com/WDOzUzZ.png)
### Étape 4 : Déployer ALB Controller

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=mon-cluster --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller --set region=ma-region --set vpcId=<vpc-id>
kubectl get deployment -n kube-system aws-load-balancer-controller
```
![](https://imgur.com/ut7k9Eg.png)
### Étape 5 : Configurer EBS CSI Plugin

```bash
eksctl create iamserviceaccount --name ebs-csi-controller-sa --namespace kube-system --cluster mon-cluster --role-name AmazonEKS_EBS_CSI_DriverRole --role-only --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy --approve
eksctl create addon --name aws-ebs-csi-driver --cluster mon-cluster --service-account-role-arn arn:aws:iam::<AWS-ID>:role/AmazonEKS_EBS_CSI_DriverRole --force
```

### Étape 6 : Installer le Helm Chart

```bash
git clone https://github.com/uniquesreedhar/RobotShop-Project.git
cd RobotShop-Project/EKS/helm
kubectl create ns robot-shop
helm install robot-shop --namespace robot-shop .
kubectl get pods -n robot-shop
```

### Étape 7 : Créer Ingress

```bash
kubectl apply -f ingress.yaml
```

Accédez à l’application via le DNS fourni par le Load Balancer et testez toutes les fonctionnalités de l’e-commerce.

![](https://imgur.com/wFtDhLC.png)
