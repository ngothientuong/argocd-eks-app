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
# CILIUM
**Note**: Core Requirements are in the Cilium ms doc in google drive
## Cilium Conflicting Requirements
**Note**: Cluster communication works without encryption and without ingressController
- kubeProxyReplacement: partial|strict is required for ingressController conflicting with encyption.type=ipSec causing cluster connectivity failed
- encryption.type=ipSec does NOT appear to work with clustermesh EKS cluster
- encyption.type=wireguard does not support L7 Proxy (ingressController)
- Working combination: 
    - IngressController.enabled=true, kubeProxyReplacement=partial, encryption.enabled=false
    - IngressController.enabled=false, kubeProxyReplacement=partial, encryption.enabled=true, encryption.type=wireguard  [CURRENTLY CHOSEN SET UP]

### Cilium MAJOR clusterPoolIPv4PodCIDRList and clusterPoolIPv4MaskSize Requirement
- When installing with helm, ensure that the **clusterPoolIPv4MaskSize** number is LARGER that has smaller the mask size in clusterPoolIPv4PodCIDRList X.X.X.X/**{Mask Size}**. 
- The clusterPoolIPv4MaskSize will be the cidr_block allocated to EACH NODE out of the cidr_blocks pool provided by the clusterPoolIPv4PodCIDRList.
- In example below:
    - The clusterPoolIPv4PodCIDRList: ["10.208.0.0/**15**","10.210.0.0/**15**"] means that the total IPs given the the cluster is 10.208.0.1 -> 10.209.255.254
    - The clusterPoolIPv4MaskSize: "**21**"  means that the IP given to each node is with cidr_blocks 10.208.0.0/21, 10.208.8.0/28, ... 10.211.248.0/21
    - This means that each node can only have the maximum of 2046 virtual IPs assigned by Cilium because of the mask size /21
    - Notice the number **21** > than **15**
```
ipam:  
  mode: "cluster-pool"
  operator:    
    clusterPoolIPv4PodCIDRList: ["10.208.0.0/15","10.210.0.0/15"]
    clusterPoolIPv4MaskSize: "21"  
```

### Cilium Other Gotchas
- After enabling clustermesh, it could take some time before you can connect between 2 cluster. 
- If seeing the below error, simply rerun the command to connect between 2 clusters:
```
Error: Unable to connect cluster: unable to patch DaemonSet cilium with patch "{\"spec\":{\"template\":{\"spec\":{\"hostAliases\":[{\"ip\":\"\", \"hostnames\":[\"project-mgmt-eks-cluster-int-2.mesh.cilium.io\"]}]}}}}": DaemonSet.apps "cilium" is invalid: spec.template.spec.hostAliases.ip: Invalid value: "": must be valid IP address
```
1. Manually copy JUST the iam-roles terraform.tfstate from original bucket over
2. Added Peer Connection from 1 region's VPC and accepted on other region's VPC
3. Allow accepter VPC to resolve DNS of hosts in both requester and accepter VPN
4. Added entry for other VPC's cidr block pointing to the Peering Connection ID for VPC route table associating with the Private Subnets
5. Ensure the same route table there has subnet associations of private subnets in the Explicit subnet associations section and NOT the subnet without explicit associations section
6. If the worker node has secondary ENI and a whole bunch of secondary IPs assigned to it, then scale down the ASG to 0 and spin it back on. This might also require manual rollout restart of the deployments in all namespaces

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
- Example:
  - EU-MGMT-2 cluster:
    - NodePort: https://github.com/ngothientuong/eks-development/blob/master/environments/eumgmt/services/eks/addon/helm/values/cilium/1-hubble-ui-eu-west-2.yaml
    - ALB: https://github.com/ngothientuong/eks-development/blob/master/environments/eumgmt/services/eks/addon/helm/values/cilium/project-eumgmt-eks-eu-west-2-hubble-ui-alb.yaml

  - US-EAST-2 clusters:
    - NodePort and ALB are in the following folder: https://github.com/ngothientuong/eks-development/tree/master/environments/mgmt/services/eks/addon/cilium

### WireGuard Encryption At Rest [IN USE]
**Note:** WireGuard is in use as it's compatible with clustermesh communication

***Tutorial***: https://isovalent.com/blog/post/tutorial-transparent-encryption-with-ipsec-and-wireguard/

- WireGuard is NOT compatible with L7 Proxy Protocol. That means service mesh from Cilium can not be used with enabling WireGuard encryption and that cannot leverage Hubble for HTTP requests visibility or apply Layer 7 policy rules. 
- However, traffic between pods/services can still be visible in hubble-ui when enabling shared global service.
- WireGuard do not need encryption key. The encryption key pair for each node is automatically generated by Cilium and key rotation is performed transparently by the WireGuard kernel module.

#### Check if WireGuard is enabled
- Access to one of the daemonset cilium pod and check for status of encryption.
- View the traffic flow of the cilium_wg0 network interface
```
kubectl -n kube-system exec -ti ds/cilium -- sh
cilium status | grep Encryption
apt-get update
apt-get install tcpdump -y
tcpdump -n -i cilium_wg0
```
- You should be seeing traffic going through this interface
```
22:13:06.142555 IP 10.208.30.162.50132 > 10.208.13.229.53: 3841+ A? helloworld.sample.sample.svc.cluster.local. (60)
22:13:06.142621 IP 10.208.30.162.50132 > 10.208.13.229.53: 4073+ AAAA? helloworld.sample.sample.svc.cluster.local. (60)
22:13:06.142874 IP 10.208.13.229.53 > 10.208.30.162.50132: 4073 NXDomain*- 0/1/0 (153)
22:13:06.142923 IP 10.208.13.229.53 > 10.208.30.162.50132: 3841 NXDomain*- 0/1/0 (153)
22:13:06.143245 IP 10.208.30.162.55243 > 10.208.13.229.53: 35762+ A? helloworld.sample.svc.cluster.local. (53)
22:13:06.143245 IP 10.208.30.162.55243 > 10.208.13.229.53: 35935+ AAAA? helloworld.sample.svc.cluster.local. (53)
22:13:06.143434 IP 10.208.13.229.53 > 10.208.30.162.55243: 35935*- 0/1/0 (146)
22:13:06.143472 IP 10.208.13.229.53 > 10.208.30.162.55243: 35762*- 1/0/0 A 172.20.174.241 (104)
22:13:09.531645 IP 10.208.30.162.42983 > 10.208.13.229.53: 38462+ A? helloworld.sample.svc.cluster.local. (53)
22:13:09.531646 IP 10.208.30.162.42983 > 10.208.13.229.53: 38816+ AAAA? helloworld.sample.svc.cluster.local. (53)
22:13:09.531890 IP 10.208.13.229.53 > 10.208.30.162.42983: 38816*- 0/1/0 (146)
22:13:09.531943 IP 10.208.13.229.53 > 10.208.30.162.42983: 38462*- 1/0/0 A 172.20.174.241 (104)
```

### IP Sec [NOT IN USED]
**Note:** IP Sec is not in use in favor of WireGuard. The note here is just for reference
- Whenever IP Sec is enable, clustermesh is broken for communication between clusters
- IPSec has a larger code base than WireGuard and is used to support older network protocol. 
#### Cilium Enable IP Sec for each cluster
- Perform the following command for each cluster to create ipsec secret keys
>     kubectl --kubeconfig ~/.kube/{{ cluster_name }}.config create -n kube-system secret generic cilium-ipsec-keys --from-literal=keys="3 rfc4106(gcm(aes)) $(echo $(dd if=/dev/urandom count=20 bs=1 2> /dev/null | xxd -p -c 64)) 128"
- Perform terraform plan, terraform apply

#### Cilium Rotate IPSec secrets for each cluster
- Perform the following set of command   
```
KEYID=$(kubectl get secret -n kube-system cilium-ipsec-keys -o yaml | awk '/^\s*keys:/ {print $2}' | base64 -d | awk '{print $1}')
if [[ $KEYID -ge 15 ]]; then KEYID=0; fi
data=$(echo "{\"stringData\":{\"keys\":\"$((($KEYID+1))) "rfc4106\(gcm\(aes\)\)" $(echo $(dd if=/dev/urandom count=20 bs=1 2> /dev/null| xxd -p -c 64)) 128\"}}")
kubectl patch secret -n kube-system cilium-ipsec-keys -p="${data}" -v=1
```

## Trouble Shoot ClusterMesh
### Inter-Region ClusterMesh connectivity
1. ***Issue: Pod from 1 region could not reach to pod from another region via global shared service.***

  - **Resolution**: After standing up EKS cluster at another region, if any of the worker node has additional ENI shown on EC2 Dashboard then scale then ASG down to 0 manually and then spin it back up.

2. ***Issue: Error during connectivity test***
> curl -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code} --silent --fail --show-error --output /dev/null --connect-timeout 2 --max-time 10 http://10.204.198.86:8080" failed: command terminated with exit code 28

  - **Resolution** : Scale down NG to 0 and bring NG back on

3. ***Issue: Cannot get service IP during connectivity test***

  - **Resolution**: 
    - Modify svc clustermesh-apiserver
    - Remove entry: "service.beta.kubernetes.io/aws-load-balancer-internal: 0.0.0.0/0"  # Deprecated
    - Add entry "service.beta.kubernetes.io/aws-load-balancer-scheme: internal"

4. ***Issue: A cilium Pod failed to get kubernetes.service***
  - **Resolution**:
    - Restart deployment/cilium-operator 
    - Restart ds/cilium daemonset
    - If failure still, we'll disable clustermesh, delete cilium-test namespace's resources, delete clustermesh secrets between 2 cluster (if any still remains after disabling clustermesh), restart deploy/cilium-operator, ds/cilium and reenabling clustermesh between 2 clusters.
      - Delete cilium-test application in BOTH clusters
      ```
      for sa in echo-other-node echo-same-node client client2; do kubectl delete sa ${sa} -n cilium-test; done
      for svc in echo-other-node echo-same-node; do kubectl delete svc/${svc} -n cilium-test; done
      for deploy in echo-same-node echo-other-node client client2; do kubectl delete deploy/${deploy} -n cilium-test; done
      ```
      - Disable cilium clustermesh in both cluster
      ```
      cilium clustermesh disable --context $CLUSTER1
      cilium clustermesh disable --context $CLUSTER2
      ```
      - Delete clustermesh secrets. Do this in BOTH cluster
      ```
      for i in clustermesh-apiserver-admin-cert clustermesh-apiserver-client-cert clustermesh-apiserver-external-workload-cert clustermesh-apiserver-server-cert  ; do kubectl delete secret $i -n kube-system ; done
      ```
      - Restart deploycilium-perator:
      ```
      kubectl rollout restart deploy/cilium-operator -n kube-system
      ```
      - Restart ds/cilium
      kubectl rollout restart ds/cilium -n kube-system
      - [Enable clustermesh](#enable-cluster-mesh)

5. ***Issue: Random pod in ds/cilium never comes up after initialization, occurs to random nodes***
  - **Resolution**: 
    - Restart deployment/cilium-operator 
    - Restart ds/cilium daemonset
    - If issue persists:
      - Delete CiliumNodes (cn) resource(s) on problematic Node or on *all* nodes if cilium pod fail on random node after restart of deployment/cilium-operator and ds/cilium daemonset
        - Get the name of the CiliumResource for each node  
        ```
        kubectl get cn
        ```
        - List of all CiliumNodes will show. The name of each CiliumNode should be the same as the name of the host generated by cilium-operator
        Example:
        ```
        NAME                                        CILIUMINTERNALIP   INTERNALIP    AGE
        ip-110-0-1-10.eu-west-2.compute.internal    10.208.18.239      110.0.1.10    24h
        ip-110-0-1-167.eu-west-2.compute.internal   10.208.10.62       110.0.1.167   24h
        ip-110-0-1-91.eu-west-2.compute.internal    10.208.30.215      110.0.1.91    24h      
        ```
        - Delete CiliumNodes (cn) resource(s) on problematic Node
        ```
        kubectl delete cn ip-110-0-1-167.eu-west-2.compute.internal
        ```
      - Restart deployment/cilium-operator
      - Restart ds/cilium daemonset
      - Restart **all other** deployment, sts, ds in the cluster. [Manual - Yes it is a painful administraive process!]
---
# Other Issues
- ***Issue: Resources hang such as namespace forever***

  **Resolution**: 
  - Attempt to delete namespace, if keep seeing messages "customresourcedefinition.apiextensions.k8s.io "applications.argoproj.io" force deleted" that means resources relating to that namespace still remains and you have to delete it manually or else associated namespace cannot be deleted.
  - Run the following command: 
  ```
  kubectl patch customresourcedefinition.apiextensions.k8s.io/applications.argoproj.io -p '{"metadata":{"finalizers":[]}}' --type=merge
  kubectl delete customresourcedefinition.apiextensions.k8s.io/applications.argoproj.io
  ```
- ***Issue: ArgoCD app resources hang and can't be deleted from the console***
  **Resolution**: 
  - Change to cluster context hosting argoCD 
  - Change to namespace argocd
  - Run the following clean up commands
    ```
    kubectl patch app APPNAME -p '{"metadata": {"finalizers": null}}' --type merge
    kubectl delete app APPNAME
    ```

- ***Issue: Amazon Linux 2 cannot authenticate against a host due to CA not known***
  **Resolution**: 
  - Copy IMMEDIATE CA and ROOT CA of the endpoint cert to /etc/pki/ca-trust/source/anchors/
  - Run command:
  ```
  sudo update-ca-trust force-enable
  sudo update-ca-trust
  ```
---
# ISTIO

## Istio Multi-Primary Requirements
### Certificates
- Must have shared root-cert.pem, ca-cert.pem, ca-key.pem, cert-chain.pem acrossing clusters. The ca-key.pem, cert-chain.pem must be able to sign downstream certificates.
- Note that if your ca-cert.pem is the same as root-cert.pem, the cert-chain.pem file should be empty.
- Create secret generic cacerts in istio-system in each cluster with these files using --from-file
- In our development set up, we use ca-cert.pem and ca-key.pem generated by Cilium and placed in argocd's istiod/{cluster name}/create_secret_cacerts.yaml task.

### Network
- Similarly to cilium, the node and pods should be able to connect to each other.
### Istiod
- Istiod is a webhook in itself and requires to have hostNetwork=true
- Istiod uses hostPort of 8080 which collides with external-secrets webhook. An attempt to change hostPort, readiness port from 8080 to 8090 failed, most likely because the pilot application inside istiod is hardcoded to find internal istio service with port 8080.
- Mitigation for hostPort conflict: Bring up multiple ec2 instances in the ASG so webhooks with port conflicts such as port 8080 each can live on a separate ASG.

## Multi-Primary Set up
- Install istio helm charts of base, istiod and gateway. [Taken care of via argocd declarative set up]
- Create istio cluster profile for each cluster [Taken care of via argocd declarative set up]:
  - Ensure they have their own meshID, clusterID 
  - Ensure they all share they same network value, i.e. "network1"
```
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  labels:
    app.kubernetes.io/instance: istio-project-eumgmt-eks-eu-west-2
  name: istio-cluster-profile
  namespace: istio-system
spec:
  values:
    global:
      meshID: mesh4
      multiCluster:
        clusterName: project-eumgmt-eks-eu-west-2
      network: network1
```
- Install remote secret in 1 cluster that provide access to to other clusters' API server. [Manual]
```
istioctl x create-remote-secret --context=$CLUSTER1 --name=project-mgmt-eks-cluster-rancher-test | kubectl apply -f - --context=$CLUSTER2
istioctl x create-remote-secret --context=$CLUSTER1 --name=project-mgmt-eks-cluster-rancher-test | kubectl apply -f - --context=$CLUSTER3
istioctl x create-remote-secret --context=$CLUSTER1 --name=project-mgmt-eks-cluster-rancher-test | kubectl apply -f - --context=$CLUSTER4


istioctl x create-remote-secret --context=$CLUSTER2 --name=project-mgmt-eks-cluster-int-test | kubectl apply -f - --context=$CLUSTER1
istioctl x create-remote-secret --context=$CLUSTER2 --name=project-mgmt-eks-cluster-int-test | kubectl apply -f - --context=$CLUSTER3
istioctl x create-remote-secret --context=$CLUSTER2 --name=project-mgmt-eks-cluster-int-test | kubectl apply -f - --context=$CLUSTER4

istioctl x create-remote-secret --context=$CLUSTER3 --name=project-mgmt-eks-cluster-int-2 | kubectl apply -f - --context=$CLUSTER1 
istioctl x create-remote-secret --context=$CLUSTER3 --name=project-mgmt-eks-cluster-int-2 | kubectl apply -f - --context=$CLUSTER2
istioctl x create-remote-secret --context=$CLUSTER3 --name=project-mgmt-eks-cluster-int-2 | kubectl apply -f - --context=$CLUSTER4


istioctl x create-remote-secret --context=$CLUSTER4 --name=project-eumgmt-eks-eu-west-2 | kubectl apply -f - --context=$CLUSTER1
istioctl x create-remote-secret --context=$CLUSTER4 --name=project-eumgmt-eks-eu-west-2 | kubectl apply -f - --context=$CLUSTER2
istioctl x create-remote-secret --context=$CLUSTER4 --name=project-eumgmt-eks-eu-west-2 | kubectl apply -f - --context=$CLUSTER3
```

- End of of setting up Istio Multi-Primary.
- Look in folder istio/example to run test on multi-cluster setup
