# ODF office hours. ODF office hours. can we use SAN FC and ISCSI with ODF?

Clone this repository at "https://gist.github.com/likid0/3947ee4ce22175cb420f48d7830b8f4b.js"
ODF office hours. ODF office hours. can we use SAN FC and ISCSI with ODF?
demo_odf_office_hours_iscsi_config
Pre-Requisites.

We need to have already deployed:
RHEL8 VM deployed
OCP 4.10 cluster.

## Configure Target ISCSI node.

On the VM(Iscsi target node), we are adding 2 VIPs
```
$ nmcli con mod ens192 +ipv4.addresses "10.16.96.100/25,10.16.96.101/25"
```

On OCP 4.10 cluster test connectivity with ISCSI target IPS from Workers
```
$ oc get nodes -l node-role.kubernetes.io/worker --no-headers -o name | xargs -I {} -- oc debug {} -- bash -c 'ping -c1 10.16.96.100; ping -c1 10.16.96.101'
```

Get ISCSI iqn from OCP 4.10 Worker nodes
```
$ oc get nodes -l node-role.kubernetes.io/worker --no-headers -o name | xargs -I {} -- oc debug {} -- bash -c 'chroot /host cat /etc/iscsi/initiatorname.iscsi' 2> /dev/null
InitiatorName=iqn.1994-05.com.redhat:14fdfef4822
InitiatorName=iqn.1994-05.com.redhat:ec30909df335
InitiatorName=iqn.1994-05.com.redhat:e3e5f3827414
```

On the VM(Iscsi target node)
```
$ yum install -y targetcli
$ systemctl start target
$ firewall-cmd --permanent --add-port=3260/tcp
$ firewall-cmd --reload
```

We use 3 100GB disks:
```
$ lsblk | grep disk
sda             8:0    0   50G  0 disk
sdb             8:16   0  100G  0 disk
sdc             8:32   0  100G  0 disk
sdd             8:48   0  100G  0 disk
```

Using TargetCLI, we create a block backingstore with 3 disks
```
$ targetcli
targetcli shell version 2.1.53
Copyright 2011-2013 by Datera, Inc and others.
For help on commands, type 'help'.

/backstores> cd /backstores/block/
/backstores/block>
/backstores/block> create disk0 /dev/sdb
Created block storage object disk0 using /dev/sdb.
/backstores/block> create disk1 /dev/sdc
Created block storage object disk1 using /dev/sdc.
/backstores/block> create disk2 /dev/sdd
Created block storage object disk2 using /dev/sdd.
/backstores/block> ls
o- block ...................................................................................................... [Storage Objects: 3]
  o- disk0 ............................................................................ [/dev/sdb (100.0GiB) write-thru deactivated]
  | o- alua ....................................................................................................... [ALUA Groups: 1]
  |   o- default_tg_pt_gp ........................................................................... [ALUA state: Active/optimized]
  o- disk1 ............................................................................ [/dev/sdc (100.0GiB) write-thru deactivated]
  | o- alua ....................................................................................................... [ALUA Groups: 1]
  |   o- default_tg_pt_gp ........................................................................... [ALUA state: Active/optimized]
  o- disk2 ............................................................................ [/dev/sdd (100.0GiB) write-thru deactivated]
    o- alua ....................................................................................................... [ALUA Groups: 1]
      o- default_tg_pt_gp ........................................................................... [ALUA state: Active/optimized]

```

We are now going to create the IQN for our iscsi target
```
/backstores/block> cd /iscsi
/iscsi> create
Created target iqn.2003-01.org.linux-iscsi.bos-iscsi.x8664:sn.4a4331f926cb.
Created TPG 1.
Global pref auto_add_default_portal=true
Created default portal listening on all IPs (0.0.0.0), port 3260.
/iscsi> ls
o- iscsi .............................................................................................................. [Targets: 1]
  o- iqn.2003-01.org.linux-iscsi.bos-iscsi.x8664:sn.4a4331f926cb ......................................................... [TPGs: 1]
    o- tpg1 ................................................................................................. [no-gen-acls, no-auth]
      o- acls ............................................................................................................ [ACLs: 0]
      o- luns ............................................................................................................ [LUNs: 0]
      o- portals ...................................................................................................... [Portals: 1]
        o- 0.0.0.0:3260 ....................................................................................................... [OK]

/iscsi> cd iqn.2003-01.org.linux-iscsi.bos-iscsi.x8664:sn.4a4331f926cb/tpg1/acls 
```

