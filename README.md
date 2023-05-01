
# <p style="text-align: center;">ARGOCD-EKS-APP</p>
---

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
# <p style="text-align: center;">CILIUM</p>
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

# <p style="text-align: center;">ISTIO</p>

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

---
# PROMETHEUS OPERATOR - ALERTMANAGER, GRAFANA AND PROMETHEUS
## Prometheus Operator - Requirements
### Helm values
- Webhook Port for prometheus operator: Change prometheus.tls.internalPort from 10250 to 10260 although there didn't seem to be any conflict with other
- Specify no certificates, or secrets even for the Prometheus Operator
- Disable all Ingress
- Disable anything relating to Thanos as it's unknown at this point
- Optional: EBS volumeClaimTemplate for Grafana & Prometheus
### Certificate and Secrets
- ClusterIssuer 
- Certificates for cert-manager to generate tls secrets to be used in AWS ALB Ingress and istio-gateway Ingress. **The names of the cert and the secretName generated by the cert should NEVER be the same.**
- No certificate or secrets needed for Prometheus Operator or any spawn components such as alertmanager, grafana and prometheus
- The certs used by aws-alb Ingress and istio-gateway Ingress must be signed by the same CA as they make direct handshake connection.
### AWS ALB & Istio Ingress
#### AWS ALB
- Create 1 singled combined rules Ingress type ALB for all hosts of the applications such as alertmanager, grafana and prometheus. 
- The list in tls.hosts has to match **exactly** with the list of DNS names specify by the certificate
- No wild-card in any of the host due to external dns writing the wild-card dns to Route 53
  ```
  Note: Creating a certificate with the set of application hosts will generate a secret by cert-manager. This secret is specified in AWS ALB's tls.secretName. Cert-manager in aws-alb ingress resource will check for the list of hosts in aws alb tls.secretName and the content in the secret for mismatch validation. Once it's complete, a cert with the same name of the secret of aws alb tls.secretName will be created.
  ```
- The AWS ALB Ingress will route traffic to the Istio Gateway Pods in istio-system namespace
- From here, you can create 1 single istio-gateway Ingress to catch traffic from the ALB
#### Istio-Gateway 
- Create 1 single Istio Gateway Ingress in istio-system namespace
- Catch all hosts with '*'
- Now you're in position to create any VS, DR in *different namespaces* by referencing the istio-gateway ingress at the address {istio-gateway-svc-name}.istio-system.svc.local, i.e, **istio-gateway.istio-system.svc.cluster.local**

### EBS Volume [Optional]
- Storage Class with provisioner 'ebs.csi.aws.com', retain policy
- VolumeClaimTemplate in Helm values for Grafana and Prometheus.
- VolumeSnapshotClass for backup up purposes later

### RBAC [Potential issue]
- If encountering issue with prometheus operator cannot access API server endpoinslices, bind sa 'prometheus-community-kube-operator' to cluster-admin role

## Prometheus Operator - Installation
- URL: https://www.cncf.io/blog/2021/10/25/prometheus-definitive-guide-part-iii-prometheus-operator/#:~:text=In%20simple%20words%2C%20Prometheus%20Operator,cluster%20monitoring%20in%20just%20a
- Github Repo: https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack

---
# KIALI
**URL**: https://github.com/kiali/kiali-operator/blob/master/crd-docs/cr/kiali.io_v1alpha1_kiali.yaml
## Kiali Requirements
- Required pre-existing components: Istio , Prometheus (preferred)
- Optional needed components: Grafana, Jaeger
- Kiali-Operator install via Helm is missing CRD. Must install separately!  [Correction: Not missing, just forgot to set createCRD: to true in helm.chart field]

## Install
- Kiali-Operator Github: https://github.com/kiali/kiali-operator
- Kiali missing CRD URL: https://github.com/kiali/kiali-operator/blob/master/crd-docs/crd/kiali.io_kialis.yaml 
- Have to install Kiali backends with Grafana, Prometheus, Istio via Kiali resource

