# Start Here

If you've followed a link to this repo, but are not really sure what it contains
or how to use it, head over to [Multicloud GitOps](http://hybrid-cloud-patterns.io/multicloud-gitops/)
for additional context and installation instructions

## Create the secrets file

When configuring vault there is a `values-secret.yaml` file that `push_secrets` Ansible playbook will use.
For Kong we will create a key-value for the license as follows:

```bash
cat << EOF >> values-secret.yaml 
secrets:
  kong:
    license: "$(sed 's/\"/\\\"/g' license.json)"
EOF
```

## Get the tokens and certs of the external clusters

Use these variables to create an entry for your cluster in the `values-secret.yaml` file using the following code:

```bash
CLUSTER_NAME=example
CLUSTER_API_URL=https://api.mycluster.jqic.p1.openshiftapps.com:6443
oc login $CLUSTER_API_URL
oc create sa argocd-external -n default
oc adm policy add-cluster-role-to-user cluster-admin system:serviceaccount:default:argocd-external
CLUSTER_TOKEN=$(oc describe secret -n default argocd-external-token | grep 'token:' | awk '{print$2}')
CLUSTER_CA=$(oc extract -n openshift-config cm/kube-root-ca.crt --to=- --keys=ca.crt | base64 | awk '{print}' ORS='')
```

Use the previous environment variables to create an entry for your cluster in the `values-secret.yaml` file using the following code:

```bash
cat << EOF >> values-secret.yaml 
  cluster_${CLUSTER_NAME}:
    name: ${CLUSTER_NAME}
    server: ${CLUSTER_API_URL}
    config: |
      {
        \"bearerToken\": \"${CLUSTER_TOKEN}\",
        \"tlsClientConfig\": {
          \"insecure\": false,
          \"caData\": \"${CLUSTER_CA}\"
        }
      }
EOF
```

## Install the main Helm chart

```bash
make install
```

## Uninstall

```bash
make uninstall
```

**NOTE:** Sometimes the ArgoCD Applications are not deleted. You will have to remove the finalizers. This happens when all the ArgoCD objects are deleted
at the same time, most often because helm uninstalls everything at once.