Using the Iqns we got from the worker nodes, we will configure ACLs in the target iscsi server so only the 3 worker nodes can access the LUNs.
```
/iscsi/iqn.20...6cb/tpg1/acls> create iqn.1994-05.com.redhat:14fdfef4822
Created Node ACL for iqn.1994-05.com.redhat:14fdfef4822
/iscsi/iqn.20...6cb/tpg1/acls> create iqn.1994-05.com.redhat:ec30909df335
Created Node ACL for iqn.1994-05.com.redhat:ec30909df335
/iscsi/iqn.20...6cb/tpg1/acls> create iqn.1994-05.com.redhat:e3e5f3827414
Created Node ACL for iqn.1994-05.com.redhat:e3e5f3827414
```

Using the backing store disks created on a previous step we will create 3 luns associated with the Iqn we created, one lun for each disk, out of the box the 3 luns will be accessible to each server.  
```
/iscsi/iqn.20...6cb/tpg1/acls> cd /iscsi/iqn.2003-01.org.linux-iscsi.bos-iscsi.x8664:sn.4a4331f926cb/tpg1/luns
/iscsi/iqn.20...6cb/tpg1/luns> create /backstores/block/disk0 lun0
Created LUN 0.
Created LUN 0->0 mapping in node ACL iqn.1994-05.com.redhat:e3e5f3827414
Created LUN 0->0 mapping in node ACL iqn.1994-05.com.redhat:ec30909df335
Created LUN 0->0 mapping in node ACL iqn.1994-05.com.redhat:14fdfef4822
/iscsi/iqn.20...6cb/tpg1/luns> create /backstores/block/disk1 lun1
Created LUN 1.
Created LUN 1->1 mapping in node ACL iqn.1994-05.com.redhat:e3e5f3827414
Created LUN 1->1 mapping in node ACL iqn.1994-05.com.redhat:ec30909df335
Created LUN 1->1 mapping in node ACL iqn.1994-05.com.redhat:14fdfef4822
/iscsi/iqn.20...6cb/tpg1/luns> create /backstores/block/disk2 lun2
Created LUN 2.
Created LUN 2->2 mapping in node ACL iqn.1994-05.com.redhat:e3e5f3827414
Created LUN 2->2 mapping in node ACL iqn.1994-05.com.redhat:ec30909df335
Created LUN 2->2 mapping in node ACL iqn.1994-05.com.redhat:14fdfef4822

/iscsi/iqn.20...t:14fdfef4822> cd /iscsi/iqn.2003-01.org.linux-iscsi.bos-iscsi.x8664:sn.4a4331f926cb/tpg1/acls/iqn.1994-05.com.redhat:14fdfef4822
/iscsi/iqn.20...t:14fdfef4822> ls
o- iqn.1994-05.com.redhat:14fdfef4822 ............................................................................. [Mapped LUNs: 3]
  o- mapped_lun0 ........................................................................................... [lun0 block/disk0 (rw)]
  o- mapped_lun1 ........................................................................................... [lun1 block/disk1 (rw)]
  o- mapped_lun2 ........................................................................................... [lun2 block/disk2 (rw)]
```

We only want one dedicated 100GB Lun per worker nodes, so using ACLs we will leave only one LUN per node, for that we need to delete the extra luns from each worker Iqn.
```
/iscsi/iqn.20...t:14fdfef4822> delete 1
Deleted Mapped LUN 1.
/iscsi/iqn.20...t:14fdfef4822> delete 2
Deleted Mapped LUN 2.

/iscsi/iqn.20...t:14fdfef4822> cd /iscsi/iqn.2003-01.org.linux-iscsi.bos-iscsi.x8664:sn.4a4331f926cb/tpg1/acls/iqn.1994-05.com.redhat:ec30909df335
/iscsi/iqn.20...:ec30909df335> delete 0
Deleted Mapped LUN 0.
/iscsi/iqn.20...:ec30909df335> delete 2
Deleted Mapped LUN 2.

/iscsi/iqn.20...t:14fdfef4822> cd /iscsi/iqn.2003-01.org.linux-iscsi.bos-iscsi.x8664:sn.4a4331f926cb/tpg1/acls/iqn.1994-05.com.redhat:14fdfef4822
/iscsi/iqn.20...6cb/tpg1/acls> cd iqn.1994-05.com.redhat:e3e5f3827414
/iscsi/iqn.20...:e3e5f3827414> ls
o- iqn.1994-05.com.redhat:e3e5f3827414 ............................................................................ [Mapped LUNs: 3]
  o- mapped_lun0 ........................................................................................... [lun0 block/disk0 (rw)]
  o- mapped_lun1 ........................................................................................... [lun1 block/disk1 (rw)]
  o- mapped_lun2 ........................................................................................... [lun2 block/disk2 (rw)]
/iscsi/iqn.20...:e3e5f3827414> delete 0
Deleted Mapped LUN 0.
/iscsi/iqn.20...:e3e5f3827414> delete 1
Deleted Mapped LUN 1.
```

