For a full overview on Authentication, refer to the official Kubernetes docs on Authentication and Authorization

For users, ideally you use an Identity provider for Kubernetes (OpenID Connect).

If you are on GKE / ACS you integrate with respective Identity and Access Management frameworks

If you self-host kubernetes (which is the case when you use kops), you may use coreos/dex to integrate with LDAP / OAuth2 identity providers - a good reference is this detailed 2 part SSO for Kubernetes article.

for Dex there are a few open source cli clients as follows:

Nordstrom/kubelogin
pusher/k8s-auth-example
If you are looking for a quick and easy (not most secure and easy to manage in the long run) way to get started, you may abuse serviceaccounts - with 2 options for specialised Policies to control access. (see below)

NOTE since 1.6 Role Based Access Control is strongly recommended! this answer does not cover RBAC setup

EDIT: Great guide by Bitnami on User setup with RBAC is also available.

Steps to enable service account access are (depending on if your cluster configuration includes RBAC or ABAC policies, these accounts may have full Admin rights!):

EDIT: Here is a bash script to automate Service Account creation - see below steps

Create service account for user Alice

kubectl create sa alice
Get related secret

secret=$(kubectl get sa alice -o json | jq -r .secrets[].name)
Get ca.crt from secret (using OSX base64 with -D flag for decode)

kubectl get secret $secret -o json | jq -r '.data["ca.crt"]' | base64 -D > ca.crt
Get service account token from secret

user_token=$(kubectl get secret $secret -o json | jq -r '.data["token"]' | base64 -D)
Get information from your kubectl config (current-context, server..)

# get current context
c=`kubectl config current-context`

# get cluster name of context
name=`kubectl config get-contexts $c | awk '{print $3}' | tail -n 1`

# get endpoint of current context 
endpoint=`kubectl config view -o jsonpath="{.clusters[?(@.name == \"$name\")].cluster.server}"`
On a fresh machine, follow these steps (given the ca.cert and $endpoint information retrieved above:

Install kubectl

brew install kubectl
Set cluster (run in directory where ca.crt is stored)

kubectl config set-cluster cluster-staging \
  --embed-certs=true \
  --server=$endpoint \
  --certificate-authority=./ca.crt
Set user credentials

kubectl config set-credentials alice-staging --token=$user_token
Define the combination of alice user with the staging cluster

kubectl config set-context alice-staging \
  --cluster=cluster-staging \
  --user=alice-staging \
  --namespace=alice
Switch current-context to alice-staging for the user

kubectl config use-context alice-staging