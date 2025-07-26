# Gitlab lab setup

## Install the tools

- Kind (To create local kubernetes cluster)
- Kubectl (To interact with Kubernetes)
- Helm (To install Gitlab on Kubernetes)
- Docker (To run containers)

On Arch Linux:
```shell
paru -S helm kind kubectl docker
```

## Setup

We will install Gitlab on Kubernetes using `Helm`. [Helm](https://helm.sh/) is a great tool to package and install software on Kubernetes.
It's a template engine that renders the defined Kubernetes resources and apply them to Kubernetes.
You can upgrade and rollback versions easily using it.

Create the Kubernetes cluster using `Kind`:
```shell
kind create
```

Add the `Gitlab` Helm chart repository:
```shell
helm repo add gitlab http://charts.gitlab.io/
```

Install the `Gitlab` using Helm on Kubernetes:
```shell
helm install my-gitlab gitlab/gitlab --version 9.2.1 --values ./setup/values.yml
```

Using Kubectl you can view the pods created on Kubernetes and their status:
```shell
kubectl get pods
```

In few minutes the pods should be `running`.

Execute kubectl `port-forward` to create access Gitlab from your machine:
```shell
sudo kubectl --kubeconfig="$HOME/.kube/config" port-forward svc/my-gitlab-webservice-default 80:8181
```

Add `gitlab.local` as 127.0.0.1 to `/etc/hosts`:
```shell
echo "127.0.0.1 gitlab.local" >> /etc/hosts
```

You can access Gitlab on http://gitlab.local:80 using a Browser

Run the following command to get the user `root` password to login on Gitlab.
```shell
kubectl get secret -o json my-gitlab-gitlab-initial-root-password | jq -r '.data.password' | base64 -d
```

Login on Gitlab using the `root` user.

### Git SSH

You can also `port-forward` the Gitlab Shell port to use Git SSH:
```shell
sudo kubectl --kubeconfig="$HOME/.kube/config" port-forward svc/my-gitlab-gitlab-shell 22:22
```

## Cleanup

Remove the Kubernetes cluster:
```shell
kind delete
```

## Further reading

- [Kind Cheatsheet](https://www.hackingnote.com/en/cheatsheets/kind/)
- [Helm](https://helm.sh/)