Once we have finalized the configuration of the target iscsi server we run a saveconfig command: 
```
/iscsi/iqn.20...:e3e5f3827414> cd /
/> saveconfig
Last 10 configs saved in /etc/target/backup/.
Configuration saved to /etc/target/saveconfig.json
/>
```

## Configure ISCSI initiator and Multipath on OCP workers.

We are first going to discover the target cli portals at ips: 10.16.96.100 and 10.16.96.101. Then we will log in to each portal using the iscsiadm command
```
$ oc get nodes -l node-role.kubernetes.io/worker --no-headers -o name | xargs -I {} -- oc debug {} -- bash -c 'chroot /host iscsiadm -m discovery -t st -p 10.16.96.100 ; chroot /host iscsiadm -m discovery -t st -p 10.16.96.101 ; chroot /host iscsiadm --mode node --target iqn.2003-01.org.linux-iscsi.bos-iscsi.x8664:sn.4a4331f926cb --portal 10.16.96.100 -l; chroot /host iscsiadm --mode node --target iqn.2003-01.org.linux-iscsi.bos-iscsi.x8664:sn.4a4331f926cb --portal 10.16.96.101 -l' 2> /dev/null
```

We will now use the iscsiadm -m session command to check that each worker sees one Lun from 2 different paths:
```
$ oc get nodes -l node-role.kubernetes.io/worker --no-headers -o name | xargs -I {} -- oc debug {} -- bash -c "chroot /host uname -a ; chroot /host iscsiadm -m session -P 3 | grep -E '(Persistent Portal|Lun|Attached)'" 2> /dev/null
```

### machine config object

At this point, we have access to the luns/disks on the worker nodes, but if we reboot the worker nodes we will lose access to the disks because iscsid won’t be started on boot, also we still need to configure the multipath configuration.

We will use a machine config object to start the needed services on boot for iscsi to work and also set the multipath configuration.

The multipath configuration file we are using is just a default configuration file, each storage vendor has certain recommended options for configuring multipath for their storage appliances.
```
$ cat <<EOF > multipath.conf
defaults {
    user_friendly_names yes
    find_multipaths yes
    enable_foreign "^$"
}

blacklist_exceptions {
        property "(SCSI_IDENT_|ID_WWN)"
}

blacklist {
}
EOF
```

As we are going to use MC with ignition we convert the multipath.conf config file into base64
```
$ cat multipath.conf|base64 -w0
ZGVmYXVsdHMgewogICAgdXNlcl9mcmllbmRseV9uYW1lcyB5ZXMKICAgIGZpbmRfbXVsdGlwYXRocyB5ZXMKICAgIGVuYWJsZV9mb3JlaWduICJeJCIKfQoKYmxhY2tsaXN0X2V4Y2VwdGlvbnMgewogICAgICAgIHByb3BlcnR5ICIoU0NTSV9JREVOVF98SURfV1dOKSIKfQoKYmxhY2tsaXN0IHsKfQo=%
```

We then create the machine config object, we use labels so the change will only happen on the nodes with the worker role: 
```
$ cat <<EOF > worker-mc-iscsi.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: 90-worker-iscsi-multipathing
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      files:
      - contents:
         source: data:text/plain;charset=utf-8;base64,ZGVmYXVsdHMgewogICAgdXNlcl9mcmllbmRseV9uYW1lcyB5ZXMKICAgIGZpbmRfbXVsdGlwYXRocyB5ZXMKICAgIGVuYWJsZV9mb3JlaWduICJeJCIKfQoKYmxhY2tsaXN0X2V4Y2VwdGlvbnMgewogICAgICAgIHByb3BlcnR5ICIoU0NTSV9JREVOVF98SURfV1dOKSIKfQoKYmxhY2tsaXN0IHsKfQo=
        filesystem: root
        mode: 420
        path: /etc/multipath.conf
    systemd:
      units:
      - name: iscsid.service
        enabled: true
      - name: iscsi.service
        enabled: true
      - name: multipathd.service
        enabled: true
EOF
```

