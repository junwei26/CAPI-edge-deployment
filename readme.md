# Pre-requisitcs

1\) For this operation you would need to install the following in your command line tool
- [`kubectl`](https://kubernetes.io/docs/tasks/tools/)
- [`argocd`](https://argo-cd.readthedocs.io/en/stable/cli_installation/)
- [`clusterctl`](https://cluster-api.sigs.k8s.io/user/quick-start.html)

2.1) You should also download the kubeconfig file from Prism Central as shown below

![Prism Central Interface](/media/download_kubeconfig.png)

2.2) Rename the file to `config` and move it to `~/.kube`. Below is an example for MacOs users.
```bash
mv ~/Downloads/config ~/.kube
```

2.3) To ensure that you are connected to the cluster correctly you can simply enter `kubectl get nodes` to get a response. If you get the following error, message please check your steps.

```bash
E0801 19:26:28.316006   67075 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": context deadline exceeded - error from a previous attempt: read tcp [::1]:52
```

3\) You need to have an additional terminal to forward the argocd server to your localhost to log in later.

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

# Setting up cluster

1\) In a new terminal use the following comand to log into your argocd account

```bash
argocd login localhost:8080
```

2\) Create a new namespace for your cluster

```bash
kubectl create namespace <new-namespace>
```

3\) Create the an Argo CD Application

```bash
argocd app create <argocd-application-name> --repo https://github.com/junwei26/CAPI-edge-deployment.git --dest-namespace default --dest-server https://kubernetes.default.svc --path ./
```
4\) Since the github repo only has the template, you need to do the following to set up your own values

```bash
argocd app set <argocd-application-name> --values values.yaml

argocd app set <argocd-application-name> -p namespace=<new-namespace> -p nke_cluster_name=<k8s-cluster-name> -p control_plane_endpoint=<control-place-endpoint>

argocd app set <argocd-application-name> -p user=<user-name> -p password=<password> -p ssh_pub_key=<public-key>

argocd app sync <argocd-application-name>
```

5\) At this stage, you should have most things set up correctly and you can run the following to get a status report. Not everything will be running imeddiately because we have not set up the network.

```bash
clusterctl describe --show-conditions all cluster <k8s-cluster-name> -n <new-namespace>
```

This is what the status will look like

![Step 5 Status](/media/step5-status.png)


6\) Finally you can set up the internal networking. As an example, we use `cilium`

```bash
argocd app create <cilium-application-name> --repo https://helm.cilium.io/ --helm-chart cilium --revision 1.14.0 --dest-namespace kube-system --dest-server https://<control-place-endpoint>:6443 # 6443 is the default port for control plane

argocd app sync <cilium-application-name>
```

7\) Monitor the setting up progress by running the following code repeatedly

```bash
clusterctl describe --show-conditions all cluster <k8s-cluster-name> -n <new-namespace>
```

When everything is ready, the ready state should be `True` for everything.

![Final Status](/media/final-status.png)
