Objective
---------

Enable **shared persistent storage across multiple users and namespaces** in **Red Hat OpenShift Dev Spaces (DevSpace)** by using a **CIFS/SMB-backed RWX volume** hosted on NAS storage.  
This shared storage is automatically mounted into DevSpaces workspaces using PVC annotations.

* * *

Architecture Overview
---------------------

*   **Storage Backend**: NAS (CIFS/SMB share)
    
*   **CSI Driver**: SMB CSI (`smb.csi.k8s.io`)
    
*   **Access Mode**: `ReadWriteMany (RWX)`
    
*   **Use Case**: Shared workspace directory across DevSpaces users
    
*   **Mount Path in Workspace**: `/shared`
    
*   **Provisioning Model**:
    *   Static PersistentVolumes (PVs)
        
    *   One PVC per namespace
        
    *   All PVs point to the **same NAS subdirectory**
        

* * *

References
----------

*   OpenShift SMB CSI configuration  
    [https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/storage/using-container-storage-interface-csi#persistent-storage-csi-smb-cifs](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/storage/using-container-storage-interface-csi#persistent-storage-csi-smb-cifs)
    
*   DevSpaces PVC auto-mounting  
    [https://docs.redhat.com/en/documentation/red_hat_openshift_dev_spaces/3.25/html/user_guide/requesting-persistent-storage-for-workspaces#requesting-persistent-storage-for-workspaces-requesting-persistent-storage-in-a-pvc](https://docs.redhat.com/en/documentation/red_hat_openshift_dev_spaces/3.25/html/user_guide/requesting-persistent-storage-for-workspaces#requesting-persistent-storage-for-workspaces-requesting-persistent-storage-in-a-pvc)
    

* * *

High-Level Steps
----------------

1.  Install and configure the **SMB CSI driver** (CIFS Samba Operator)
    
2.  Create a **shared directory** on NAS storage
    
3.  Create **static PersistentVolumes** pointing to the same NAS subdirectory
    
4.  Create **namespace-scoped PVCs** bound to those PVs
    
5.  Annotate PVCs so DevSpaces automatically mounts them into workspaces
    
6.  Validate shared access across namespaces
    

* * *

Step 1: Install and Configure SMB CSI Driver
--------------------------------------------

*   Deploy **CIFS Samba Operator**
    
*   Configure SMB credentials as a Kubernetes Secret (`smbcreds`) in the `samba` namespace
    
*   Ensure the SMB CSI driver (`smb.csi.k8s.io`) is available cluster-wide
    

* * *

Step 2: Create Shared Directory on NAS
--------------------------------------

Example:

`\\nas.example.com\temporary\shared-pv`

This directory will be **shared across all DevSpaces users and namespaces**.

* * *

Step 3: Create Shared PersistentVolumes (Static Provisioning)
-------------------------------------------------------------

Each namespace uses a **dedicated PV**, but all PVs reference the **same NAS subdirectory**.

### PersistentVolume: `shared-pv`

```
kind: PersistentVolume
apiVersion: v1
metadata:
  name: shared-pv
  annotations:
    pv.kubernetes.io/provisioned-by: smb.csi.k8s.io
    volume.kubernetes.io/provisioner-deletion-secret-name: smbcreds
    volume.kubernetes.io/provisioner-deletion-secret-namespace: samba
spec:
  capacity:
    storage: 3Gi
  csi:
    driver: smb.csi.k8s.io
    volumeHandle: nas.example.com/temporary#shared-pv##
    volumeAttributes:
      csi.storage.k8s.io/pv/name: shared-pv
      csi.storage.k8s.io/pvc/name: shared
      csi.storage.k8s.io/pvc/namespace: user1-devspaces
      source: //nas.example.com/temporary
      subdir: shared-pv
    nodeStageSecretRef:
      name: smbcreds
      namespace: samba
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: samba
  mountOptions:
    - dir_mode=0777
    - file_mode=0777
    - uid=1001
    - gid=1001
  volumeMode: Filesystem
```
* * *

### PersistentVolume: `shared-pv2`

> Note: Points to the **same NAS subdirectory** (`shared-pv`)
```
kind: PersistentVolume
apiVersion: v1
metadata:
  name: shared-pv2
  annotations:
    pv.kubernetes.io/provisioned-by: smb.csi.k8s.io
    volume.kubernetes.io/provisioner-deletion-secret-name: smbcreds
    volume.kubernetes.io/provisioner-deletion-secret-namespace: samba
spec:
  capacity:
    storage: 3Gi
  csi:
    driver: smb.csi.k8s.io
    volumeHandle: nas.example.com/temporary#shared-pv##
    volumeAttributes:
      csi.storage.k8s.io/pv/name: shared-pv2
      csi.storage.k8s.io/pvc/name: shared
      csi.storage.k8s.io/pvc/namespace: user2-devspaces
      source: //nas.example.com/temporary
      subdir: shared-pv
    nodeStageSecretRef:
      name: smbcreds
      namespace: samba
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: samba
  mountOptions:
    - dir_mode=0777
    - file_mode=0777
    - uid=1001
    - gid=1001
  volumeMode: Filesystem

```


* * *

Step 4: Create PVCs per Namespace
---------------------------------

Each namespace has:
*   One PVC
    
*   Bound to its dedicated PV
    
*   Annotated for **automatic DevSpace mounting**
    

* * *

### Namespace: `user1-devspaces`

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  annotations:
    controller.devfile.io/mount-path: /shared
  name: shared
  labels:
    controller.devfile.io/mount-to-devworkspace: 'true'
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 3Gi
  volumeName: shared-pv
  storageClassName: samba
  volumeMode: Filesystem
```

* * *

### Namespace: `user2-devspaces`

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  annotations:
    controller.devfile.io/mount-path: /shared
  name: shared
  labels:
    controller.devfile.io/mount-to-devworkspace: 'true'
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 3Gi
  volumeName: shared-pv2
  storageClassName: samba
  volumeMode: Filesystem
```

* * *

Step 5: DevSpaces Auto-Mount Behavior
-------------------------------------

Because of the following metadata:
```
`labels:
   controller.devfile.io/mount-to-devworkspace: "true" 
 annotations:
   controller.devfile.io/mount-path: /shared`
```
*   The PVC is **automatically mounted**
    
*   Available inside the DevSpace container at:
    
    `/shared`

Step 6: Testing / Validation:
Namespace user2-devspaces:
```
oc exec workspacedf96f9758b55491f-849cf676df-7r8cn \
  -c universal-developer-image -n user2-devspaces -- ls /shared
Output:
testfile111
testfile222

```

Namespace user1-devspaces:
```
oc exec workspacef27ef0db39d740fb-6559dbc5df-f8w6t \
  -c tools -n user1-devspaces -- ls /shared
Output:
testfile111
testfile222

```
Key Considerations:
*   RWX is provided by SMB
    
*   Reclaim policy Retain ensures data persistence
    
*   NAS permissions must allow multi-user access
    
*   UID/GID mapping (1001) aligns with DevSpaces containers
    
*   Each namespace requires its own PV and PVC
