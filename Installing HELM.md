# Helm Install

Run Helm install script:

`curl https://raw.githubusercontent.com/helm/helm/master/scripts/get | bash`

Create a new ServiceAccount:  

`kubectl create serviceaccount --namespace kube-system tiller`

Create a ClusterRoleBinding for tiller:

`kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller`

```kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'```

Initialize HELM:

`helm init --service-account tiller`

in case you have already done helm init:

`helm init --service-account tiller --upgrade`

Add bitname repo:
```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install stable/heapster
```

Other commands:
```
helm del $(helm ls --all --short) --purge
helm create mychart
helm install --dry-run --debug ./mychart
helm install --dry-run --debug ./mychart --set service.internalPort=8080
helm install onerepo/onechart --name foobar
helm chart list
helm delete example
```

# HELM 3

```console
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTQ4NzE1MzA3M119
-->