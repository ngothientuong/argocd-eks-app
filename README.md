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

--- 
## Cilium Conflicting Requirements
**Note**: Cluster communication works without encryption and without ingressController
- kubeProxyReplacement: partial|strict is required for ingressController conflicting with encyption.type=ipSec causing cluster connectivity failed
- encryption.type=ipSec does NOT appear to work with clustermesh EKS cluster
- encyption.type=wireguard does not support L7 Proxy (ingressController)
- Working combination: 
    - IngressController.enabled=true, kubeProxyReplacement=partial, encryption.enabled=false
    - IngressController.enabled=false, kubeProxyReplacement=partial, encryption.enabled=true, encryption.type=wireguard  [CURRENTLY CHOSEN SET UP]

### Cilium Other Gotchas
- After enabling clustermesh, it could take some time before you can connect between 2 cluster. 
- If seeing the below error, simply rerun the command to connect between 2 clusters:
```
Error: Unable to connect cluster: unable to patch DaemonSet cilium with patch "{\"spec\":{\"template\":{\"spec\":{\"hostAliases\":[{\"ip\":\"\", \"hostnames\":[\"project-mgmt-eks-cluster-int-2.mesh.cilium.io\"]}]}}}}": DaemonSet.apps "cilium" is invalid: spec.template.spec.hostAliases.ip: Invalid value: "": must be valid IP address
```

### Cilium Multi-Cluster Setup
#### URL
https://docs.cilium.io/en/stable/network/clustermesh/clustermesh/

#### Share CA secret set up (Has to install Cilium twice)
##### **WAY 1 - via Helm Install**
First install:

1.  Copy files from local_exec/python_scripts/rendered_shell_files 
    and put under helm/values/cilium
2.  Remove the ca.cert and ca.key value in helm's value file for each cluster if exists
3.  Run terraform init, plan and apply as usual

Second Install:

4.  After Cilium was installed the first time, view the cilium-ca secret to obtain certificate, key
5.  Insert the ca.cert and ca.key into helm's value files for each cluster
6.  Rerun terraform plan, terraform apply to update the ca certs across cluster
7.  Check if cilium-ca and hubble-ca-secret have the same keys and values across ALL cluster as it's needed for cluster communication


###### **Way 2 - Via Manual set up (risky)**
- After Cilium was installed, manually copy secret content of **cilium-ca** and **hubble-ca-secret** from 1 main cluster to others

Example for **cilium-ca** secret:

- cilium-ca secret: Copy manually .data.ca\\\\.crt and .data.ca\\\\.key content from secret cilium-ca from cluster 1 to other cluster
>    kg secret cilium-ca $ksn -oyaml 

- **Note**: Should the connectivity test in the following test failed, ensure to add the CNI interfaces to the trusted zone (firewalld?). In our case, the firewall is not enforced in the linux worker nodes and SG is used in stead. SG already is already configured to open port 1025 - 65535 as well as ICMP 8-0.
            
            Reference URL: 
                https://github.com/cilium/cilium-cli/issues/673#issuecomment-1018708961

                https://github.com/firewalld/firewalld/issues/767

##### Enable Cluster Mesh
- export and source clusters' contexts (context from kubeconfig) in **~/.bashrc**
>   CLUSTER1=arn:aws:eks:us-east-2:682331604648:cluster/project-mgmt-eks-cluster-rancher-test

>     CLUSTER2=arn:aws:eks:us-east-2:682331604648:cluster/project-mgmt-eks-cluster-int-test

>     CLUSTER3=arn:aws:eks:us-east-2:682331604648:cluster/project-mgmt-eks-cluster-int-2

- Optional: Check cluster status: 
    
    cilium clustermesh status --context $CLUSTER1 --wait

    cilium clustermesh status --context $CLUSTER2 --wait

- Connect Cluster
>     cilium clustermesh connect --context $CLUSTER1 --destination-context $CLUSTER2        

- Test connectivity between clusters which takes a long while:
>     cilium connectivity test --context $CLUSTER1 --multi-cluster $CLUSTER2

### Cilium Hubble-UI set up
- Change type svc for hubble to type NodePort for each cluster
- Create ALB API Resources to point to the Node Group NodePort

## Cilium Enable IP Sec for each cluster
- Perform the following command for each cluster to create ipsec secret keys
>     kubectl --kubeconfig ~/.kube/{{ cluster_name }}.config create -n kube-system secret generic cilium-ipsec-keys --from-literal=keys="3 rfc4106(gcm(aes)) $(echo $(dd if=/dev/urandom count=20 bs=1 2> /dev/null | xxd -p -c 64)) 128"
- Perform terraform plan, terraform apply

## Cilium Rotate IPSec secrets for each cluster
- Perform the following set of command   
```
KEYID=$(kubectl get secret -n kube-system cilium-ipsec-keys -o yaml | awk '/^\s*keys:/ {print $2}' | base64 -d | awk '{print $1}')
if [[ $KEYID -ge 15 ]]; then KEYID=0; fi
data=$(echo "{\"stringData\":{\"keys\":\"$((($KEYID+1))) "rfc4106\(gcm\(aes\)\)" $(echo $(dd if=/dev/urandom count=20 bs=1 2> /dev/null| xxd -p -c 64)) 128\"}}")
kubectl patch secret -n kube-system cilium-ipsec-keys -p="${data}" -v=1
```
## Trouble Shoot ClusterMesh
### Inter-Region ClusterMesh connectivity
- Issue: Pod from 1 region could not reach to pod from another region via global shared service.

  **Resolution**: After standing up EKS cluster at another region, if any of the worker node has additional ENI shown on EC2 Dashboard then scale then ASG down to 0 manually and then spin it back up.

