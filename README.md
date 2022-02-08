# Crossplane in AWS

- [Crossplane in AWS](#crossplane-in-aws)
  - [Imperative](#imperative)
    - [Install (Imperative)](#install-imperative)
      - [Setup Management k8s Cluster](#setup-management-k8s-cluster)
      - [Setup Crossplane](#setup-crossplane)
      - [Setup Managed Resources](#setup-managed-resources)
    - [Cleanup (Imperative)](#cleanup-imperative)
      - [Delete Managed Resources](#delete-managed-resources)
      - [Delete Crossplane](#delete-crossplane)
      - [Delete Management k8s Cluster](#delete-management-k8s-cluster)
  - [Declarative (Argo)](#declarative-argo)
    - [Install (Declarative)](#install-declarative)
      - [Standup Management k8s Cluster](#standup-management-k8s-cluster)
      - [Standup Argo CD (non-HA)](#standup-argo-cd-non-ha)
      - [Standup Crossplane via Argo](#standup-crossplane-via-argo)
      - [Create an EKS cluster with Argo & Crossplane](#create-an-eks-cluster-with-argo--crossplane)
      - [Spin up the EKS cluster](#spin-up-the-eks-cluster)
      - [Add the Cluster to Argo CD](#add-the-cluster-to-argo-cd)
      - [Deploy apps to our cluster(s) via Argo](#deploy-apps-to-our-clusters-via-argo)
    - [Cleanup (Declarative)](#cleanup-declarative)
  - [Other Notes](#other-notes)

- Heavily inspired by this:
  - [Blog](https://aws.amazon.com/blogs/containers/gitops-model-for-provisioning-and-bootstrapping-amazon-eks-clusters-using-crossplane-and-argo-cd/)
  - [Github](https://github.com/aws-samples/eks-gitops-crossplane-argocd)

## Imperative

### Install (Imperative)

#### Setup Management k8s Cluster

- It might be possible to install this on EKS Fargate, but we would need to double-check. Fargate nodes appear to be tainted by default.

```sh
eksctl create cluster --name xplane-mgmt --version 1.21
```

- And ~23 minutes later...

#### Setup Crossplane

```sh
aws eks update-kubeconfig --name xplane-mgmt
#
curl -sL https://raw.githubusercontent.com/crossplane/crossplane/v1.6.2/install.sh | sh
sudo mv kubectl-crossplane /usr/local/bin
kubectl crossplane --help
#
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
#
kubectl create namespace crossplane-system
helm install crossplane --namespace crossplane-system crossplane-stable/crossplane --version 1.6.2
```

- Install the aws-provider

```sh
kubectl crossplane install provider crossplane/provider-aws:v0.23.0
```

- There are a few options for Auth, we are just passing in credentials for this test.

- Create a file called `secret-aws-creds` with the AWS credentials in it, in this format:

```ini
[default]
aws_access_key_id =ABCDEFGHIJ0123456789
aws_secret_access_key = Ow3HUaP8BbqkV4dUrZr0H7yT5nGP5OPFcZJ+
```

- Create a k8s secret in the cluster that contains AWS credentials from the file.

```sh
kubectl create secret generic aws-credentials --namespace crossplane-system --from-file=credentials=./secret-aws-creds
```

- Create `xplane-aws-provider-creds.yaml`

```yaml
apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-credentials
      key: credentials
```

```sh
cd crossplane-xrds/eks-configuration
rm *.xpkg
kubectl crossplane build configuration
kubectl crossplane push configuration ${IMAGE_REPO}:${IMAGE_TAG}
kubectl crossplane install configuration ${IMAGE_REPO}:${IMAGE_TAG}
#kubectl crossplane install configuration ./*.xpkg
#kubectl crossplane install configuration registry.upbound.io/upbound/platform-ref-aws:v0.2.2
cd ../..
```

#### Setup Managed Resources

- Provide reasonable IAM roles in the `eks-cluster-xr.yaml` file and then apply it.
  - The role likely needs at least these permissions: `arn:aws:iam::aws:policy/AmazonEKSClusterPolicy`, `arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy`, `arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy`, `arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly`
  - and these service principals: `ec2.amazonaws.com`, `eks.amazonaws.com`

```sh
kubectl apply -f ./eks-cluster-xr.yaml
```

- And ~23 minutes later, check to see if the EKS cluster is ready.

```sh
# Get all resources that represent a unit of external infrastructure such as the EKS cluster.
kubectl get managed
```

  -Other useful commands:

```sh
kubectl get crossplane  # get all resources related to Crossplane.
kubectl get composite   # get all resources that represent an XR
kubectl get claim
kubectl get configuration.pkg
```

### Cleanup (Imperative)

#### Delete Managed Resources

```sh
kubectl delete -f ./eks-cluster-xr.yaml
```

#### Delete Crossplane

- Delete the configuration(s) returned by:

```sh
kubectl get configurations
```

- and then uninstall everything else.

```sh
kubectl delete providerconfig.aws.crossplane.io/default
kubectl delete -f ./xplane-aws-provider-creds.yaml
kubectl delete -n crossplane-system secret aws-credentials
helm uninstall crossplane --namespace crossplane-system
```

#### Delete Management k8s Cluster

```sh
eksctl delete cluster --name=xplane-mgmt
```

## Declarative (Argo)

### Install (Declarative)

#### Standup Management k8s Cluster

- It might be possible to install this on EKS Fargate, but we would need to double-check. Fargate nodes appear to be tainted by default.

```sh
eksctl create cluster --name xplane-mgmt --version 1.21
```

- And ~23 minutes later...

#### Standup Argo CD (non-HA)

```sh
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
# Install the `argocd `CLI via https://argo-cd.readthedocs.io/en/stable/cli_installation/
kubectl port-forward -n argocd service/argocd-server 8080:80
# External API Access could be achieved with: kubectl -n argocd patch svc argocd-server -p '{"spec": {"type": "LoadBalancer"}}'
kubectl -n argocd get secret argocd-initial-admin-secret --template={{.data.password}} | base64 -D; echo
argocd login localhost:8443
argocd account update-password
kubectl -n argocd delete secret argocd-initial-admin-secret
kubectl apply -f ./argo-app-projects.yaml
```

- Want to look at the UI?

```sh
kubectl port-forward -n argocd service/argocd-server 8443:443
```

- Then open up a web browser pointing to [https://127.0.0.1:443/](https://127.0.0.1:443/) and login with the credentials you set earlier.

#### Standup Crossplane via Argo

```sh
# argocd app create --file argo-application-crossplane.yaml
kubectl apply -f ./argo-application-crossplane.yaml
# Wait for things to stabilize...
```

- [Install](https://github.com/bitnami-labs/sealed-secrets/releases) the `kubeseal` CLI
- Prepare real secrets for the AWS provider

```sh
cd argo-apps/crossplane-complete/examples
cp 3-aws-credentials-unsealed-unencrypted secret-3-aws-credentials-unsealed-unencrypted
```

- Create a file called `secret-aws-creds` with the AWS credentials in it, in this format:

```ini
[default]
aws_access_key_id =ABCDEFGHIJ0123456789
aws_secret_access_key = Ow3HUaP8BbqkV4dUrZr0H7yT5nGP5OPFcZJ+
```

- Get the secret into a Base64 encoded string.

```sh
cat ./secret-aws-creds | base64
```

- Add the base64 encoded string to the file `secret-3-aws-credentials-unsealed-unencrypted`
- Seal the secret

```sh
kubeseal --format yaml --controller-namespace=sealed-secrets --controller-name=sealed-secrets-controller < secret-3-aws-credentials-unsealed-unencrypted > ../templates/3-aws-credentials-sealed.yaml
cd ../../..
```

- This is gitops, so push the changes.

```sh
git add .
git commit -m "update"
git push origin master
# Wait for things to stabilize...
```

#### Create an EKS cluster with Argo & Crossplane

- Build and install a Crossplane XRD for our EKS cluster

```sh
cd crossplane-xrds/eks-configuration
rm *.xpkg
kubectl crossplane build configuration
kubectl crossplane push configuration ${IMAGE_REPO}:${IMAGE_TAG}
kubectl crossplane install configuration ${IMAGE_REPO}:${IMAGE_TAG}
#kubectl crossplane install configuration ./*.xpkg
#kubectl crossplane install configuration registry.upbound.io/upbound/platform-ref-aws:v0.2.2
cd ../..
```

#### Spin up the EKS cluster

- Provide reasonable IAM roles in the `eks-cluster-xr.yaml` file and then apply it.
  - The role likely needs at least these permissions: `arn:aws:iam::aws:policy/AmazonEKSClusterPolicy`, `arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy`, `arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly`
  - and these service principals: `ec2.amazonaws.com`, `eks.amazonaws.com`

```sh
kubectl apply -f ./argo-eks-cluster.yaml
```

- And ~23 minutes later, check to see if the EKS cluster is ready.

```sh
# Get all resources that represent a unit of external infrastructure such as the EKS cluster.
kubectl get managed
```

- Other useful commands:

```sh
kubectl get crossplane  # get all resources related to Crossplane.
kubectl get composite   # get all resources that represent an XR
kubectl get claim
kubectl get configuration.pkg
```

- Get new cluster name and update it's context:

```sh
aws eks --region us-west-2 list-clusters
aws --region us-west-2 eks update-kubeconfig --name crossplane-prod-cluster-${CLUSTER_ID}
# Switch our context back to the Argo cluster
aws --region us-west-2 eks update-kubeconfig --name xplane-mgmt
```

#### Add the Cluster to Argo CD

- Add cluster to ArgoCD

```sh
argocd login localhost:8443
argocd cluster add ${NEW_CLUSTER_CONTEXT} --name eks.cluster
```

#### Deploy apps to our cluster(s) via Argo

```sh
#argocd app create --file application-apps.yaml
kubectl apply -f ./argo-apps-of-apps.yaml
# Wait for things to stabilize...
```

- Explore the apps

```sh
# Quantum Game v1
kubectl port-forward -n nodejs service/quantum-game-svc 8080:8080
```

- Then open up a web browser pointing to [https://127.0.0.1:8080/](https://127.0.0.1:8080/)

```sh
# Grafana UI
kubectl port-forward -n monitoring service/observability-grafana 3000:80
```

- Then open up a web browser pointing to [https://127.0.0.1:3000/](https://127.0.0.1:3000/)

### Cleanup (Declarative)

...
## Other Notes

- [crossplane-contrib/provider-argocd](https://github.com/crossplane-contrib/provider-argocd)
- [crossplane-contrib/provider-terraform](https://github.com/crossplane-contrib/provider-terraform)
  - [crossplane/terrajet](https://github.com/crossplane/terrajet)
- [crossplane-contrib/provider-kubernetes](https://github.com/crossplane-contrib/provider-kubernetes)
- [crossplane-contrib/provider-helm](https://github.com/crossplane-contrib/provider-helm)
- [crossplane-contrib/provider-github](https://github.com/crossplane-contrib/provider-github)