---

# <p style="text-align: center;">ELK - ELASTIC SEARCH, KIBANA, LOGSTASH, FILEBEAT</p>
---
# <p style="text-align: center;">ELASTIC SEARCH</p>
**URL**: https://github.com/elastic/helm-charts/tree/main/elasticsearch
- Elasticsearch is a distributed, free and open search and analytics engine for all types of data, including textual, numerical, geospatial, structured
## ES Operator Gotchas
- When installing ES cluster via the Operator, every time the ES Cluster restarts, it'll generate new secrets for elasticsearch-master-credentials and elasticsearch-master-certs which affects hard coded connection from downstream application such as Logstash, Kibana, Jaeger, FileBeat.

## TO DO
- Need to work on installing ES with predefined certificate and credential.
- You can configure additional admin users in ES during the ES creation and use those for downstream clusters
## ES Requirements
### Storage
- Use EBS Storage, can be io1 type.
- Must have iopsPerGB (max value 50)
- AllowVolumeExpansion and ReclaimPolicy Retain, WaitForFirstConsumer
- Install storage class in cluster_wide ahead of time
### ES Helm Values
```
esConfig:
  elasticsearch.yml: |
    network.host: 0.0.0.0  
createCert: true
secret:
  enabled: true
  password: "" 
networkHost: "0.0.0.0"
volumeClaimTemplate:
  ### Tim - added storageClassName
  storageClassName: ebs-sc
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 55Gi
persistence:
  # enabled: true
  ### Tim
  enabled: true
ingress:
  enabled: false    
```
## ES Installation
- Standard install via argoCD


# <p style="text-align: center;">KIBANA</p>
**URL**: https://github.com/elastic/helm-charts/tree/main/kibana
- Kibana is an free and open frontend application that sits on top of the Elastic Stack, providing search and data visualization capabilities for data indexed
## Kibana Connection to ES cluster
- Kibana will need ES credential and cert, i.e secrets generated by ES to connect to ES cluster

## Kiabana Requirements
### Secrets & Certificates 
- Must have Kiabana in the same namespace with elastic search so it can grab the secrets generated by elastic search install such as ca.crt, elasticsearch-master-certs and elasticsearch-master-credentials
- Add additional hostname for Kiabna in certificate creation of tngo-eks-alb-gw-cert and tngo-eks-apps-certs for the AWS ALB ingress and Istio Ingress via cluster_wide argocd app.
- If elastic search pods are restarted or its app in argocd UI is synced, new secret is generated. Must delete the **elastic-kibana-es-token** secret and re-sync Kibana!
### Kibana Helm Values
```
### Tim
replicas: 2
resources:
  requests:
    cpu: "1000m"
    memory: "2Gi"
  limits:
    cpu: "1000m"
    memory: "2Gi"
### Tim
kibanaConfig: 
  kibana.yml: |    
    server.host: 0.0.0.0
    server.publicBaseUrl: https://tngo-eks-kibana-mgmt-eu.bylightsdc.bylight.com
### Tim
automountToken: false
```
### Kibana AWS ALB Ingress and Istio VS-DR
- Add additional hostname for Kiabna for the AWS ALB ingress **gw-ingress-alb** cluster_wide argocd app in the **istio-system** namespace
- Create Istio VS-DR in the **elastic-system** namespace for Kibana

## Kibana Installation
- Must wait until elastic search complete the install to generate the elasticsearch-master secrets.
- Must not refresh/sync elastic search app from argoCD UI as it will generate new master secrets and make down stream apps such as Kibana, Logstash, Jaeger, FileBeat out of sync.

## Kibana Login UI
- Credential is from **elasticsearch-master-credentials** secret

