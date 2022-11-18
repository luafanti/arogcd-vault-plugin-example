# ArgoCD Secret Management with the argocd-vault-plugin example

The [argocd-vault-plugin](https://argocd-vault-plugin.readthedocs.io/en/stable/) is a ArgoCD plugin for retrieving secrets from HashiCorp Vault and injecting them into Kubernetes YAML files.
This repo contains samples how to install plugin and inject secrets to kubernetes resources.

## Prerequisites

Instruction assumes you have kubernetes cluster and helm installed. Cluster can be local one like minikube or k3s. 

#### Vault setup

If you don't have Vault installed on k8s you can check my another [repo](https://github.com/luafanti/vault-terraform-example) with instruction how to setup Vault and resources with terraform

#### Argo CD setup

Easiest way to install ArgoCD is using Helm chart. Install it with helm in dedicated namespace

```
kubectl create ns argocd
kubectl config set-context --current --namespace=argocd

helm repo add argo https://argoproj.github.io/argo-helm
helm install argo-cd argo/argo-cd
```

#### Argo CD Vault Plugin installation

You can install `argocd-vault-plugin` on two different ways:

* Installation via `argocd-cm` ConfigMap
* Installation via a sidecar container 

Let's focus here on installation with `argocd-cm`
To install plugin we need to create & update few Argo kubernetes resources create by Helm chat.

For `argocd-cm-installation/argocd-vault-plugin-credentials.yaml` you have to provide your Vault `approle` credentials and IP address of Vault instance in k8s.
```
kubectl apply -f argocd-cm-installation/argocd-vault-plugin-credentials.yaml

kubectl delete cm argocd-cm
kubectl apply -f argocd-cm-installation/argocd-cm.yaml

kubectl delete deployments.apps argo-cd-argocd-repo-server
kubectl apply -f argocd-cm-installation/argocd-repo-server.yaml
```

Instead, deleting and applying new YAML's you can also edit manifest comparing diffs under `argocd-cm-installation/`.
Now `argocd-vault-plugin` has been installed. 


## Demo project setup

You can create Argo CD app in UI or with CLI as below:

```
# create dedicated namespace for demo resources
kubectl create ns argocd-vault-demo

# expose Argo server locally
kubectl port-forward svc/argo-cd-argocd-server 8080:80  -n argocd

# authroize Argo CD CLI
argocd login localhost:8080 --username admin --password $(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)

# create Argo CD app with demo k8s resources
argocd app create argocd-vault-plugin-demo --repo https://github.com/luafanti/arogcd-vault-plugin-example.git --path kubernetes-resources --dest-namespace argocd-vault-demo --dest-server https://kubernetes.default.svc --config-management-plugin argocd-vault-plugin

# sync Argo demo app
argocd app sync argocd-vault-plugin-demo
```

Now you should see in `argocd-vault-demo` namespace two deployments and one secret created and controlled by Argo CD. 
One of the great things about the plugin is that if the value changes in Vault Argo will notice these changes and display `OutOfSync` status.

```
# update kv in Vault
vault kv put dog-kv-v2/credentials password=dog-pass user=dog_v10

# refresh application data as well as target manifests cache
argocd app get argocd-vault-plugin-demo --hard-refresh

# sync Argo demo app again
argocd app sync argocd-vault-plugin-demo
```

## Notes

* `argocd-vault-demo` support only kv-v2 and kv-v2 secret engines. Database secret engines is in-progress. https://github.com/argoproj-labs/argocd-vault-plugin/issues/270
* Installing plugins via `argocd-cm` ConfigMap is deprecated in Argo CD v2.5. Starting in Argo CD v2.6 it will be no longer supported method of installation.
