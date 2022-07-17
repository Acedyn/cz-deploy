# Cardboard Citizens deployement

Configuration files to deploy / build the Cardboard Citizens applications stack.
The stack is deployed as a kubernetes cluster on AWS using EKS.

## Setup

First, you need to install the AWS cli and connect to your IAM role. The
command differs from you OS/architecture, you can get the command on the
[official documentation](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

You will also need to install kubectl, same as aws cli installation, you can
get the installation command on the [official documentation](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)

Login to your IAM account using the aws CLI. It will prompt for you access key ID
and your secret acces key

```bash
aws configure
```

Then, you need to configure your kubeclt config to access the kubernetes cluster.

```bash
aws eks update-kubeconfig --region us-east-1 --name cz-cluster
```

To check if you have access to the cluster you can get the active pods in the
alpha namespace

```bash
kubectl get pods --namespace cz-alpha-stack
```

If you get an error saying you don't have the required access, login as an IAM user
that do have the required access (or ask for someone to do it) and edit the
``aws-auth`` configmap.

```bash
kubectl edit -n kube-system configmap/aws-auth
```

Add a user on the ``data.mapUsers`` field. The group will determine the right you
are providing. To provide full access user the ``system:serviceaccounts:dev`` group

```yaml
apiVersion: v1
data:
  mapUsers: |
    - groups:
        - system:serviceacounts:dev
      userrarn: <fing you user rarn in the IAM page>
      username: <same username as you IAM user>
```

## Apply deployement configs

```bash
kubectl apply -f deployment/alpha-deployment.yml
kubectl apply -f deployment/alpha-service.yml
```

The deployment will describe the various containers that will be deployed.
Kubernetes will make sure these requirement are always up.

The service describe the entrypoint of the cluster, it's like a reverse proxy.