# <p style="text-align: center;">FILEBEAT</p>
**URL**: https://github.com/elastic/helm-charts/tree/main/filebeat
- Filebeat is logs collecting agent.
- In k8s world, filebeat collects containers logs and send to either ES backend directly or to Logstash then ES backend.
- In our setup, we keep the install prestined and have filebeat send logs directly to ES backend
- Filebeat is installed as a deamonset.
## Filebeat Connection to ES cluster
- Filebeat will need ES credential and cert, i.e secrets generated by ES to connect to ES cluster
## Filebeat Requirement
### Helm Values
- N/A
### Installation
- Standard install via argoCd App

# <p style="text-align: center;">LOGSTASH</p>
URL: https://github.com/elastic/helm-charts/tree/main/logstash

## Logstash Connection to ES cluster
- Logstash will need ES credential and cert, i.e secrets generated by ES to connect to ES cluster.

## Logstash Requirements
- Must have xpac.monitoring logstash.yaml setting set with xpack.monitoring.
- Must have path.config pointing to the folders containing the pipeline conf files such as /usr/logstash/pipeline.

### Logstash Storage
- Use **EFS** Storage Class.
- Must have AWS EFS file system created ahead of time.
- Create AWS EFS file system manually for now as described in section **Create an Amazon EFS file system** in this document: https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html
- AllowVolumeExpansion and ReclaimPolicy Retain, WaitForFirstConsumer
- Install storage class in cluster_wide ahead of time
- 
### Logstash Helm Values
```
replicas: 2

logstashConfig: 
  logstash.yml: |
    xpack.monitoring.enabled: true
    xpack.monitoring.elasticsearch.hosts: [ "https://elasticsearch-master:9200" ]
    xpack.monitoring.elasticsearch.username: "elastic" 
    xpack.monitoring.elasticsearch.password: "lYuVyDtfwId2JwDm"
    xpack.monitoring.elasticsearch.ssl.verification_mode: none
    http.host: "0.0.0.0"
    path.data: /usr/share/logstash/data
    pipeline.workers: 4
    pipeline.batch.size: 254
    pipeline.batch.delay: 90
    pipeline.unsafe_shutdown: false
    pipeline.ordered: auto
    path.config: /usr/share/logstash/pipeline
    config.test_and_exit: false
    config.reload.automatic: true
    config.reload.interval: 3s
    config.debug: true
    log.level: trace
    queue.type: memory
    path.queue: /usr/share/logstash/queue
    queue.page_capacity: 64mb
    queue.max_events: 0
    queue.max_bytes: 1024mb
    queue.checkpoint.acks: 1024
    queue.checkpoint.writes: 1024
    queue.checkpoint.interval: 1000
    dead_letter_queue.enable: false
    log.level: info
    path.logs: /usr/share/logstash/data/logs    

logstashPipeline:
  logstash.conf : |
    input {
      beats {
        port => 5044
      }
    }

    output {
      elasticsearch {
        hosts => ["https://elasticsearch-master:9200"]
        index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
        ssl => false
        ssl_certificate_verification => false 
        user => "elastic" 
        password => "lYuVyDtfwId2JwDm"
      }
    }

resources:
  requests:
    cpu: "100m"
    memory: "1536Mi"
  limits:
    cpu: "1000m"
    memory: "1536Mi"

volumeClaimTemplate:
  ### Tim - added storageClassName
  storageClassName: efs-sc
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 59Gi        
persistence:
  # enabled: false
  ### Tim
  enabled: true

livenessProbe:
  httpGet:
    path: /
    port: http
    # host: 127.0.0.1
  initialDelaySeconds: 300
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
  successThreshold: 1

readinessProbe:
  httpGet:
    path: /
    port: http
    # host: 127.0.0.1
  initialDelaySeconds: 60
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
  successThreshold: 3
```
### Logstash AWS ALB Ingress and Istio VS-DR
- Add additional hostname for Logstash for the AWS ALB ingress **gw-ingress-alb** cluster_wide argocd app in the **istio-system** namespace
- Create Istio VS-DR in the **elastic-system** namespace for Logstash

