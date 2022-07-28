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

If you are deploying the stack for the first time on the cluster you need to create
the namespaces first

```bash
kubectl create namespace cz-alpha-stack
kubectl create namespace cz-beta-stack
kubectl create namespace cz-prod-stack
```

Otherwise you can get the pods in an existing stack to make sure you have access
to the cluster

```bash
kubectl get pods --namespace cz-alpha-stack
```

If you get an error saying you don't have the required access rights, login as
an IAM user that do have the required access (or ask for someone to do it)
and edit the ``aws-auth`` configmap.

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

## Apply deployement/service configs

```bash
kubectl apply -f ressource/alpha/front.yml -n cz-alpha-stack
```

The deployment will describe the various containers that will be deployed.
Kubernetes will make sure these requirement are always up. This command
can be used to create or update a deployment.

The service describe the entrypoint of the cluster, it's like a reverse proxy.

## First configuration

If you are deploying the cluster for the first time, you will need to setup:

- [deploy the secret ressources](https://kubernetes.io/fr/docs/concepts/configuration/secret/)
- [nginx ingress controller](https://docs.nginx.com/nginx-ingress-controller)
This will deploy a reverse proxy pod and a load balancer service
- [cloudflaire dns records](https://developers.cloudflare.com/dns/manage-dns-records/how-to/create-dns-records/)
In order to use the cardboardcitizen.com domain name, you will configure the redirection
- [ssl certificate with aws certificate manager](https://docs.aws.amazon.com/acm/latest/userguide/acm-overview.html)
This will create an ssl certificate that will be renewed automatically

### Deploy the secrets

Some variables are needed by the deployed pods but must be hidden by others for
security reasons. Before doing anything, if you are setting up this cluster for the
first time, make sure to create the required secrets.

Copy the ``.env.example`` file as ``.env`` and update the variables with your secret
values then run

```bash
kubectl create secret generic cz-secret --from-env-file ./.env -n cz-alpha-stack
```

### Install the nginx ingress controller

The installation of the nginx controller is done using [helm](https://helm.sh/docs/intro/install/)
Helm is like a package manager that bundles kubernetes configurations files
into what they call ``charts``. We could install the nginx ingress controller without
helm, but it just makes it simpler.

First add the nginx repository:

```bash
helm repo add nginx-stable https://helm.nginx.com/stable
```

Install the chart

```bash
helm install my-release nginx-stable/nginx-ingress -n cz-alpha-stack
```

Normally you will see a pod and a load balancer service have been deployed

```bash
kubectl get pod -n cz-alpha-stack
```

The configuration of the reverse proxy, you can edit the ``ingress.yml`` file.
The nginx pod will look at the ingress with the ``ingressClassName`` set to nginx
and update its nginx.conf automatically. The ingress config acts as an abtraction
for any reverse proxy services.

### Create the DNS records on cloudflaire

If you didn't already make sure you are using cloudflaire's DNS server. If you
bought your domain on something like OVH, the domain will likely be handled by
and OVH dns server. Add a website passing your domain name, you then be able to
change the DNS using the urls provided by cloudflaire to allow it to take the
lead on redirections (it might take a day for the transition to be complete).

You will need to get the external IP of the nginx load balancer that you created
in the previous step:
(it should looks like [garbage].amazon.aws.com)

```bash
kubectl get service -n cz-alpha-stack
```

Once ready, go to the DNS page on cloudflaire, remove all garbage records and
create the following:

- Type: CNAME, Name: @, Content: [The DNS name of the loadbalancer]
- Type: CNAME, Name: *, Content: [The DNS name of the loadbalancer]

The first one is redirecting the root to the load balancer, and the second one
is redirecting all subdomain to that same load balancer.

The reason why we are doing that is because all the redirection is actually
handled by the nginx reverse proxy that we just created. So the configuration
is made in the ingress.yml file.

### Get an ssl certificate using aws

Go to the [AWS certificate manager page](https://console.aws.amazon.com/acm/)
and ask for a public certificate. Set the domain name to alphatestingcz.xyz and
leave everything to default.

Once the certificate has been created, you will need to validate the certificate.
To do so, go the the details of the certificate and get the ``CNAME name`` and the
``CNAME value`` (the name should look like [garbage].alphatesting.xyz and
[garbage].acm-validation.aws). To prove that you are the owner of the domain, you
will have to go to the cloudflaire DNS and create a CNAME entry with the name and
the value you just got (since cloudflaire is a CDN, make sure to disable the proxy
options otherwise it won't work)
You can  then wait up to 10 minutes for the domain name to be validated on AWS.

Now you need to tell to the nginx load balancer to use that certificate. Go to the
load balancer section of the AWS console and update the listeners. There should
already have two listeners listenning on TCP. Update them both, one to HTTP and
one to HTTPS. When chaging the protocol it will change the instance port, but you
need to keep the port that was set ! If you don't know which port it was set to
you can always check the kubernetes service configuration using:

```bash
kubectl get service -n cz-alpha-stack
```

And check the port mapping on the nginx service.
For the HTTPS listener, select your ACM SSL certificate, and whatever encryption
algorythm.
