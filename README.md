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

### AWS EFS CSI
- URL: https://github.com/kubernetes-sigs/aws-efs-csi-driver
- Watch out for the node.DNS as a modification has to be required for use across different VPCs.
- FIPS encryption at rest is enabled. Has to be mounting your file system using encryption of data in transit.
- Need to read through other requirements