### Logstash Pipeline
- Logstash can receive traffic from filebeat or other logs collector agents. Logstash can then set up pipeline to send traffic to backend elastic search servers via pipeline conf files.

---
# <p style="text-align: center;">OBSERVABILITY</p>
---
# <p style="text-align: center;">JAEGER</p>
**URL**: https://github.com/jaegertracing/helm-charts/tree/main/charts/jaeger-operator

## Jaeger Connection to ES cluster
- Jaeger will need ES credential and cert, i.e secrets generated by ES to connect to ES cluster.

## Jaeger Requirements
### ES Connection
- Jaeger will need backend storage for Production use. ES is one of the storage recommended.
### Jaeger Certiciate & Secrets
- Much like Kibana, Logstash, Filebeat, Jaeger uses certs and connection strings generated via ES to connect to ES.
- The tls.crt that is used to authenticate against the ES has to be the CA of ES itself or the tls.crt that contains just the service name of ES cluster in the form {elasticsearch-clustername}.{namespace} with no suffix 'svc.cluster.local', e.g. elasticsearch-master.elastic-system 
### Jaeger Helm Values
```
certs:
  issuer:
    create: true
    name: "tngo-eks-ca"
  certificate:
    create: true
    namespace: "observability"
    secretName: "jaeger-operator-tls"
    # Specify the cert-manager issuer kind to use an existing cert-manager
    # issuer; typically Issuer or ClusterIssuer
    # This field will be ignored if issuer.create is true
    issuerKind: AWSPCAClusterIssuer

    jaeger:
  # Specifies whether Jaeger instance should be created
  create: false
  # namespace where Jaeger resource should be created default to .Release.Namespace
  namespace: "observability"  
```  
### Jaeger - Create 'kind: Jaeger' with type: Production
- Type Production is meant for production use with proper backend such as Elastic Search
- Must have ES credentials **ES_USERNAME** and **ES_PASSWORD** saved in storage.secretName
- Must have ES certificate or ES CA certificate saved in spec.volumes/0/secret/secretName, i.e spec.volumes[0].secret.secretName.
```
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  labels:
    jaegertracing.io/operated-by: observability.jaeger-operator
  name: jaeger-prod-stragtegy
  namespace: observability
spec:
  strategy: production
  collector:
    strategy:
      type: RollingUpdate
      rollingUpdate: 
        maxSurge: 10%  
    maxReplicas: 2
    resources:
      limits:
        cpu: 100m
        memory: 128Mi  
  storage:
    type: elasticsearch 
    deployment on and off
    #   schedule: "55 23 * * *"                       # cron expression for it to run
    #   sparkMaster:                                  # spark master connection string, when empty spark runs in embedded local mode
    esIndexCleaner:
      enabled: true                                # // turn the cron job deployment on and off
      numberOfDays: 7                              # // number of days to wait before deleting a record
      schedule: "55 23 * * *"                      # // cron expression for it to run    
    options:
      es:
        server-urls: https://elasticsearch-master.elastic-system:9200
        index-prefix: jaeger_
        create-index-templates: false  ## Needed to resolve template issues
        tls.enabled: true
        tls.ca: /es/certificates/tls.crt   
    secretName: elasticsearch-master-credentials
  ingress:
    enabled: false
  volumeMounts: 
    - name: certificates
      mountPath: /es/certificates/
      readOnly: true
  volumes:
    - name: certificates
      secret:
        secretName: elasticsearch-master-certs
```
### Jaeger AWS ALB Ingress and Istio VS-DR
- Add additional hostname for Jaeger for the AWS ALB ingress **gw-ingress-alb** cluster_wide argocd app in the **istio-system** namespace
- Create Istio VS-DR in the **elastic-system** namespace for Jaeger

## Jaeger Installation
- Standard argoCd app install