```
$ oc create -f worker-mc-iscsi.yaml
machineconfig.machineconfiguration.openshift.io/90-worker-iscsi-multipathing created
```

We can double-check that the MC has been created, and the worker MCP has started rolling out the changes to the workers 
```
$ oc get mc | grep iscs
90-worker-iscsi-multipathing                                                                  3.2.0             2m57s

$ oc get nodes
NAME                      STATUS                     ROLES    AGE   VERSION
bos3-5zhw9-master-0       Ready                      master   75m   v1.24.0+cb71478
bos3-5zhw9-master-1       Ready                      master   76m   v1.24.0+cb71478
bos3-5zhw9-master-2       Ready                      master   75m   v1.24.0+cb71478
bos3-5zhw9-worker-cmhcb   Ready,SchedulingDisabled   worker   58m   v1.24.0+cb71478
bos3-5zhw9-worker-lmkvs   Ready                      worker   58m   v1.24.0+cb71478
bos3-5zhw9-worker-q6zzq   Ready                      worker   52m   v1.24.0+cb71478

$ oc get mcp | grep -v master
NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
worker   rendered-worker-808ab45b6e4943f79b6708e90993553d   False     True       False      3              0                   0                     0                      71m
```

Once the worker nodes finish rolling out the changes we can see that multipath is configured and operational, We are working in Active/Passive mode:

```
$ oc get no -l node-role.kubernetes.io/worker --no-headers -o name | xargs -I {} -- oc debug {} -- bash -c 'chroot /host echo "###  HOSTNAME: $(uname -n)  ###" ; chroot /host multipath -ll' 2> /dev/null
###  HOSTNAME: bos3-nkfb7-worker-gfkqn  ###
mpatha (36001405ceb3e0af31a842efafd183706) dm-0 LIO-ORG,disk0
size=100G features='0' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 4:0:0:0 sdc 8:32 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 3:0:0:0 sdb 8:16 active ready running
###  HOSTNAME: bos3-nkfb7-worker-hzsgx  ###
mpatha (360014057bb363ac21fe437691dbfde4b) dm-0 LIO-ORG,disk1
size=100G features='0' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 3:0:0:1 sdb 8:16 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 4:0:0:1 sdc 8:32 active ready running
###  HOSTNAME: bos3-nkfb7-worker-vwrdd  ###
mpatha (36001405e21579190e8d4d6fa2c454ab1) dm-0 LIO-ORG,disk2
size=100G features='0' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 3:0:0:2 sdb 8:16 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 4:0:0:2 sdc 8:32 active ready running
```

## Configure LSO using Localvolume

We will deploy the LSO operator using the CLI:
```
$ cat <<EOF > namespace-lso.yaml
---
apiVersion: v1
kind: Namespace
metadata:
 name: openshift-local-storage
spec: {}
EOF

$ oc create -f namespace-lso.yaml


$ cat <<EOF >local-operator-group-lso.yaml
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
 name: local-operator-group
 namespace: openshift-local-storage
spec:
 targetNamespaces:
 - openshift-local-storage
EOF

$ oc create -f local-operator-group-lso.yaml

$ cat <<EOF > sub-lso.yaml
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
 name: local-storage-operator
 namespace: openshift-local-storage
spec:
 channel: "4.10"
 installPlanApproval: Automatic
 name: local-storage-operator
 source: redhat-operators
 sourceNamespace: openshift-marketplace
EOF

$ oc create -f sub-lso.yaml
```

Once the LSO operator pod is running we can create the local volume object to configure LSO against our multipath devices. 
We are setting the node selector to use all nodes that have the openshift-storage label set, and create a new storage class with all devices with path /dev/mapper/mpatha: 

Set label to a node
```
oc label node nodename \
  cluster.ocs.openshift.io/openshift-storage=''
[user@host ~]$ oc get nodes -l \
  cluster.ocs.openshift.io/openshift-storage=""
```

