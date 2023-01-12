# OpenShift 4.11 on Nutanix AOS
The main idea of the use cases described below is just showing different kind of options for applications' disaster recovery if environment is only having two different physical locations for your OpenShift environments and you want to be in safe if you lost one of your data centers. It's good to be aware if running applications in active-active mode on two different OpenShift clusters your application technologies must support it as well.

Pre-reqs for the setup:
1. Three OpenShift Container Platforms running version 4.11
   - One acting as Management Cluster - Red Hat ACM (hub). Installation instructions can be found from [here](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.6/html/install/installing#installing-while-connected-online)
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
4. Create Nutanix VolumesnapShotClass
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

## Use Case 1 - Running workloads in two DCs
![Guestbook Architecture](/pics/guestbook-arch.png)


## Use Case 2 - Replicating data on persistent volume level from DC to another
![volSync Architecture](/pics/volsync-arch.png)
