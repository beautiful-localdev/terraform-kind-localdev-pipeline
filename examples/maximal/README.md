# Maximal Example

This example provisions a KIND cluster and deployws:

* cert-manager
* ingress-nginx
* argocd
* gitea
* sealed secrets
* docker registry (as separate container)

For convenience, we also write useful items like credentials to a local env file at './.pipeline/pipeline.env'.

## One-time Setup Steps

### Set up minimal DNS for a custom localdev domain
It's often very useful to have domains other than localhost resolve to 127.0.0.1 while we're working with our cluster. This is optional, a domain can be provided by another mechanism and set with the kind_cluster_local_domain variable in terraform. For use cases where local name resolution beyond localhost isn't required, we can ignore the kind_cluster_local_domain variable and use kubectl to port-forward as necessary to reach resources in the cluster.

Here we'll set up a "stub" domain `localdev` with dnsmasq and point it to 127.0.0.1. This will allow us to use subdomains of `localdev` as ingress hosts for services in our cluster without needing to do anything tedious like manage individual /etc/hosts entries. Because the default value of `kind_cluster_local_domain` is `localdev` we will not need to take any further action to make our friendly domains work automatically.

The instructions below work on MacOS using Homebrew.

```
brew install dnsmasq
mkdir -p $(brew --prefix)/etc/
echo 'address=/.localdev/127.0.0.1' >> $(brew --prefix)/etc/dnsmasq.conf
sudo brew services start dnsmasq

# Test
dig arbitrary-subdomain.localdev @127.0.0.1
```

### Set up a trusted local CA 

The terraform-kind-localdev-pipeline module includes a CA ClusterIssuer for cert-manager to facilitate smooth interactions between localhost and services running within the cluster using trusted self-signed certificates. If we do not supply a certificate and key, they will be automatically generated for use by cert-manager. Any CA Certificate trusted by localhost can be supplied, or if they are omitted the generated certificate can then be trusted on localhost by whatever mechanism is preferred. In this example, we will use [mkcert](https://github.com/FiloSottile/mkcert) to easily generate and trust a local CA certificate. 

```
mkdir -p ~/.mkcert/localdev
CAROOT=~/.mkcert/localdev mkcert --install
```
Enter your password when prompted (and prompted again). Output should look like this:
```
# Example output
# Created a new local CA 💥
# Sudo password:
# The local CA is now installed in the system trust store! ⚡️

```
This will create `rootCA-key.pem' and `rootCA.pem` at `~/.mkcert/localdev`. You can copy their contents into `ca_cert` and `ca_key` variables in a terraform.tfvars file to set the cert and private key variables for the CA ClusterIssuer, but for the sake of this ephemeral example let's just use the `-var` argument to terraform to set them at the CLI.

## Apply our terraform

If you are using your own CA cert and key rather than the ones we generated with mkcert above, update the paths in our terraform apply var arguments below to reflect the paths to your cert and key. If you'd prefer not to trust the CA locally, you can omit both var arguments to generate a CA cert and key with terraform. Note that unless the generated cert and key are then installed into the local trust store(s), portions of this example may not perform as expected.

```
# Start in this directory
cd examples/maximal

# Run terraform apply and supply the contents of our mkcert CA certificate and key as arguments.
terraform init
terraform apply -var "ca_cert=$(cat ~/.mkcert/localdev/rootCA.pem)" -var "ca_key=$(cat ~/.mkcert/localdev/rootCA-key.pem)"
```

## Kick the tires on our cluster

The default config file at ~/.kube/config is updated to add the KIND cluster credentials, so following terraform apply, you can interact with the cluster directly via kubectl.

```
# Check provisioned resources with kubectl
kubectl get node
kubectl get all -A
```

## Explore our services with a demo app

### Set up our environment

We can source the env file we created for ourselves with terraform to interact easily with our services from the shell, and read it to get credentials for browser logins.

```
cat ./.pipeline/pipeline.env
source ./.pipeline.env
```

### Check out our argocd-server web interface

We should be able to see our argocd URL in our terraform outputs as argocd_address, but it's also in the .pipeline/pipeline.env file we sourced, so we can view it if we need a reminder. If you haven't changed this value in terraform, it'll look like the output below.
```
echo $ARGOCD_ADDRESS

# example output
# https://argocd.maximal-default-cluster.localdev:9443
```

### Verify cert-manager and ingress-nginx functionality

We can access our ArgoCD web interface in the browser by way of a friendly localdev domain thanks to our localdev setup with dnsmasq above and ingress-nginx. When we do, we should see that the browser trusts the certificate we're using for https (unless you've omitted the step to use a trusted cert above). If you did not supply a trusted cert, you will need to click past a warning in the browser to access the ArgoCD login page, but you should still be able to view the self-signed certificate issued by the CA ClusterIssuer in the browser.


The argocd default admin user name is always 'admin.' We can find the password with our sourced env var if we didn't set it in terraform or we need a reminder.
```
echo $ARGOCD_ADMIN_PASSWORD
```

When we log into ArgoCD, we'll find it empty and waiting for its first app, so let's give it something to do.

### Deploy an application with ArgoCD

For a quick and dirty test, we can deploy an ArgoCD example app via either of the mechanisms from the [ArgoCD declarative setup documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/). For this walkthrough, let's set up a local repository in Gittea and configure ArgoCD to deploy from it.

For convenience, we can just push this repository to Gittea. There's a manifest for an ArgoCD app 


```
echo $GITEA_ADDRESS

# Example output
#https://gitea.maximal-default-cluster.localdev:9443
```