LocalVoluem definition with selector
```
$ cat <<EOF > local-volume.yaml
---
apiVersion: local.storage.openshift.io/v1
kind: LocalVolume
metadata:
  name: local-block
  namespace: openshift-local-storage
spec:
  nodeSelector:
    nodeSelectorTerms:
    - matchExpressions:
        - key: cluster.ocs.openshift.io/openshift-storage
          operator: In
          values:
          - ""
  storageClassDevices:
    - storageClassName: localblock
      volumeMode: Block
      devicePaths:
        - /dev/mapper/mpatha
EOF
```

After creating the Local Volume object we can check if we have the disk maker pods running, one per storage node.
```
$ oc get pods -o wide
NAME                                      READY   STATUS    RESTARTS       AGE    IP           NODE                      NOMINATED NODE   READINESS GATES
diskmaker-manager-h6vj7                   2/2     Running   0              4m1s   10.19.10.8   bos3-zk262-worker-dc79r   <none>           <none>
diskmaker-manager-hjm8s                   2/2     Running   0              4m1s   10.19.8.15   bos3-zk262-worker-gkxt2   <none>           <none>
diskmaker-manager-qhsp5                   2/2     Running   1 (3m5s ago)   4m1s   10.19.6.13   bos3-zk262-worker-rr5sx   <none>           <none>
local-storage-operator-787dfbfd9c-b52zl   1/1     Running   0              11m    10.19.6.12   bos3-zk262-worker-rr5sx   <none>           <none>
```

3 PVs get created with a size of 100GBs
```
$ oc get pv
NAME                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
local-pv-42f3ac9f   100Gi      RWO            Delete           Available           localblock              9m48s
local-pv-88e8fb4e   100Gi      RWO            Delete           Available           localblock              9m48s
local-pv-d3853c97   100Gi      RWO            Delete           Available           localblock              9m8s
```

Using the multipath device mpatha
```
$ oc get pv -o yaml | grep mpat
      storage.openshift.com/device-id: dm-name-mpatha
      path: /mnt/local-storage/localblock/dm-name-mpatha
      storage.openshift.com/device-id: dm-name-mpatha
      path: /mnt/local-storage/localblock/dm-name-mpatha
      storage.openshift.com/device-id: dm-name-mpatha
      path: /mnt/local-storage/localblock/dm-name-mpatha
```

Creating the Storage Class localblock that we will use for our ODF deployment
```
$ oc get sc
NAME             PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
localblock       kubernetes.io/no-provisioner   Delete          WaitForFirstConsumer   false                  8m26s
thin (default)   kubernetes.io/vsphere-volume   Delete          Immediate              false                  74m
thin-csi         csi.vsphere.vmware.com         Delete          WaitForFirstConsumer   true                   69m
```

## Deploy ODF using the localblock SC

## Validate Multipath configuration

We are going to fail a path from the ISCSI target and observe how it affects a stateful application consuming a CephFS volume.

To verify the app behaviour during a failure we will use the IO load generator available on the ocs-ha-tests git repo.

$ git clone https://gitlab.consulting.redhat.com/iberia-consulting/inditex/ocs/ocs-ha-tests.git
$ oc project loadgen
$ cd ocs-ha-tests
$ helm install odf-loadgenerator helm

Once the pod is running and writing to the filesystem we will fail a path, on the ISCS target node, we remove a portal IP:
```
# nmcli con mod ens192 -ipv4.addresses "10.16.96.100/25"
# nmcli con up ens192
```

First thing to check is that we have a failed path on the multipath -ll output, and that there has been a failover of the Active path:
```
oc get no -l node-role.kubernetes.io/worker --no-headers -o name | xargs -I {} -- oc debug {} -- bash -c 'chroot /host echo "###  HOSTNAME: $(uname -n)  ###" ; chroot /host multipath -ll' 2> /dev/null
###  HOSTNAME: bos3-nkfb7-worker-gfkqn  ###
mpatha (36001405ceb3e0af31a842efafd183706) dm-0 LIO-ORG,disk0
size=100G features='0' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 4:0:0:0 sdc 8:32 active ready running
`-+- policy='service-time 0' prio=0 status=enabled
  `- 3:0:0:0 sdb 8:16 failed faulty running
```
