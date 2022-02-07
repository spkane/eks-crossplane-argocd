# Crossplane in AWS

## Install (Imperative)

* Heavily inspired by this:
  * [Blog](https://aws.amazon.com/blogs/containers/gitops-model-for-provisioning-and-bootstrapping-amazon-eks-clusters-using-crossplane-and-argo-cd/)
  * [Github](https://github.com/aws-samples/eks-gitops-crossplane-argocd)

## Setup Management k8s Cluster

* It might be possible to install this on EKS Fargate, but we would need to double-check. Fargate nodes appear to be tainted by default.

```sh
eksctl create cluster --name xplane-mgmt --version 1.21
```

* And ~23 minutes later...

## Setup Crossplane

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

* Install the aws-provider

```sh
kubectl crossplane install provider crossplane/provider-aws:v0.23.0
```

* There are a few options for Auth, we are just passing in credentials for this test.

* Create a file called `secret-aws-creds` with the AWS credentials in it, in this format:

```ini
[default]
aws_access_key_id =ABCDEFGHIJ0123456789
aws_secret_access_key = Ow3HUaP8BbqkV4dUrZr0H7yT5nGP5OPFcZJ+
```

* Create a k8s secret in the cluster that contains AWS credentials from the file.

```sh
kubectl create secret generic aws-credentials --namespace crossplane-system --from-file=credentials=./secret-aws-creds
```

* Create `xplane-aws-provider-creds.yaml`

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
cd ../..
```

* Provide reasonable IAM roles in the `eks-cluster-xr.yaml` file and then apply it.
  * `arn:aws:iam::aws:policy/AmazonEKSClusterPolicy` is likely the minimum.

## Setup Managed Resources

```sh
kubectl apply -f ./eks-cluster-xr.yaml
```

* And ~23 minutes later, check to see if the EKS cluster is ready.

```sh
# Get all resources that represent a unit of external infrastructure such as the EKS cluster.
kubectl get managed
```

* Other useful commands:

```sh
kubectl get crossplane  # get all resources related to Crossplane.
kubectl get composite   # get all resources that represent an XR
```

## Cleanup (Imperative)

### Delete Managed Resources

```sh
kubectl delete -f ./eks-cluster-xr.yaml
```

### Delete Crossplane

* Delete the configuration(s) returned by:

```sh
kubectl get configurations
```

* and then uninstall everything else.

```sh
kubectl delete providerconfig.aws.crossplane.io/default
kubectl delete -f ./xplane-aws-provider-creds.yaml
kubectl delete -n crossplane-system secret aws-credentials
helm uninstall crossplane --namespace crossplane-system
```

### Delete Management k8s Cluster

```sh
eksctl delete cluster --name=xplane-mgmt
```

## Install (Declarative - Argo)

## Standup Management k8s Cluster

* It might be possible to install this on EKS Fargate, but we would need to double-check. Fargate nodes appear to be tainted by default.

```sh
eksctl create cluster --name xplane-mgmt --version 1.21
```

* And ~23 minutes later...

## Standup Argo CD (non-HA)

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
# argocd app create --file argo-application-crossplane.yaml
kubectl apply -f ./argo-application-crossplane.yaml
# Wait for things to stabilize...
```
* Want to look at the UI?

```sh
kubectl port-forward -n argocd service/argocd-server 8443:443
```

* Then open up a web browser pointing to [https://127.0.0.1:443/](https://127.0.0.1:443/) and login with the credentials you set earlier.
* [Install](https://github.com/bitnami-labs/sealed-secrets/releases) the `kubeseal` CLI
* Prepare real secrets for the AWS provider

```sh
cd argo-apps/crossplane-complete/examples
cp 3-aws-credentials-unsealed-unencrypted secret-3-aws-credentials-unsealed-unencrypted
```

* Create a file called `secret-aws-creds` with the AWS credentials in it, in this format:

```ini
[default]
aws_access_key_id =ABCDEFGHIJ0123456789
aws_secret_access_key = Ow3HUaP8BbqkV4dUrZr0H7yT5nGP5OPFcZJ+
```

* Get the secret into a Base64 encoded string.

```sh
cat ./secret-aws-creds | base64
```

* Add the base64 encoded string to the file `secret-3-aws-credentials-unsealed-unencrypted`
* Seal the secret

```sh
kubeseal --format yaml --controller-namespace=sealed-secrets --controller-name=sealed-secrets-controller < secret-3-aws-credentials-unsealed-unencrypted > ../templates/3-aws-credentials-sealed.yaml
cd ../../..
```

* This is gitops, so push the changes.

```sh
git add .
git commit -m "update"
git push origin master
# Wait for things to stabilize...
```

* Register a remote k8s server with Argo and then deploy some apps to it (or we can deploy them to the mangement cluster if that is the goal).

```sh
#argocd app create --file application-apps.yaml
kubectl apply -f ./argo-apps-of-apps.yaml
```
