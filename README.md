# Installation Openshift 4.2 with OCS 4



#### Requirement for Openshift Container Storage 4.2

> **WARNING :** Please make sure the spec is same and size memory is very important

1. Worker 1 for OCS = 16vCPU, 64 GB memory ( disk 1 = 100GB , disk 2 = 10 GB , disk 3 = 2Tib)
2. Worker 2 for OCS =  16vCPU, 64 GB memory ( disk 1 = 100GB , disk 2 = 10 GB , disk 3 = 2Tib)
3. Worker 3 for OCS = 16vCPU, 64 GB memory ( disk 1 = 100GB , disk 2 = 10 GB , disk 3 = 2Tib)

Disk 1 : for CoreOS

Disk 2 : for MONs

Disk 3 : for OSD

==============================================================================

 Please make sure you have 3 worker for OCS environment :

```
root@bastion# oc get nodes

NAME                            STATUS   ROLES    AGE    VERSION
master-0.ocp.datacenter.com   Ready    master   3d2h   v1.14.6+c07e432da
sto-1.ocp.datacenter.com      Ready    worker   3d2h   v1.14.6+c07e432da
sto-2.ocp.datacenter.com      Ready    worker   3d2h   v1.14.6+c07e432da
sto-3.ocp.datacenter.com      Ready    worker   3d2h   v1.14.6+c07e432da
worker-0.ocp.datacenter.com   Ready    worker   3d2h   v1.14.6+c07e432da
worker-1.ocp.datacenter.com   Ready    worker   3d2h   v1.14.6+c07e432da
```

I use sto-1..3.ocp.datacenter.com for Storage Cluster

You must install first Local Storage in Operator

1. Create the `local-storage` project:

   ```
   $ oc new-project local-storage
   ```

