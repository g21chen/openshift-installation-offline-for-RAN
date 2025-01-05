# openshift-installation-offline-for-RAN

# **Descriptions**
This repository is to specify how to offline install openshift due to some network restriction scenarios.


# **Prerequistes**
## **Hardware**
a. HPE/Dell/Supermicro/Nvidia server as target server which need install openshift

b. HPE/Dell server as image registry server

c. Jumper server which is used to execute the commands to configure and install openshift for target server

d. DNS server (optional)

e. NTP server (optional)

note: if possible, DNS,NTP and image registry can be in same one hardware server

## **software tool**
a. docker image registry   refer: https://github.com/g21chen/PriviateImageRegistry
   Note: image registry server need the access to redhat openshift registry for openshift images pulling.
   
b. DNS server

c. Chronyd NTP server

d. oc-mirror    donwload example: wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.16.4/oc-mirror.tar.gz

e. openshift-client-linux download example: https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.16.4/openshift-client-linux-4.16.4.tar.gz 

note: move binaries oc-mirror and oc into /usr/local/bin/


# **installation steps**
## **1. mirror redhat openshift images to private image registry**
### **1.1 prepare for the pull secret file**
Pulling redhat openshift images requests the pull secret file of specific redhat account. Your personal pull secret for the Red Hat online repositories can be downloaded from Red Hat (requires Red Hat account).
![pullsecret](https://github.com/user-attachments/assets/da4c0fe4-3b9b-4a98-b8a6-0f7fba347e36)

add the credentials of accessing private image registry into pull-secret.json file
```json
{
  "auths": {
    "cloud.openshift.com": {
      "auth": "",
      "email": ""
    },
    "quay.io": {
      "auth": "",
      "email": ""
    },
    "registry.connect.redhat.com": {
      "auth": "",
      "email": ""
    },
    "registry.redhat.io": {
      "auth": "",
      "email": ""
    },
    "xxx.yyy.zzz.ttt:5000": {          //xxx.yyy.zzz.ttt is the IP address of priviate image registry
       "auth": ""                      //base64 encoded credentials on priviate image registry
     }
   }
}
```
### **1.2 cp pull-secret file to global docker config file**
cp pull-secret.json ~/.docker/config.json


### **1.3 prepare for the ImageSetConfiguraion file**
The imageSetConfiguraiton.yaml file defines all necesrray image files and operators catalog. Here are the example for ImageSetConfiguration.yaml file
```yaml
{
apiVersion: mirror.openshift.io/v1alpha2
kind: ImageSetConfiguration
mirror:
  platform:
    channels:
    - name: stable-4.16
      minVersion: 4.16.24
      maxVersion: 4.16.24
    graph: true
  operators:
  - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.16
    packages:
    - name: openshift-gitops-operator
    - name: local-storage-operator
    - name: mcg-operator
    - name: ocs-operator
    - name: odf-operator
    - name: topology-aware-lifecycle-manager
    - name: ptp-operator
    - name: lvms-operator
    - name: sriov-network-operator
    - name: nfd
  additionalImages:
  - name: registry.redhat.io/rhel8/support-tools:latest
  - name: registry.redhat.io/ubi8/ubi:latest
  - name: registry.redhat.io/odf4/odf-must-gather-rhel9:v4.16
  - name: registry.redhat.io/rhacm2/acm-must-gather-rhel9:v2.11
  - name: registry.redhat.io/openshift4/
}
```

note: if needs the images and catalog not from redhat, it needs specify the separate imageSetConfiguration file to define them. e.g Nvidia-GPU operator is not provided by redhat.

### **1.4 mirror the openshift images to private image registry**
There are two options to mirror images to private image registry, one is directly mirror images from remote redhat registry to local private image registry. another one is firstly mirror from remote redhat image registry to local disk and then upload from local to priviate image registry. the example is using the 2nd option.

#### **1.4.1 mirror remote redhat image registry to local disk**

oc mirror --config=./ImageSetConfiguration.yaml file:release_mirror/
```bash
$ ls -la release_mirror/
total 109383720
drwxr-x--- 3 rmatusch rmatusch         4096 Aug  5 13:08 .
drwxr-x--- 3 rmatusch rmatusch         4096 Aug  5 10:33 ..
-rw-r--r-- 1 rmatusch rmatusch 112008886272 Aug  5 13:21 mirror_seq1_000000.tar
drwxr-xr-x 4 rmatusch rmatusch         4096 Aug  5 13:04 oc-mirror-workspace
```

#### **1.4.2 uploading the images from local disk to private image registry**
oc mirror --from=.release_mirror/mirror_seq1_000000.tar docker://xxx.yyy.zzz.ttt:5000         //xxx.yyy.zzz.ttt is the ip of priviate image registry


#### **1.4.3 verify the image availability via querying repository**
curl -s -u "xxx:yyy" https://zzz.ttt.aaa.bbb:5000/v2/_catalog --cacert domain.crt | jq .

```bash
xxx: account of auth when creating priviate docker image registry
yyy: password of auth when creating priviate docker image registry
zzz.ttt.aaa.bbb: ip address of priviate docker image registry
domain.crt: certificate of priviate docker image registry
```

```json
{
  "repositories": [
    "ocp/images/lvms4/lvms-must-gather-rhel9",
    "ocp/images/odf4/odf-must-gather-rhel9",
    "ocp/images/rhel9/support-tools",
    "ocp/redhat-operator/compliance/openshift-file-integrity-operator-bundle",
    "ocp/redhat-operator/kmm/kernel-module-management-webhook-server-rhel9",
    "ocp/redhat-operator/oadp/oadp-velero-restic-restore-helper-rhel9",
    "ocp/redhat-operator/odf4/odf-csi-addons-rhel9-operator",
    "ocp/redhat-operator/odf4/odf-operator-bundle",
    "ocp/redhat-operator/odf4/odf-rhel9-operator",
    "ocp/redhat-operator/openshift-logging/cluster-logging-rhel9-operator",
    "ocp/redhat-operator/openshift-logging/fluentd-rhel9",
    "ocp/redhat-operator/openshift-logging/vector-rhel9",
    "ocp/redhat-operator/openshift4/metallb-rhel9-operator",
    "ocp/redhat-operator/openshift4/ose-csi-external-snapshotter-rhel8",
    "ocp/redhat-operator/openshift4/ose-kube-rbac-proxy",
    "ocp/redhat-operator/openshift4/ose-metallb-operator-bundle",
    "ocp/redhat-operator/openshift4/ose-sriov-network-device-plugin",
    "ocp/redhat-operator/openshift4/ose-sriov-network-operator",
    "ocp/redhat-operator/openshift4/topology-aware-lifecycle-manager-operator-bundle",
    "ocp/redhat-operator/rhacm2/console-rhel9",
    "ocp/redhat-operator/rhacm2/grafana-dashboard-loader-rhel9",
    "ocp/redhat-operator/rhacm2/kube-rbac-proxy-rhel9",
    "ocp/redhat-operator/rhacm2/multicloud-integrations-rhel9",
    "ocp/redhat-operator/rhacm2/observatorium-rhel9",
    "ocp/redhat-operator/rhacm2/thanos-rhel9",
    "ocp/redhat-operator/workload-availability/self-node-remediation-rhel8-operator",
    "ocp4-release/odf4/cephcsi-rhel9",
    "ocp4-release/odf4/mcg-core-rhel9",
    "ocp4-release/odf4/mcg-operator-bundle",
    "ocp4-release/odf4/mcg-rhel9-operator",
    "ocp4-release/odf4/ocs-client-console-rhel9",
    "ocp4-release/odf4/ocs-client-operator-bundle",
    "ocp4-release/odf4/ocs-client-rhel9-operator",
    "ocp4-release/odf4/ocs-metrics-exporter-rhel9",
    "ocp4-release/odf4/ocs-operator-bundle",
    "ocp4-release/odf4/ocs-rhel9-operator",
    "ocp4-release/odf4/odf-cli-rhel9",
    "ocp4-release/odf4/odf-console-rhel9",
    "ocp4-release/odf4/odf-cosi-sidecar-rhel9",
    "ocp4-release/odf4/odf-csi-addons-operator-bundle",
    "ocp4-release/odf4/odf-csi-addons-rhel9-operator",
    "ocp4-release/odf4/odf-csi-addons-sidecar-rhel9",
    "ocp4-release/odf4/odf-must-gather-rhel9",
    "ocp4-release/odf4/odf-operator-bundle",
    "ocp4-release/odf4/odf-prometheus-operator-bundle",
    "ocp4-release/odf4/odf-rhel9-operator",
    "ocp4-release/odf4/odr-recipe-operator-bundle",
    "ocp4-release/odf4/odr-rhel9-operator",
    "ocp4-release/odf4/rook-ceph-operator-bundle",
    "ocp4-release/odf4/rook-ceph-rhel9-operator",
    "ocp4-release/openshift",
    "ocp4-release/openshift-logging/logging-loki-rhel9",
    "ocp4-release/openshift-logging/loki-operator-bundle",
    "ocp4-release/openshift-logging/loki-rhel9-operator",
    "ocp4-release/openshift-logging/lokistack-gateway-rhel9",
    "ocp4-release/openshift-logging/opa-openshift-rhel9",
    "ocp4-release/openshift/graph-image",
    "ocp4-release/openshift/release",
    "ocp4-release/openshift/release-images",
    "ocp4-release/openshift4-release-images",
    "ocp4-release/openshift4/ose-configmap-reloader",
    "ocp4-release/openshift4/ose-csi-external-attacher-rhel9",
    "ocp4-release/openshift4/ose-csi-external-provisioner",
    "ocp4-release/openshift4/ose-csi-external-provisioner-rhel9",
    "ocp4-release/openshift4/ose-csi-external-resizer",
    "ocp4-release/openshift4/ose-csi-external-resizer-rhel9",
    "ocp4-release/openshift4/ose-csi-external-snapshotter-rhel9",
    "ocp4-release/openshift4/ose-csi-node-driver-registrar",
    "ocp4-release/openshift4/ose-csi-node-driver-registrar-rhel9",
    "ocp4-release/openshift4/ose-haproxy-router",
    "ocp4-release/openshift4/ose-kube-rbac-proxy",
    "ocp4-release/openshift4/ose-kube-rbac-proxy-rhel9",
    "ocp4-release/openshift4/ose-local-storage-diskmaker-rhel9",
    "ocp4-release/openshift4/ose-local-storage-mustgather-rhel9",
    "ocp4-release/openshift4/ose-local-storage-operator-bundle",
    "ocp4-release/openshift4/ose-local-storage-rhel9-operator",
    "ocp4-release/openshift4/ose-oauth-proxy",
    "ocp4-release/openshift4/ose-oauth-proxy-rhel9",
    "ocp4-release/openshift4/ose-prometheus-alertmanager-rhel9",
    "ocp4-release/openshift4/ose-prometheus-config-reloader-rhel9",
    "ocp4-release/openshift4/ose-prometheus-rhel9",
    "ocp4-release/openshift4/ose-prometheus-rhel9-operator",
    "ocp4-release/openshift4/topology-aware-lifecycle-manager-aztp-rhel9",
    "ocp4-release/openshift4/topology-aware-lifecycle-manager-operator-bundle",
    "ocp4-release/openshift4/topology-aware-lifecycle-manager-precache-rhel9",
    "ocp4-release/openshift4/topology-aware-lifecycle-manager-recovery-rhel9",
    "ocp4-release/openshift4/topology-aware-lifecycle-manager-rhel9-operator",
    "ocp4-release/openshift4/ztp-site-generate-rhel8",
    "ocp4-release/redhat/redhat-operator-index",
    "ocp4-release/rh-sso-7/sso75-openshift-rhel8",
    "ocp4-release/rh-sso-7/sso76-openshift-rhel8",
    "ocp4-release/rhceph/rhceph-6-rhel9",
    "ocp4-release/rhceph/rhceph-7-rhel9",
    "ocp4-release/rhel8/postgresql-12",
    "ocp4-release/rhel8/redis-6",
    "ocp4-release/rhel8/support-tools",
    "ocp4-release/rhel9/postgresql-13",
    "ocp4-release/rhel9/postgresql-15",
    "ocp4-release/ubi8/ubi",
    "ocp4-release/ubi8/ubi-micro"
  ]
}
```

#### **1.4.4 identify the image content source policy and catalog source custom resource manifests**
After uploading the image to priviate docker image registry, it auto generates the image content source policy and catalog source customer resources manifests file under directory./oc-mirror-workspace/, those manifests files are requested for OCP instllation in later phase. the example:
```bash
xxx:~/sam/openshift/registry/oc-mirror-workspace$ ll results-1735268307
total 156
drwxrwxr-x 4 aods aods   4096 Dec 27 03:36 ./
drwxrwxr-x 8 aods aods   4096 Jan  1 04:18 ../
-rwxrwxr-x 1 aods aods    226 Dec 27 03:36 catalogSource-cs-redhat-operator-index.yaml*
drwxrwxr-x 2 aods aods   4096 Dec 27 02:58 charts/
-rwxrwxr-x 1 aods aods   1892 Dec 27 03:36 imageContentSourcePolicy.yaml*
-rw-rw-r-- 1 aods aods 127483 Dec 27 03:36 mapping.txt
drwxrwxr-x 2 aods aods   4096 Dec 27 03:34 release-signatures/
-rwxrwxr-x 1 aods aods    317 Dec 27 03:36 updateService.yaml*

```
```yaml
xxx:~/sam/openshift/registry/oc-mirror-workspace/results-1735268307$ cat imageContentSourcePolicy.yaml
---
apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  name: generic-0
spec:
  repositoryDigestMirrors:
  - mirrors:
    - xxx.yyy.zzz.ttt:5000/openshift4
    source: registry.redhat.io/openshift4
  - mirrors:
    - xxx.yyy.zzz.ttt:5000/ubi8
    source: registry.redhat.io/ubi8
  - mirrors:
    - xxx.yyy.zzz.ttt:5000/ubi8
    source: registry.access.redhat.com/ubi8
  - mirrors:
    - xxx.yyy.zzz.ttt:5000/rhel8
    source: registry.redhat.io/rhel8
---
apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  labels:
    operators.openshift.org/catalog: "true"
  name: operator-0
spec:
  repositoryDigestMirrors:
  - mirrors:
    - xxx.yyy.zzz.ttt:5000/openshift-gitops-1
    source: registry.redhat.io/openshift-gitops-1
  - mirrors:
    - xxx.yyy.zzz.ttt:5000/odf4
    source: registry.redhat.io/odf4
  - mirrors:
    - xxx.yyy.zzz.ttt:5000/rh-sso-7
    source: registry.redhat.io/rh-sso-7
  - mirrors:
    - xxx.yyy.zzz.ttt:5000/lvms4
    source: registry.redhat.io/lvms4
  - mirrors:
    - xxx.yyy.zzz.ttt:5000/rhel8
    source: registry.redhat.io/rhel8
  - mirrors:
    - xxx.yyy.zzz.ttt:5000/rhceph
    source: registry.redhat.io/rhceph
  - mirrors:
    - xxx.yyy.zzz.ttt:5000/multicluster-engine
    source: registry.redhat.io/multicluster-engine
  - mirrors:
    - xxx.yyy.zzz.ttt:5000/openshift4
    source: registry.redhat.io/openshift4
  - mirrors:
    - xxx.yyy.zzz.ttt:5000/rhacm2
    source: registry.redhat.io/rhacm2
  - mirrors:
    - xxx.yyy.zzz.ttt:5000/rhel9
    source: registry.redhat.io/rhel9
---
apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  name: release-0
spec:
  repositoryDigestMirrors:
  - mirrors:
    - xxx.yyy.zzz.ttt:5000/openshift/release
    source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
  - mirrors:
    - xxx.yyy.zzz.ttt:5000/openshift/release-images
    source: quay.io/openshift-release-dev/ocp-release

```

```yaml
xxx:~/sam/openshift/registry/oc-mirror-workspace/results-1735268307$ cat catalogSource-cs-redhat-operator-index.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: cs-redhat-operator-index
  namespace: openshift-marketplace
spec:
  image: xxx.yyy.zzz.ttt/redhat/redhat-operator-index:v4.16
  sourceType: grpc

```
## **2 deploy openshift container platform**
### **2.1 tool insallation in jump server**
install nmstate.  nmstate is required for the network configuration in ocp installation

redhat/centos/rocky: 
```bash
$ sudo dnf -y install nmstate
 
$ dnf list nmstate --installed
Installed Packages
nmstate.x86_64                                                                        2.2.21-2.fc38                                                                        @updates
```
### **2.2 prepare for the configuration file for OCP deployment**
#### "2.2.1 image content source policy"
ImageContentSourcePolicy.yaml
```yaml
apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  name: test
spec:
  repositoryDigestMirrors:
  - mirrors:
    - xxx.yyy.zzz.ttt:5000/openshift/release
    source: quay.io/openshift-release-dev/ocp-v4.0-art-dev

```
#### "2.2.2 pull-secret.json"
```json
{
    "auths": {
        "xxx.yyy.zzz.ttt:5000": {
           "auth": "xxxyyyzzztttooo",
           "email": "xxx.yyy@zzz.com"
           }
     }
}
```
auth are base64 encoded result of the credentials when create the priviate image registry

#### "2.2.3 prepare for agent config file"
agent-config file configure the infra network to communcate with outside, dns server. below is the example of single node:
```yaml
apiVersion: v1alpha1
metadata:
  name: test
hosts:
  - hostname: master0
    role: master
    interfaces:
     - name: ens14f0                     
       macAddress: b4:96:91:e1:xx:yy      //Mac address of infra network interface
    rootDeviceHints:
      deviceName: "/dev/nvme0n1"          //disk of ocp system installation  
    networkConfig:
      dns-resolver:
        config:
          server:
            - 10.48.xx.yy               //DNS server address
      interfaces:
        - name: ens14f0
          type: ethernet
          state: up
          ipv4:
            enabled: true
            dhcp: false
          ipv6:
            enabled: false
          mac-address: b4:96:91:e1:xx:yy
          mtu: 1500
          ethernet:
            auto-negotiation: true
            duplex: full
            speed: 25000
        - name: ens14f0.zzz         //zzz is the vlan   
          type: vlan
          state: up
          ipv4:
            enabled: true
            dhcp: false
            address:
            - ip: 192.168.xxx.yyy    //infra IP of target server
              prefix-length: 28
          vlan:
            base-iface: ens14f0
            id: zzz             //vlan ID
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 192.168.xxx.zzz    //gateway IP of infra subnetwork
            next-hop-interface: ens14f0.zzz      //vlan ID
            table-id: 254

```

#### **2.2.4 prepare for install-config file**
install-config file includes the hardware cpu model, deployment type, internal cluster network, ssh public key, image source contents policy, pull-secret and certificates.
```bash
apiVersion: v1
baseDomain: vran.mnrancis.nsn-rdnet.net
compute:
 - architecture: amd64
   hyperthreading: Enabled
   name: worker
   replicas: 0
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  replicas: 1
metadata:
  name: test
networking:
  clusterNetwork:
    - cidr: 172.21.0.0/16
      hostPrefix: 23
  machineNetwork:
    - cidr: 192.168.xxx.yyy/28   //infra network
  networkType: OVNKubernetes
  serviceNetwork:
    - 172.22.0.0/16
platform:
  none: {}
pullSecret: "{\"auths\": {\"xxx.yyy.zzz.ttt:5000\": {\"auth\": \"xyzhigzzzzzhffff\"}}}"   //auths of private image registry
sshKey: 'ssh-rsa AAAAB3NzaC1yc2EAAAADxxxAQDITBmY6zqniC21HxxxxxxxxxxxxPUQFvN5xP8yLNF5aCFi4yjfLHCxxxxxxxx88rE5qtZCje519ADQjD3D5sQw6DWLGXbIQ2/Xl9LJpKLG5f316in8DITIxxxxxxxxxxn0E+9IMEjtWjvEoWxMLGSekb4SXMqk3rZ2tufa2bpFleqn5Vd5y+Yj0D8Tx7XCJct01Vpgw3R2FTH47DppQiGskKjgy+gGGekUTdPucMsbRRpTShTENj4YAJssCT5+4AiP7xJVETD+VpwdUaElUdObQcrOyC723PfZPGBpGKnDYyYySQgNJ0NqAGMYQiFc3TvagUMQ5Hrfa8QjKTAOtDUIcJAVBxxxxxxxx1KO3i8CaTSCe0wipRfKTKcWy7bWnTj41uDTFGBsOUsBx7bIgDiebtcI5c70hAj2SX6fFin5Lr0CQKWxxxxxxxxxIMUU3Rr/vS/cmZI7RcMhH9yT4pmR1XzwmOj83OHCmwhn8z++eexBhrZIXoEu0DF5fS8zN82oAE2epEYxBTzllegYxxxxxpubv7n0nUO9hxxxxx9mmxmSkeAFQgdV/lInLsaNM0b8lArehUZ2lqFgwspVuvp/Ifqt4NcvK391SpYf63lt4TcPpUfw90oXxxNw3Mhak+zC1pPNKjs5B5QNh4Dw2rMIw==xxxxxxxxxxxx'
imageContentSources:
- mirrors:
  - xxxx.yyy.zzz.tttt:5000/openshift/release                // xxxx.yyy.zzz.tttt: ip address of docker image registry
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
- mirrors:
  - xxxx.yyy.zzz.tttt:5000/openshift/release-images
  source: quay.io/openshift-release-dev/ocp-release
additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
  MIIF/jCCA+agAwIBAgIUHBDFdEqt+kdBkPjuy75DzV/ZS4kwDQYJKoZIhvcNAQEL
  BQAwgYUxCzAJxxxxxxxTAlVTMQ4wDAYDVQQIDAVUZXhhczEPMA0GA1UEBwwGRGFs
  bGFzMQwwCgYDVQQKDAN0b20xDTALBgNVBAsMBGphY2sxEjAQBgNVBAMMCWNyYW4u
  Y3JhbjEkMCIGCSqGSIb3DQEJARYVZ2FuZy4xLmNoZW5Abm9raWEuY29tMB4XDTI0
  MTIxNDA1MDcyyyyyyy1MTIxNDA1MDc1NVowgYUxCzAJBgNVBAYTAlVTMQ4wDAYD
  VQQIDAVUZXhhczEPMA0GA1UEBwwGRGFsbGFzMQwwCgYDVQQKDAN0b20xDTALBgNV
  BAsMBGphY2sxEjAQBgNVBAMMCWNyYW4uY3JhbjEkMCIGCSqGSIb3DQEJARYVZ2Fu
  Zy4xLmNoZW5AbxxxxxxxxxxxxIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKC
  AgEAm4z5lRZto0DqH3pJNHgCMudaK0iwIrxp805b+o1SXINzIHZjqiUoNgVQZkFN
  GFe19/j3uZa271LG3wwrgjCXouYJOTXxfmx8qECu51W7iPtbIRU1D+8pVfRTjCTL
  jABrlRrBjEHRwzzzzzzzzzzzzyFVfUTCLVfgvy5Onh1PEGhpsA7QLkMQkjZSyOWB
  1zVCSZ3aVLx1ea+pvsFU3oBsofmN4KIUoN1nhIqErHeHRUmAwxCPUP7r0Wh8adsi
  4I5N0J14CeGjmqK0S9q4TTHT0bqc7L80+MDSjTuskRKbCmLUykacKLpPEHslSobE
  NT/CR1GV6ymjvzbbbbbbbbbbbbbbPPZ3fA9BewgeDzyDChW4fe3fUaaweZcqI/23
  CojlQO10AMsicifUKFkdAo1Z6fTL526g2R9UqSdcx4IRNoI4lqcnhF8Rk5D6dyiX
  H9hxJ668DEGefEBRyuZ0ylXIw65JOPplwh2cyDiYTm2rYTgvYRwrOsiW/lb/JF8B
  WaVlIq+tDVdoMpZcFlLufpsE8wMGCh59MYFSzRRKmqMOZbtkPevXahyGjpY3kOty
  P97mW6Z3mdCJ7u369qVty/2a6zzRmZLDSm6E2mUEvgf9gzpSSim0EgsU+L6dTAIW
  sMeiTYl2qisK7dspbc9RdOPCwah4Xsf+6XgJSulTKMxYPicCAwEAAaNkMGIwHQYD
  VR0OBBYEFDwGrW3gmEhLg8DCWpCH2f1anI3xMB8GA1UdIwQYMBaAFDwGrW3gmEhL
  g8DCWpCH2f1anI3xMA8GA1UdEwEB/wQFMAMBAf8wDwYDVR0RBAgwBocECjAIxzAN
  BgkqhkiG9w0BAQsFAAOCAgEAGPBfMjHm0v9DDpNAzHLppSQwELGIfjFchc/4fILo
  rfdktapDZoUfMyNLQujgfb3D1HKhNhXTejgZWzMCjObOfRTH9W6hEQw2H3i0GaPb
  oXpkzB31HAT9+gTXIXZIm9EWl8EKxchEx6WJvj/7tYQPx4n68zw8vBiKcdVQxU+X
  UT+uKqKyd9+SfLI0Asa5FnHx8sgkg1x4jclkOKzqr7VpqfyyeQzbS8p/H3BhE1ee
  5yUf3cfk7BVsgQkrLxxxxxxxxxxxxxxxmLPThHJGFIKWfZTsU4u+u5r3yj1v6gPN
  DX/A5GwqqP4EGWpdMlxMAAPgWrItpobHSkYbQjKHWkBQyluqZx2fGeSsNpTVki1N
  ThF1mMJBp75gHUN6xuyTx1+X2qtBG8se61ONebHZXY7imRmOfyW/I9CWxcFpy47L
  AWyO6tMuUiqHJ/By5xxxxxxxxxxxxxxxxxxxxDIwIfmI/l2ABHUiufsT/Dw8yakR
  EgvAyKytP4XQuWXwh+SGcCCRKfcZSiH+66PszhNL/es7LNG7Ne58OduRIe3TfEq1
  HS+XbOQAwjC5e+qbjilxxxxxxxxxxxxxxxxRWjW9JuTx/eGYhMkqxFak59R/kgwJ
  UdWIRBXAJYiGhwD9+xsKdY98peUXtWupasCwRiVmZE0QEd0V6wrXac2Qk3Z2+0pf
  7VY=
  -----END CERTIFICATE-----
```
#### **2.2.5 prepare for chronyd file**
The target server as NTP client need configure the NTP server address in chronyd configuration file
```yaml
[xxx]$ cat 99-masters-chrony-configuration.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: master
  name: 99-masters-chrony-configuration
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      files:
      - contents:
          source: data:text/plain;charset=utf-8;base64,ZHJpZnRmaWxlIC92YXIvbGliL2Nocm9ueS9kcmlmdAptYWtlc3RlcCAxLjAgMwpydGNzeW5jCmxvZ2RpciAvdmFyL2xvZy9jaHJvbnkKc2VydmVyIG50cC4xMC40OC54eHgueXl5IGlidXJzdA==
        mode: 420
        overwrite: true
        path: /etc/chrony.conf
```
```bash
$ echo "ZHJpZnRmaWxlIC92YXIvbGliL2Nocm9ueS9kcmlmdAptYWtlc3RlcCAxLjAgMwpydGNzeW5jCmxvZ2RpciAvdmFyL2xvZy9jaHJvbnkKc2VydmVyIG50cC4xMC40OC54eHgueXl5IGlidXJzdA==" | base64 -d
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
logdir /var/log/chrony
server 10.48.xxx.yyy iburst
$
```

### **2.3 deploy OCP**
#### **2.3.1 extract the openshift-install tool from docker image registry**
The prepared files in 2.2.1, 2.2.2 and certiicates are required for this step, the commandline as below:
```bash
oc adm release extract --command=openshift-install --certificate-authority=/home/labuser/sam/ocp-install/ocp/artifacts/openshift/domain.crt xxx.yyy.zzz.ttt:5000/openshift/release-images:4.16.24-x86_64 --icsp-file=/home/labuser/sam/ocp-install/ocp/artifacts/openshift/ImageContentSourcePolicy.yaml --to=/home/labuser/sam/ocp-install/ocp/artifacts/ -a /home/labuser/sam/ocp-install/ocp/artifacts/openshift/pull_secret.json

```
after this step, it generates the openshift-install bin.

#### **2.3.2 copy the configuration into work directory**
copy all mandatory files into the work directory. example as below:
```bash
[xxxxxxx]$ tree
.
├── artifacts
│   ├── 99-masters-chrony-configuration.yaml
│   ├── agent-config.yaml
│   ├── install-config.yaml
│   ├── openshift
│   │   ├── domain.crt
│   │   ├── ImageContentSourcePolicy.yaml
│   │   └── pull_secret.yaml
│   └── openshift-install

```

#### **2.3.3 create iso dicovery image**
 ./openshift-install agent create image --dir /home/labuser/sam/ocp-install/ocp/artifacts

output:
```bash
 [test]$ tree ./artifacts/
./artifacts/
├── 99-masters-chrony-configuration.yaml
├── agent.x86_64.iso
├── auth
│   ├── kubeadmin-password
│   └── kubeconfig
├── openshift
│   ├── domain.crt
│   └── pull_secret.json
├── openshift-install

```
it generates the necessary results:
agent.x86_64.iso: iso image
kubeadmin-password: password on access the redhat console
kubeconfig: kube config file to access k8s API server
note: after this step, some configuration files in this directory will be automatcially removed.

#### **2.3.4 mount iso image into target server**
##### **2.3.4.1 login the BMC network**

![ILO BMC](https://github.com/user-attachments/assets/db3b4aea-de43-45e5-9b1e-248b37d29b53)

##### **2.3.4.2 configure server boot from CD/DVD Drive**

![BOOT-FROM-CD](https://github.com/user-attachments/assets/4cc29f5d-c3bc-4efe-9e11-e4cfae913255)


##### **2.3.4.3 mount iso file**

![mount iso](https://github.com/user-attachments/assets/b61635a1-afcd-4ac9-97be-7a2e062e1556)

##### **2.3.4.4 reset server**
![reset](https://github.com/user-attachments/assets/cda771fa-e070-486c-9de4-6611af1adf75)

after reset, the iso is loaded in server and one simiple redhat OS is available, and it will trigger the connection to redhat assisted installer.
