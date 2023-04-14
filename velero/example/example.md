# How to run backup
- Install velero cli

### Perform manual backup

> velero backup logs initial-full-backup-with-argocd

### Schedule a backup every 6 hours
> velero schedule create full-backup-6h --schedule="0 0/6 * * *"