2. Install the Local Storage Operator from the web console.

   1. Log in to the OpenShift Container Platform web console.
   2. Navigate to **Operators** → **OperatorHub**.
   3. Type **Local Storage** into the filter box to locate the Local Storage Operator.
   4. Click **Install**.
   5. On the **Create Operator Subscription** page, select **A specific namespace on the cluster**. Select **local-storage** from the drop-down menu.
   6. Adjust the values for the **Update Channel** and **Approval Strategy** to the desired values.
   7. Click **Subscribe**.

   ![](https://raw.githubusercontent.com/h4ckersmooth88/ocs4/master/2.PNG)

   

3. Once finished, the Local Storage Operator will be listed in the **Installed Operators** section of the web console.

Now we can create Local Storage for Mon and OSD, please check the configuration :

For Mon :

```
[root@bastion ~]# cat local-sc-mon.yaml
apiVersion: "local.storage.openshift.io/v1"
kind: "LocalVolume"
metadata:
  name: "local-sc-mon"
  namespace: "local-storage"
spec:
  nodeSelector:
    nodeSelectorTerms:
    - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - sto-1.ocp.datacenter.com
          - sto-2.ocp.datacenter.com
          - sto-3.ocp.datacenter.com
  storageClassDevices:
    - storageClassName: "local-sc-mon"
      volumeMode: Filesystem
      fsType: xfs
      devicePaths:
        - /dev/sdb
        
[root@bastion ~]# oc create -f local-sc-mon.yaml
```

For OSD :

```
[root@bastion ~]# cat local-sc-osd.yaml
apiVersion: "local.storage.openshift.io/v1"
kind: "LocalVolume"
metadata:
  name: "local-sc-osd"
  namespace: "local-storage"
spec:
  nodeSelector:
    nodeSelectorTerms:
    - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - sto-1.ocp.datacenter.com
          - sto-2.ocp.datacenter.com
          - sto-3.ocp.datacenter.com
  storageClassDevices:
    - storageClassName: "local-sc-osd"
      volumeMode: Block
      devicePaths:
        - /dev/sdc

[root@bastion ~]# oc create -f local-sc-osd.yaml
```

for YAML you can download in github

please make sure PV created :

```
root@bastion# oc get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY  ....
local-pv-7746386c                          2000Gi     RWO            Delete             
local-pv-779de3f3                          10Gi       RWO            Delete               
local-pv-a2215bd6                          2000Gi     RWO            Delete           
local-pv-b845a036                          10Gi       RWO            Delete           
local-pv-d7522569                          2000Gi     RWO            Delete              
```

Now we labeling worker for OCS Storage :

```
root@bastion# oc label nodes sto-1.ocp.datacenter.com cluster.ocs.openshift.io/openshift-storage=

root@bastion# oc label nodes sto-2.ocp.datacenter.com cluster.ocs.openshift.io/openshift-storage=

root@bastion# oc label nodes sto-3.ocp.datacenter.com cluster.ocs.openshift.io/openshift-storage=
```

You can check with command this :

```
[root@bastion ~]# oc describe nodes/sto-1.ocp.datacenter.com
Name:               sto-1.ocp.i3datacenter.com
Roles:              worker
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    cluster.ocs.openshift.io/openshift-storage=
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=sto-1.ocp.datacenter.com
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/worker=
                    node.openshift.io/os_id=rhcos
                    topology.rook.io/rack=rack2
```

Now we install Openshift Cluster Storage in Operator :

```
root@bastion# oc create namespace openshift-storage
```

And Then : 

1.  Click **Administration → Namespaces** in the left pane of the OpenShift Web Console. 							
2.  Click **Create Namespaces**. 							
3. In the Create Namespace dialog box, enter `openshift-storage` for Name and `openshift.io/cluster-monitoring=true` for Labels. This label is required to get the dashboards. 	Select **No restrictions** option for **Default Network Policy**. 							
4. Click **Create**. 							

**Procedure**

1. Log in to the Red Hat OpenShift Container Platform Web Console. 					

2. Click **Operators → OperatorHub**. 					

   **Figure 1.1. List of operators in the Operator Hub**

   [![Screenshot of list of operators in the Operator Hub of the OpenShift Web Console.](https://access.redhat.com/webassets/avalon/d/Red_Hat_OpenShift_Container_Storage-4.2-Deploying_OpenShift_Container_Storage-en-US/images/eb9b0019a51afd1449f10583cd3dff38/ocs_install_operatorhub_1.png)](https://access.redhat.com/webassets/avalon/d/Red_Hat_OpenShift_Container_Storage-4.2-Deploying_OpenShift_Container_Storage-en-US/images/eb9b0019a51afd1449f10583cd3dff38/ocs_install_operatorhub_1.png)

3. Search for **OpenShift Container Storage Operator** from the list of operators and click on it. 				

4. On the OpenShift Container Storage Operator page, click **Install**. 					

5. On the Create Operator Subscription page, the Installation Mode,  Update Channel, and Approval Strategy options are available. 					

   **Figure 1.2. Create Operator Subscription page**

   [![Screenshot of create operator subscription page.](https://access.redhat.com/webassets/avalon/d/Red_Hat_OpenShift_Container_Storage-4.2-Deploying_OpenShift_Container_Storage-en-US/images/a334ae270689aa0e84d7de1f079b7020/ocs_install_subscription.png)](https://access.redhat.com/webassets/avalon/d/Red_Hat_OpenShift_Container_Storage-4.2-Deploying_OpenShift_Container_Storage-en-US/images/a334ae270689aa0e84d7de1f079b7020/ocs_install_subscription.png)

   1.  Select **A specific namespace on the cluster** for the Installation Mode option. 							

      -  Select `openshift-storage` namespace from the drop down menu. 									

       stable-4.2** channel is selected by default for the Update Channel option. 							

   2.  Select an Approval Strategy: 							

      -  **Automatic** specifies that you want OpenShift Container Platform to upgrade OpenShift Container Storage automatically. 									
      - **Manual** specifies that you want to have control to upgrade OpenShift Container Storage manually. 									

6.  Click **Subscribe**. 					

   **Figure 1.3. Installed operators**

   [![Screenshot of the installed operators.](https://access.redhat.com/webassets/avalon/d/Red_Hat_OpenShift_Container_Storage-4.2-Deploying_OpenShift_Container_Storage-en-US/images/7e0bcdc0b35dec13fbcda87f748b120d/ocs_installed_operators.png)](https://access.redhat.com/webassets/avalon/d/Red_Hat_OpenShift_Container_Storage-4.2-Deploying_OpenShift_Container_Storage-en-US/images/7e0bcdc0b35dec13fbcda87f748b120d/ocs_installed_operators.png)

    						The Installed Operators page is displayed with the status of the operator. 					

You can monitoring with command :

```
root@bastion# oc get pod -n openshift-storage

root@bastion# oc get csv -n openshift-storage
NAME                  DISPLAY                       VERSION   REPLACES   PHASE
ocs-operator.v4.2.1   OpenShift Container Storage   4.2.1                Installing
```

Now you apply the Storage Cluster :

```
[root@bastion ~]# cat clusterstorage.yaml
apiVersion: ocs.openshift.io/v1
kind: StorageCluster
metadata:
  namespace: openshift-storage
  name: ocs-storagecluster
spec:
  manageNodes: false
  monPVCTemplate:
    spec:
      storageClassName: local-sc-mon
      accessModes:

ReadWriteOnce
sources:
requests:
  storage: 10Gi
  resources:
    mon:
      requests: {}
      limits: {}
    mds:
      requests: {}
      limits: {}
    rgw:
      requests: {}
      limits: {}
    mgr:
      requests: {}
      limits: {}
    noobaa-core:
      requests: {}
      limits: {}
    noobaa-db:
      requests: {}
      limits: {}
  storageDeviceSets:

name: ocs-deviceset
count: 1
resources: {}
placement: {}
dataPVCTemplate:
  spec:
    storageClassName: local-sc-osd
    accessModes:

ReadWriteOnce
volumeMode: Block
    resources:
requests:
  storage: 2000Gi
portable: true
replica: 3
```

Now you can monitoring :

root@bastion# oc get pod -n openshift-storage

```
[root@bastion ~]# oc get pod -n openshift-storage
NAME                                                              READY   STATUS                  RESTARTS   AGE
csi-cephfsplugin-2jswn                                            3/3     Running
csi-cephfsplugin-72kzg                                            3/3     Running
csi-cephfsplugin-98m5n                                            3/3     Running
csi-cephfsplugin-l95nc                                            3/3     Running
csi-cephfsplugin-provisioner-77bc9b6b86-h4p4v                     4/4     Running         
csi-cephfsplugin-provisioner-77bc9b6b86-k7x4j                     4/4     Running         
csi-cephfsplugin-sdfkl                                            3/3     Running
csi-rbdplugin-cq6zr                                               3/3     Running
csi-rbdplugin-dhkpw                                               3/3     Running
csi-rbdplugin-dl2hb                                               3/3     Running
csi-rbdplugin-provisioner-546748fccc-4bhzj                        4/4     Running         
csi-rbdplugin-provisioner-546748fccc-rl2tt                        4/4     Running         
csi-rbdplugin-x67kr                                               3/3     Running
csi-rbdplugin-xs6mv                                               3/3     Running
my-operator-catalog-mfnjw                                         1/1     Running
noobaa-core-0                                                     2/2     Running
noobaa-operator-749568949c-qq4f2                                  1/1     Running
ocs-operator-546569ff58-qsxbh                                     1/1     Running
rook-ceph-drain-canary-sto-1.ocp.i3datacenter.com-6df7bf87nr7kv   1/1     Running
rook-ceph-drain-canary-sto-2.ocp.i3datacenter.com-5655bc4fdnkwn   1/1     Running
rook-ceph-drain-canary-sto-3.ocp.i3datacenter.com-65445848vm4qr   1/1     Running
rook-ceph-mds-ocs-storagecluster-cephfilesystem-a-84ddc686m4xz7   1/1     Running
rook-ceph-mds-ocs-storagecluster-cephfilesystem-b-57849df446blj   1/1     Running
rook-ceph-mgr-a-7b84674877-b9lwp                                  1/1     Running
rook-ceph-mon-a-8445555dcd-nbbsv                                  1/1     Running
rook-ceph-mon-b-5497c9678c-kxjnb                                  1/1     Running
rook-ceph-mon-c-749c4c9b6d-vf4wf                                  1/1     Running
rook-ceph-operator-5f87474f49-lgrqb                               1/1     Running
rook-ceph-osd-0-855768d7f9-qw2jf                                  1/1     Running
rook-ceph-osd-1-7b5bf5bf4-48ngn                                   1/1     Running
rook-ceph-osd-2-96dc47f85-54wvq                                   1/1     Running
rook-ceph-osd-prepare-ocs-deviceset-0-0-qjgvt-5tqmd               1/1     Completed       
rook-ceph-osd-prepare-ocs-deviceset-1-0-lldvq-dpbv7               1/1     Completed       
rook-ceph-osd-prepare-ocs-deviceset-2-0-qtjsq-bsbsb               1/1     Completed       
rook-ceph-rgw-ocs-storagecluster-cephobjectstore-a-c6b4588xqzhv   1/1     
```

and you can also in Dashboard :

![](https://raw.githubusercontent.com/h4ckersmooth88/ocs4/master/1.PNG)