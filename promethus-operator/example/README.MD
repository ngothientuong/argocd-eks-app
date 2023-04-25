# Prometheus Requirements
## Storage
- Storage class. In this set up, we'll use EBS but it's not neccessary that way.
- Optional: VolumSnapshotClass with "deletionPolicy: Retain" for backup purpose
- PersistentVolumeClaim: Dynamically configured in Prometheus-server deployment's container as volumeClaimTemplate
- prometheus-server deployment: Configure volumeClaimTemplate referencing the ebs storage class

## 
