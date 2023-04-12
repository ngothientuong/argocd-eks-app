# argocd-eks-app
Repo for deploying apps to EKS cluster

## Critical item

- Ensure to update the CA certificate:
    - External-Snapshotter:
        - From AWSPCA when it expires
        - When changing the certificate PCA vendor from AWSPCA  to others such as Venafi, Lets Encrypt


