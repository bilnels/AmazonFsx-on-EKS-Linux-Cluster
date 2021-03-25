# Using Amazon FSx for Windows File Server on EKS Linux Cluster
Currently, there is no existing Amazon Fsx for Windows File server Container Storage Interface (CSI ) Driver that provides a CSI interface that allows Amazon EKS clusters in Linux to manage the lifecycle of Amazon FSx for Windows File server file systems. Today, we are going to walk through a step-by-step process on how to deploy *csi-driver-smb* CSI Driver to allow using Amazon FSx for Windows File Server as persistent storage for Linux containers running on running on Amazon Elastic Kubernetes Service (EKS) and verify that it works. We recommend using latest version [CSI Driver](https://github.com/kubernetes-csi/csi-driver-smb) of the driver.


### To deploy the csi-driver-smb CSI driver to an Amazon EKS Linux cluster:

1. Install the CSI Driver
   Using helm, install latest stable version of the driver:
   ```
   $ helm repo add csi-driver-smb https://raw.githubusercontent.com/kubernetes-csi/csi-driver-smb/master/charts
   $ helm install csi-driver-smb csi-driver-smb/csi-driver-smb --namespace kube-system --version v0.6.0
   ```
   **Note:** Above command will install all the required Kubernetes resources to get the driver running: ServiceAccount, ClusterRole, ClusterRoleBinding, DaemonSet, Deployment, and CSI Driver

2. Create a Kubernetes secret with credential information to mount the Amazon FSx volume for Windows File Server
   ```
   kubectl create secret generic smbcreds --from-literal username=admin --from-literal password="XXXXXXX"
   ```
   Output:
   ```
   $ kubectl describe secrets smbcreds
   Name:         smbcreds
   Namespace:    default
   Labels:       <none>
   Annotations:  <none>
   Type:  Opaque
   
   Data
   ====
   password:  8 bytes
   username:  5 bytes
   ```
   The above are Active Directory Credentials used to access the filesystem for FSx for Windows File Serve
   You can create as many secrets as the number of users within you cluster required to mount the different volumes of the FsX
   
3. Create a Storage class resource/ Persistent Volume
   ```
   $ cat pv-smb.yaml 
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: pv-smb
    spec:
      capacity:
        storage: 100Gi
      accessModes:
        - ReadWriteMany
      persistentVolumeReclaimPolicy: Retain
      mountOptions:
        - dir_mode=0777
        - file_mode=0777
        - vers=3.0
      csi:
        driver: smb.csi.k8s.io
        readOnly: false
        volumeHandle: unique-volumeid  # make sure it's a unique id in the cluster
        volumeAttributes:
          source: "//<FSx Server IP>/share" #Fsx IP address
        nodeStageSecretRef:
          name: smbcreds
          namespace: default
    ```
4. Using the Persistent Volume Claim
   ```
    cat pvc-smb.yaml  
    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: pvc-smb
    spec:
      accessModes:
        - ReadWriteMany
      resources:
        requests:
          storage: 10Gi
      volumeName: pv-smb
      storageClassName: ""
     ```

5. Confirm that the file system is provisioned
   ```
   $kubectl get pv
   NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS   REASON   AGE
   pv-smb   100Gi      RWX            Retain           Bound    default/pvc-smb                           3h41m
   ```
   ```
   $ kubectl get pvc
   NAME      STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
   pvc-smb   Bound    pv-smb   100Gi      RWX                           3h36m
   ```

6. Deploy the sample application to verify that the CSI driver is working
   ```
    $ cat pv-smb-pod.yaml 
    apiVersion: v1
    kind: Pod
    metadata:
      name: task-pv-pod
    spec:
      volumes:
        - name: smb-pv-storage
          persistentVolumeClaim:
            claimName: pvc-smb
      containers:
        - name: pv-smb-container
          image: nginx
          ports:
            - containerPort: 80
              name: "http-server"
          volumeMounts:
            - mountPath: "/usr/share/nginx/html"
              name: smb-pv-storage
    ```
7. Verify that the sample application is running and Amazon Fsx volume is correctly mounted
   ```
    kubectl exec task-pv-pod -it  bash     
    root@task-pv-pod:/# cd /usr/share/nginx/html/
    root@task-pv-pod:/usr/share/nginx/html# echo "Testing volume mount" >test.html
    root@task-pv-pod:/usr/share/nginx/html# ls -l
    total 17
    -rwxrwxrwx 1 root root 33 Feb 23 06:38 'Fsx Hello world.txt'
    drwxrwxrwx 2 root root  0 Feb 23 06:40  deployment
    -rwxrwxrwx 1 root root 21 Feb 23 07:28  test.html
   ```
   
8. Confirm the mount and created sample files are accessible within the Fsx windows server

   ![image](https://user-images.githubusercontent.com/6791436/112352107-67537980-8cc2-11eb-94b6-7b1c8c2f086f.png)

# Conclusion

Following the above steps, you will be able to configure your EKS Linux Cluster pods to mount a volume through an SMB share hosted in Amazon FSx for Windows File Server. The approach of data persistence will be suitable for legacy example ASP.NET applications which provides data other cloud native apps are dependant on.

# Cleanup
Once completed, make sure to delete the resources created post-testing

