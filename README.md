# OpenShift 4.11 on Nutanix AHV
The main idea of the use cases described [below](https://github.com/suulperi/ocp-nutanix/blob/main/README.md#use-case-1---running-workloads-in-two-dcs) is just showing a couple ideas for applications' disaster recovery when there is only two physical data centers for OpenShift Container Platform deployments and you want to be in safe if you lost one of your data centers. It's good to be aware if running applications in active-active mode on two different OpenShift clusters your application technologies must support it as well.

### Pre-reqs for the setup:
1. Three OpenShift Container Platforms running version 4.11 running on Nutanix AHV. Installation instructions can be found from [here](https://docs.openshift.com/container-platform/4.11/installing/installing_nutanix/preparing-to-install-on-nutanix.html)
   - One acting as Management Cluster - Red Hat ACM (hub). Installation instructions can be found from [here](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.6/html/install/installing#installing-while-connected-online). In real life scenario your management cluster should be located outside of your two data centers. For example in a public cloud.
   - Two clusters for running applications and managed by Red Hat ACM

2. Nutanix CSI Operator installed (2.6 or later) on each OpenShift Container Platform. Operator can be found from OperatorHub.
   - Follow instructions given by Operator
3. Enable iSCSI daemon on OpenShift Worker Nodes
```
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: 99-worker-enable-iscsid
spec:
  config:
    ignition:
      version: 3.2.0
    systemd:
      units:
      - enabled: true
        name: iscsid.service
```
4. Create Nutanix VolumesnapShotClass - it will be used with volSync
```   apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: nutanix-snapshot-class
driver: csi.nutanix.com
parameters:
  storageType: NutanixVolumes
  csi.storage.k8s.io/snapshotter-secret-name: ntnx-secret
  csi.storage.k8s.io/snapshotter-secret-namespace: openshift-cluster-csi-drivers
deletionPolicy: Delete
```
You can also use RH ACM Governance(policies) to enable iSCSI Daemon and create VolumeSnapshotClass. Policies can be found from [here](https://github.com/suulperi/ocp-nutanix/tree/main/policies)

5. OpenShift Image registry configuration
   - Instructions [here](https://opendocs.nutanix.com/openshift/post-install/)

6. Deploy Submariner
   - At the time of writing this RH ACM did not support deploying Submariner on Red Hat OpenShift Container Platform on Nutanix AHV by using RH ACM Web Console deployment must be done following bare metal instructions.
   - Take care of next firewall openings are done (between clusters):
     - IPsec NATT 4500/UDP
     - VXLAN 4800/UDP
     - Submariner metrics port 8080/TCP
     - Globalnet metrics port 8081/TCP
   - Deploy [these](https://github.com/suulperi/ocp-nutanix/tree/main/submariner) configurations on RH ACM Cluster - change values like namespace if currents are not suitable :
     - `oc create -f 00-managedClusterSet.yaml`
     - `oc create -f 01-broker.yaml` - if network CIDRs are overlapping on both clusters `globalnetEnabled`value must be `true`
     - `oc label managedclusters <your-managed-cluster-name> "cluster.open-cluster-management.io/clusterset=ocp-nutanix" --overwrite`
     - `oc label managedclusters <your-managed-cluster-name> "cluster.open-cluster-management.io/clusterset=ocp-nutanix" --overwrite`
     - `oc create -f 03-submarinerconfig-01.yaml`
     - `oc create -f 03-submarinerconfig-02.yaml`
     - `oc create -f 04-ManagedClusterAddOn-01.yaml`
     - `oc create -f 04-ManagedClusterAddOn-02.yaml`

   - Check Submariner status by using RH ACM Web Console. If everything is green like in image below you are good to continue

![Submariner status](/pics/submariner.png)

7. Install VolSync using RH ACM Web Console
   - Select one of the managed clusters from the Clusters page in the hub cluster and choose edit labels.
      - Add next label for your OCP `addons.open-cluster-management.io/volsync=true`
   - Repeat previous task on another managed cluster
   - Verify on both clusters volSync Operator is installed by running command `oc get csv -n openshift-operators` 
   - Now you are good to start configuring volSync(Use Case 2)
 
## Use Case 1 - Running workloads in two DCs
![Guestbook Architecture](/pics/guestbook-arch.png)

- Be aware **YOU** have to implement GLB. For demo purposes it can be single HAProxy instance or something similar. Quick and dirty HAProxy configuration below:
```
frontend guestbook-glb
    bind *:80
    default_backend guestbook

backend guestbook
    mode http
    balance roundrobin
    option forwardfor
    option httpchk
    http-check connect
    http-check send meth GET uri /
    http-check expect string Guestbook
    http-send-name-header Host
    server guestbook-guestbook.apps.ocp-01.your-domain.example guestbook-guestbook.apps.ocp-01.your-domain.example:80 check
    server guestbook-guestbook.apps.ocp-02.your-domain.example guestbook-guestbook.apps.ocp-02.your-domain.example:80 check
```
- Follow the instructions [here](https://github.com/suulperi/submariner-acm) to deploy Guestbook App and Redis Databases.
- Add some content into Redis Database using Guestbook. Validate both Guestbook instances sees same data after new inserts.
- It would require Redis Enterprise Cluster or **Red Hat Data Grid** to be able to run in-memory database in Active-Active mode. This scenario just proves how to synchronize data securely between two Redis instances over Submariner IPSec Tunnel. If you want to try a test case which would include losing redis-leader then you need to run command `redis-cli replicaof no one` on redis-follower to change role of it and make it able to write in Redis Database.

## Use Case 2 - Replicating data on persistent volume level from DC to another
![volSync Architecture](/pics/volsync-arch.png)

This use case could be usable if you are having an stateful application which cannot be scaled out but you want to be sure you have at least data synchronized to another data center. But do remember backups are still mandatory and really usable with containers as well. You must have disaster recovery plan for each application.

Use [these](https://github.com/suulperi/ocp-nutanix/blob/main/volsync/) configuration files part of volSync configurations

- Create a namespace on **DESTINATION** cluster `oc new-project destination-ns`
- Create a Replication Destination resource on **DESTINATION** cluster `oc create -f 00-replication-destination.yaml`
- Create a Submariner serviceExport **DESTINATION** cluster `oc create -f 01-serviceexport.yaml`
- Get Submariner Service Export external ip `oc get services -n destination-ns` --> Modify 03-replication-source.yaml - add Submariner external IP to address.rsync.spec
- Get sshKeys SSHKEYS=`oc get replicationdestination replication-destination -n destination-ns --template={{.status.rsync.sshKeys}}`
- echo $SSHKEYS --> Add output to Add it to 03-replication-source.yaml spec.rsync.sshKeys
- Get Secret: `oc get secret -n destination-ns $SSHKEYS -o yaml > /tmp/secret.yaml`
- Modify secret.yaml --> Change namespace to be your **SOURCE** cluster namespace --> Remove the owner references (.metadata.ownerReferences)
- Create a secret on the **SOURCE** cluster `oc create -f /tmp/secret.yaml`
- Create a namespace on **SOURCE** cluster `oc new-project source-ns`
- Implement Redis on **SOURCE** cluster using OCP Web Console
  - Choose Developer from top left corner
  - Click +Add
  - Click All Services
  - Search Redis --> Click it and Click Instantiate Template and Create Redis with default configuration
  - Check the name of Redis Persistent Volume Claim - it should be Redis
  - Add name of Redis' PVC to 03-replication-source.yaml spec.sourcePVC
- Add some content to Redis Database just for verifying replication works properly - Use OpenShift Web Console Terminal for example
  ![redis-cli commands](/pics/redis-cli.png)
  - redis-cli
  - auth your-password
  - Set mykey1 value1
  - Set mykey2 value2
  - KEYS *
- Create a replication source resource on **SOURCE** cluster `oc create -n source-ns -f 03-replication_source.yaml`
- Verify replication is working without issues on **SOURCE** cluster `oc describe ReplicationSource -n source-ns replication-source`
- Let's restore PVC from a Snapshot - name it Redis
![restore pvc](/pics/restore-pvc.png)
- Deploy Redis like earlier on the **SOURCE** cluster
- Verify Redis is having Keys which added in the source cluster
![verify Redis content](/pics/redis-dst.png)

That's it!



