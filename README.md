# argocd-eks-app
Repo for deploying apps to EKS cluster

## Successful rolled out components
- external-secrets (tested)
- external-snapshotter (untested)

## Critical item

### Ensure to update the CA certificate
- External-Snapshotter:
    - From AWSPCA when it expires
    - When changing the certificate PCA vendor from AWSPCA  to others such as Venafi, Lets Encrypt

- External-Snapshotter:
    - Certificate requested as snapshot-validation-secret and mounted to snapshot-validation-deployment (the webhook) 

### Buckets
- Ensure buckets for backup storage for Velero exists
