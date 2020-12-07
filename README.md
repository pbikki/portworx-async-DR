# portworx-async-DR
Portworx Async DR PoC setup on IBM Cloud managed OpenShift Cluster

This work will provide details about the setup and detailed steps required for a Portworx Asynchronous DR setup on IBM Cloud ROKS VPC gen2 environment. This setup is used for DR when clusters are in different regions. 
Before proceeding, review [Portworx docs for async DR setup](https://docs.portworx.com/portworx-install-with-kubernetes/disaster-recovery/)

> Disclaimer: Please note that the steps cover the portworx async DR setup for a test or dev setup and this document should only be used a reference. This is not an official guide. Real environments might need additional configuration and a more secure setup.


## Prerequisites
- Review the [Prereqs](https://docs.portworx.com/portworx-install-with-kubernetes/disaster-recovery/async-dr/#prerequisites) from portworx docs
- Two ROKS clusters in different regions
- Portworx installations v2.1 or later on **source and target ROKS clusters**. Portworx Enterprise with Disaster Recovery (DR) can be installed  through [IBM Cloud Catalog](https://cloud.ibm.com/catalog/services/portworx-enterprise)
    - Each Portworx uses its own ICD ETCD database
    - KeyProtect instance in each region to store root key used to configure secret store for each portworx cluster
- If portworx is setup to use IBM Key Protect, both clusters should use the same KP root key. This can achieved by importing the same root key material into 2 KP instances configured to be used with 2 Portworx clusters. Refer section for steps [KeyProtect setup for portworx for async DR](#keyProtect-setup-for-portworx)

- Portworx expects some ports to be open for connectivity between two clusters. For this PoC setup, ROKS clusters are in different VPCs. A security rule to open TCP ports  `17001-17020` was added to default security group of destination cluster's VPC
    > Caution: Please take a look at the `portworx-service` only open ports required for your env. 
    > Reference - [Opening the required ports in the console](https://cloud.ibm.com/docs/containers?topic=containers-vpc-network-policy#security_groups_ui)

- [Create and configure](https://cloud.ibm.com/interconnectivity/transit) Transit gateway to allow network traffic flow between clusters in different regions and VPCs. 
   > Reference - [Getting started with IBM Cloud Transit Gateway](https://cloud.ibm.com/docs/transit-gateway/getting-started.html?locale=en)

- Create a COS instance. 
    - A bucket can be pre-created and associated with cloud creds that'll be created later in this doc
    - Or, Portworx can create bucket automatically

- [oc/kubectl](https://cloud.ibm.com/docs/openshift?topic=openshift-openshift-cli#cli_oc) CLI
- [storkctl](https://docs.portworx.com/portworx-install-with-kubernetes/disaster-recovery/async-dr/#prerequisites)

## Setup used here
- Gen2 VPCs in `us-south` and `us-east`
- VPC ROKS clusters in `us-south` (source region) and `us-east` (target region)
- [Portworx installed](https://cloud.ibm.com/docs/openshift?topic=openshift-portworx) on both clusters
    - ICD ETCD database instances in `us-south` and `us-east`
    - Block volumes attached to ROKS workers (workers that will be part of portworx storage cluster)
- KeyProtect instances `us-south` and `us-east`

### KeyProtect setup for portworx

When you setup [Portworx with volume encryption using IBM KeyProtect](https://cloud.ibm.com/docs/openshift?topic=openshift-portworx#encrypt_volumes), a root key will be used.

In async DR setup, as we have two clusters - active and standby; and volumes are migrated from active to standby cluster, portworx can be configured to use the customer root key (CRK) that is stored in KeyProtect instance in both regions. For more info on how KeyProtect works with Portworx, refer [Portworx per-volume encryption workflow](https://cloud.ibm.com/docs/openshift?topic=openshift-portworx#px_encryption)

- Create/Use KP instance in `us-south`
- Create/Use KP instance in `us-east`
- Import same Customer Root Key (CRK) into KP instances in both regions
  Refer [import-root-key](https://cloud.ibm.com/docs/key-protect?topic=key-protect-import-root-keys#import-root-key-gui)

  In normal cases, this will be a BYOK from customer. For PoC purposes, you can generate a random [base64 key material using openssl](https://cloud.ibm.com/docs/key-protect?topic=key-protect-import-root-keys#open-ssl-encoding-root-new-key-material)

  Eg: 
  ```
  $ openssl rand -base64 32
  ```
- Once you import key using the provided or generated key material, the root keys ids and other KP details can be used to create a secret tat will be referenced by Portworx to encrypt volumes. Steps are documented [here](https://cloud.ibm.com/docs/openshift?topic=openshift-portworx#setup_encryption)

## High-level steps
- Verify Portworx installation
- Create cloud credentials on target cluster
- Generate a clusterpair yaml from target cluster
- Edit the clusterpair yaml to point to the destinationclusters portworx service
- Create cloud credentials on source cluster
- Create `ClusterPair` object on source cluster making use of clusterpair yaml
- Create a migration namespace and a test app to test migration
- Create a schedule policy to specify when to schedule a migration
- Create policy to specify the migration details and create migration objects
- Monitor migration status
- Verify resources are migrated in the target/standby  cluster

## Verify portworx
Verify that portworx is operational and has the expected License on both source and target clusters
```
▶ PX_POD=$(kubectl get pods -l name=portworx -n kube-system -o jsonpath='{.items[0].metadata.name}')
▶ kubectl exec $PX_POD -n kube-system -- /opt/pwx/bin/pxctl status
Status: PX is operational
  License: PX-Enterprise IBM Cloud DR (expires in 1356 days)
  Node ID: <id>
  	IP: <ip1>
  	Local Storage Pool: 1 pool
  	POOL	IO_PRIORITY	RAID_LEVEL	USABLE	USED	STATUS	ZONE		REGION
  	0	LOW		raid0		128 GiB	8.4 GiB	Online	us-south-1	us-south
  	Local Storage Devices: 1 device
  	Device	Path		Media Type		Size		Last-Scan
  	0:1	/dev/vdd	STORAGE_MEDIUM_MAGNETIC	128 GiB		17 Jul 20 14:40 UTC
  	total			-			128 GiB
  	Cache Devices:
  	No cache devices
  Cluster Summary
  	Cluster ID: px-storage-cluster
  	Cluster UUID: <pwx-cluster-id>
  	Scheduler: kubernetes
  	Nodes: 3 node(s) with storage (3 online)
  	IP	ID	SchedulerNodeName StorageNode Used Capacity  Status StorageStatus	Version		Kernel		OS
  	<ip1>	<node>	<ip1>		Yes		8.4 GiB	128 GiB		Online	Up		2.5.2.0-176ddb7	3.10.0-1127.13.1.el7.x86_64	Red Hat
  	<ip2>	<node>	<ip2>		Yes		8.4 GiB	128 GiB		Online	Up		2.5.2.0-176ddb7	3.10.0-1127.13.1.el7.x86_64	Red Hat
  	<ip3>	<node>	<ip3>		Yes		8.4 GiB	128 GiB		Online	Up (This node)	2.5.2.0-176ddb7	3.10.0-1127.13.1.el7.x86_64	Red Hat
  	Warnings: 
  		 WARNING: Persistent journald logging is not enabled on this node.
  Global Storage Pool
  	Total Used    	:  25 GiB
  	Total Capacity	:  384 GiB
```


## On target cluster

[Login to the target cluster](https://cloud.ibm.com/docs/openshift?topic=openshift-access_cluster#access_public_se) 

### Create cloud credentials
Create object store creds on portworx. This step should be done on source cluster as well before creating a clusterpair (to follow)

> Refer [portworx docs](https://docs.portworx.com/portworx-install-with-kubernetes/disaster-recovery/async-dr/#create-object-store-credentials-for-cloud-clusters) for more details 

1. Get `UUID` of the target cluster
    
    ```
    ▶ PX_POD=$(kubectl get pods -l name=portworx -n kube-system -o jsonpath='{.items[0].metadata.name}')

    ▶ kubectl exec $PX_POD -n kube-system -- /opt/pwx/bin/pxctl status | grep UUID | awk '{print $3}'                            
    82812475-a501-4061-89e5-b85490892063
    ```

2. Create cloud creds

    In this setup, a bucket  was pre-created in `us-geo` and the bucket is specified below. You can let Portworx will create a bucket without specifying it below. 

    ```
    /opt/pwx/bin/pxctl credentials create --provider s3 --s3-access-key <access-key> --s3-secret-key <secret-key> --bucket <bucket-name> --s3-region us-geo --s3-endpoint s3.direct.us.cloud-object-storage.appdomain.cloud  clusterpair_<UUID_of_destination_cluster>
    ```

### Portworx service

 > **Caution**: In the below example, as this for poc setup, portworx service is exposed directly. Do not expose portworx service directly on public network for real scenarios without further review. There might be additional security considerations. 
 
 > Manage the service exposure and network connectivity to allow 2 portworx clusters to communicate should be customized based on security requirements. For eg: private exposure might be sufficient
 
Refer to portworx docs [Enable load balancing on cloud clusters](https://docs.portworx.com/portworx-install-with-kubernetes/disaster-recovery/async-dr/#enable-load-balancing-on-cloud-clusters)

```
▶ oc get svc -n kube-system | grep portworx
```
> Reference https://www.ibm.com/support/producthub/icpdata/docs/content/SSQNUZ_current/cpd/admin/disaster-recovery-config-target.html

- option 1 - Portworx service can be exposed by changing the service `portworx-service`  to use a `LoadBalancer` type. In this case, a vpc loadbalancer hostname will be generated and it maps to 9001 service port
   > Do not do this in real environments withoit further consideration
    ```
    ▶ kubectl edit svc portworx-service -n kube-system
    ```

- option 2 - Alternatively, a route can be generated which by default is exposed on port 80 and maps to 9001 port of `portworx-service`
   > Do not do this in real environments withoit further consideration
    ```
    ▶ oc expose -n kube-system svc/portworx-service 
    ```
The lb/route hostname generated and port will be used in the `clusterpair.yaml` in next steps

## Generate and edit ClusterPair spec
> Refer portworx docs [Generate a ClusterPair on destionation cluster](https://docs.portworx.com/portworx-install-with-kubernetes/disaster-recovery/async-dr/#generate-a-clusterpair-on-the-destination-cluster)
- Create namespace for migration setup
    ```
    ▶ oc new-project test-async-dr
    ```
- Generate clusterpair 

    ```
    ▶ oc project default
    ▶ storkctl generate clusterpair -n test-async-dr remote-us-east-cluster > clusterpair.yaml
    ```
    The name `remote-us-east-cluster` will be the identifier  for the clusterpair object in source cluster referenced during migration 

- Edit the `clusterpair.yaml` file generated from above

  > **Note** For this PoC setup, that steps here are documented with cluster config with temporary OpenShift access token which expires after specific amount of time. With alternate options, this can be improved 
  Eg - use of service account token where token does not expire.
  
  1. Get token for destination portworx cluster
     ```
     ▶ PX_POD=$(kubectl get pods -l name=portworx -n kube-system -o jsonpath='{.items[0].metadata.name}')
     ▶ kubectl exec $PX_POD -n kube-system -- /opt/pwx/bin/pxctl cluster token show
     Token is xxxxxxxxxxxxxxxxxxxxxxx....
     ```
  2. Get ROKS cluster user token
     ```
     ▶ oc whoami --show-token                                                                                
     jsPKsxxxxxxxxxxxxxxxxxxxxx
     ```
  3. Edit the `clusterpair.yaml` to include 
     ```
     options:
       ip:     <ip_of_remote_px_node>
       port:   <port_of_remote_px_node_default_9001>
       token:  <token_from_step_1>
       mode: DisasterRecovery
     ```
  > Refer portworx docs - [Enable disaster recovery mode](https://docs.portworx.com/portworx-install-with-kubernetes/disaster-recovery/async-dr/#enable-disaster-recovery-mode)
  
  Sample `clusterpair.yaml` is below:
  
  > Note that the structure of the file might vary depending on the method used for logging in to the cluster to generate this file
    ```
    apiVersion: stork.libopenstorage.org/v1alpha1
    kind: ClusterPair
    metadata:
        name: remote-us-east-cluster
        namespace: migrationnamespace
    spec:
        config:
        clusters:
            c100-e-us-east-containers-cloud-ibm-com:30736:
            LocationOfOrigin: <kube-config>
            server: https://c100-e.us-east.containers.cloud.ibm.com:30736
        contexts:
            default/c100-e-us-east-containers-cloud-ibm-com:30736/IAM#<id>:
            LocationOfOrigin: <kube-config>
            cluster: c100-e-us-east-containers-cloud-ibm-com:30736
            namespace: default
            user: <id>/c100-e-us-east-containers-cloud-ibm-com:30736
        current-context: default/c100-e-us-east-containers-cloud-ibm-com:30736/<id>
        preferences: {}
        users:
            IAM#<id>/c100-e-us-east-containers-cloud-ibm-com:30736:
            LocationOfOrigin: <kube-config>
            token: <token_from_step2_above>
        options:
        ip: "<route/loadbalancer hostname>"
        port: "<port>" #typically 9001 if using LB or port 80 route is used
        token: <token_from_step1_above>
    ```
### Create custom storage class   
If you want to use one of the existing storage classes, you can skip this. Edit the `test-statefulset.yaml` accordingly

(optional) Create custom storage class. Later in this doc, we will migrate a test statefulset that will make use of this storage class. So, create the custom storage class before migration on destination cluster
  ```
  ▶ oc create -f custom-sc.yaml
  ```
   

## On Source/Primary cluster

### Create cluster pairing
- Create cloud credentials. Follow same steps from section [Create cloud credentials](#create-cloud-credentials)
 > Note that, UUID used in credentials name must be still be the destination cluster's UUID `clusterPair_<clusterUUID_of_destination>` (Both source and destination should have creds with same name)

- Create namespace for migration setup
    ```
    ▶ oc new-project test-async-dr
    ```
- Create cluster pairing
    ```
    ▶ oc create -f clusterpair-new.yaml 
    NAME                     AGE
    remote-us-east-cluster   25s
    ```
- Verify pair status
    ```
    ▶ storkctl get clusterpair -n test-async-dr
    NAME                     STORAGE-STATUS   SCHEDULER-STATUS   CREATED
    remote-us-east-cluster   Ready            Ready              24 Sep 20 01:26 CDT
    ```
    ```
    ▶ kubectl describe clusterpair remote-us-east-cluster

    <....>
    Status:
    Remote Storage Id:  82812475-a501-4061-89e5-b85490892063
    Scheduler Status:   Ready
    Storage Status:     Ready
    Events:
    Type    Reason  Age   From   Message
    ----    ------  ----  ----   -------
    Normal  Ready   89s   stork  Storage successfully paired
    Normal  Ready   89s   stork  Scheduler successfully paired
    ```
### Create a test statefulset 
On source cluster (`us-south`), create project and a test stateful set using `test-statefulset.yaml`. This project will be migrated to destination cluster along with its resources
- Create a project
  ```
  ▶ oc new-project pb-pwx-test
  ```
- Create storageclass

  Note that storage class has `secure` parameter set to `true`. The volumes created by this sc will be encrypted using the KP key that portworx cluster is configured to use
  ```
  ▶ oc create -f custom-sc.yaml
  ```

- Create test statefulset
  ```
  ▶ oc create -f test-statefulset.yaml
  ```
  The container in the stateful above, writes current time and date to /usr/share/test/date.log and the directory is mounted as a persistent volume
- Check pods and volumes
    ```
    ▶ oc get pvc
    NAME                      STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                       AGE
    datelog-test-stateful-0   Bound     pvc-7e0aa776-7069-46c0-b1f1-96d2c58da293   1Gi        RWO            pwx-custom-high-random-secure-sc   35h
    datelog-test-stateful-1   Bound     pvc-3ba1966a-a2e1-4994-bb87-e881e848cf8a   1Gi        RWO            pwx-custom-high-random-secure-sc   35h
    ```
    ```
    ▶ oc get pods
    NAME              READY     STATUS              RESTARTS   AGE
    test-stateful-0   1/1       Running             0          36h
    test-stateful-1   1/1       Running    0          36h
    ```
### Schedule Migrations
> Refer [portworx docs](https://docs.portworx.com/portworx-install-with-kubernetes/disaster-recovery/async-dr/#scheduling-migrations)

- Edit stork deployment to specify admin namespace if required. 
Follow steps from [Set up a Cluster Admin namespace for Migration](https://docs.portworx.com/portworx-install-with-kubernetes/migration/cluster-admin-namespace/)
    ```
    ▶ oc edit deployment -n kube-system stork
    ```
    Add the variable admin-namespace to point to the namespace you want to configure for migration related resources
    ```
    - command:
    - /stork
    - --driver=pxd
    - --verbose
    - --leader-elect=true
    - --admin-namespace=test-async-dr
    ```
    This will restart the stork pods

- Create schedulepolicy
    ```
    ▶ oc create -f schedulepolicy.yaml               
    schedulepolicy.stork.libopenstorage.org/failoverschedulepolicy created
    ```
    We specified `5m` interval in the yaml for testing purposes.

- Create migration policy
  Refer portworx docs - [Start a migration](https://docs.portworx.com/portworx-install-with-kubernetes/migration/#start-a-migration) for details
    ```
    ▶ oc create -f migrationschedule.yaml 
    migrationschedule.stork.libopenstorage.org/failovermigrationschedule created
    ```

    ```
    ▶ oc get migrationschedule
    NAME                        AGE
    failovermigrationschedule   96s
    ```
- Observe migrations and status of each migration
    ```
    ▶ oc get migration
    NAME                                                   AGE
    failovermigrationschedule-interval-2020-09-24-063248   2m17s
    ```
    Status will eventually change to `Inprogress`. This may take some time
    ```
    ▶ oc describe migration failovermigrationschedule-interval-2020-09-24-063248

    <...>
    Status:
    Finish Timestamp:  <nil>
    Resources:         <nil>
    Stage:             Volumes
    Status:            InProgress
    Volumes:
        Namespace:                pb-pwx-test
        Persistent Volume Claim:  datelog-test-stateful-0
        Reason:                   Volume migration has started. Backup in progress. BytesDone: 0 BytesTotal: 34209792 ETA: 10 seconds
        Status:                   InProgress
        Volume:                   pvc-7e0aa776-7069-46c0-b1f1-96d2c58da293
        Namespace:                pb-pwx-test
        Persistent Volume Claim:  datelog-test-stateful-1
        Reason:                   Volume migration has started. Backup in progress.
        Status:                   InProgress
        Volume:                   pvc-3ba1966a-a2e1-4994-bb87-e881e848cf8a
    Events:                       <none>
    ```

- Suspend migrations
Since `5m` interval was specified,since this is for testing purposes, you can temporarily suspend the migrationschedule so that no further migrations are invoked
    ```
    storkctl suspend  migrationschedules failovermigrationschedule -n test-asyc-dr
    ```

### Monitoring a migration
```
storkctl get migration --namespace test-async-dr                                               

NAME                                                   CLUSTERPAIR              STAGE     STATUS       VOLUMES   RESOURCES   CREATED               ELA
PSED
failovermigrationschedule-interval-2020-09-24-063248   remote-us-east-cluster   Volumes   InProgress   0/2       0/0         24 Sep 20 01:32 CDT   7m2
.085518s
```
If the migration is successful, the Stage will go from Volumes→ Application→Final

```
▶ storkctl get migration --namespace test-async-dr 
NAME                                                   CLUSTERPAIR              STAGE   STATUS       VOLUMES   RESOURCES   CREATED               ELAPSED
failovermigrationschedule-interval-2020-09-24-063248   remote-us-east-cluster   Final   Successful   2/2       12/12       24 Sep 20 01:32 CDT   15m57s

```
### Migrated resources


- Migrated project
   ```
   apiVersion: stork.libopenstorage.org/v1alpha1
   kind: MigrationSchedule
   metadata:
     name: failovermigrationschedule
     namespace: test-async-dr
   spec:
     template:
       spec:
         clusterPair: remote-us-east-cluster
         includeResources: true
         includeVolumes: true
         startApplications: true
         namespaces:
         - pb-pwx-test
     schedulePolicyName: failoverschedulepolicy
   ```
   Using the above migrationschedule object, it is specified that the goal is to migrate namespace `pb-pwx-test` to the destination cluster identified  as `remote-us-east-cluster` through the clusterpair object created earlier.

- Once the migration is successful, describe the migration object created to see a list of k8s resources that are migrated
    ```
    ▶ oc describe migration failovermigrationschedule-interval-2020-09-24-063248
    Name:         failovermigrationschedule-interval-2020-09-24-063248
    Namespace:    test-async-dr
    Labels:       <none>
    Annotations:  <none>
    API Version:  stork.libopenstorage.org/v1alpha1
    Kind:         Migration
    Metadata:
    Creation Timestamp:  2020-09-24T06:32:48Z
    Finalizers:
        stork.libopenstorage.org/finalizer-cleanup
    Generation:  8
    Owner References:
        API Version:     stork.libopenstorage.org/v1alpha1
        Kind:            MigrationSchedule
        Name:            failovermigrationschedule
        UID:             054c4968-6cf5-4ded-9198-7bf49950d9f1
    Resource Version:  40767960
    Self Link:         /apis/stork.libopenstorage.org/v1alpha1/namespaces/test-async-dr/migrations/failovermigrationschedule-interval-2020-09-24-063248
    UID:               5a9aca8f-b95d-48e7-bb18-48d0806da477
    Spec:
    Admin Cluster Pair:               
    Cluster Pair:                     remote-us-east-cluster
    Include Optional Resource Types:  <nil>
    Include Resources:                true
    Include Volumes:                  true
    Namespaces:
        pb-pwx-test
    Post Exec Rule:           
    Pre Exec Rule:            
    Purge Deleted Resources:  false
    Selectors:                <nil>
    Start Applications:       true
    Status:
    Finish Timestamp:  2020-09-24T06:48:45Z
    Resources:
        Group:      core
        Kind:       ConfigMap
        Name:       example
        Namespace:  pb-pwx-test
        Reason:     Resource migrated successfully
        Status:     Successful
        Version:    v1
        Group:      core
        Kind:       ConfigMap
        Name:       storage-test-pwx
        Namespace:  pb-pwx-test
        Reason:     Resource migrated successfully
        Status:     Successful
        Version:    v1
        Group:      core
        Kind:       PersistentVolumeClaim
        Name:       datelog-test-stateful-0
        Namespace:  pb-pwx-test
        Reason:     Resource migrated successfully
        Status:     Successful
        Version:    v1
        Group:      core
        Kind:       PersistentVolumeClaim
        Name:       datelog-test-stateful-1
        Namespace:  pb-pwx-test
        Reason:     Resource migrated successfully
        Status:     Successful
        Version:    v1
        Group:      core
        Kind:       Service
        Name:       hello-world
        Namespace:  pb-pwx-test
        Reason:     Resource migrated successfully
        Status:     Successful
        Version:    v1
        Group:      core
        Kind:       Service
        Name:       test-stateful
        Namespace:  pb-pwx-test
        Reason:     Resource migrated successfully
        Status:     Successful
        Version:    v1
        Group:      core
        Kind:       ServiceAccount
        Name:       default
        Namespace:  pb-pwx-test
        Reason:     Resource migrated successfully
        Status:     Successful
        Version:    v1
        Group:      core
        Kind:       PersistentVolume
        Name:       pvc-3ba1966a-a2e1-4994-bb87-e881e848cf8a
        Namespace:  
        Reason:     Resource migrated successfully
        Status:     Successful
        Version:    v1
        Group:      core
        Kind:       PersistentVolume
        Name:       pvc-7e0aa776-7069-46c0-b1f1-96d2c58da293
        Namespace:  
        Reason:     Resource migrated successfully
        Status:     Successful
        Version:    v1
        Group:      core
        Kind:       Secret
        Name:       all-icr-io
        Namespace:  pb-pwx-test
        Reason:     Resource migrated successfully
        Status:     Successful
        Version:    v1
        Group:      apps
        Kind:       StatefulSet
        Name:       test-stateful
        Namespace:  pb-pwx-test
        Reason:     Resource migrated successfully
        Status:     Successful
        Version:    v1
        Group:      rbac.authorization.k8s.io
        Kind:       RoleBinding
        Name:       admin
        Namespace:  pb-pwx-test
        Reason:     Resource migrated successfully
        Status:     Successful
        Version:    v1
    Stage:        Final
    Status:       Successful
    Volumes:
        Namespace:                pb-pwx-test
        Persistent Volume Claim:  datelog-test-stateful-0
        Reason:                   Migration successful for volume
        Status:                   Successful
        Volume:                   pvc-7e0aa776-7069-46c0-b1f1-96d2c58da293
        Namespace:                pb-pwx-test
        Persistent Volume Claim:  datelog-test-stateful-1
        Reason:                   Migration successful for volume
        Status:                   Successful
        Volume:                   pvc-3ba1966a-a2e1-4994-bb87-e881e848cf8a
    Events:                       <none>
    ```
 
## Check resources created on destination cluster

All the resources listed in the migration object above are now migrated to the destination cluster.

Once the migration is successful, new project will automatically be created and, portworx volumes, statefulset and, other k8s resources are migrated.
```
▶ oc projects | grep pb-pwx-test
  * pb-pwx-test
```
```
▶ oc get pvc
NAME                      STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                       AGE
datelog-test-stateful-0   Bound     pvc-7e0aa776-7069-46c0-b1f1-96d2c58da293   1Gi        RWO            pwx-custom-high-random-secure-sc   21h
datelog-test-stateful-1   Bound     pvc-3ba1966a-a2e1-4994-bb87-e881e848cf8a   1Gi        RWO            pwx-custom-high-random-secure-sc   21h
```
```
▶ oc get pods
NAME              READY     STATUS    RESTARTS   AGE
test-stateful-0   1/1       Running   0          21h
test-stateful-1   1/1       Running   0          21h
```

The container of the test statefulset, write the timestamp to the log file every 10 secs. Notice that logs written to the volume on the source cluster are preserved without major data loss.

```
▶ oc exec -it test-stateful-0 -- cat /usr/share/test/date.log
Wed Sep 23 15:28:17 UTC 2020
Wed Sep 23 15:28:27 UTC 2020
Wed Sep 23 15:28:37 UTC 2020
Wed Sep 23 15:28:47 UTC 2020
Wed Sep 23 15:28:57 UTC 2020
Wed Sep 23 15:29:07 UTC 2020
Wed Sep 23 15:29:17 UTC 2020
Wed Sep 23 15:29:27 UTC 2020
Wed Sep 23 15:29:37 UTC 2020
Wed Sep 23 15:29:47 UTC 2020
Wed Sep 23 15:29:57 UTC 2020
.
.
.
.
```
## Caution

- Try to avoid migration of system namespaces like `default`, `kube-system` etc.. that have cluster components. When migrated, the resources like service accounts might be overwritten corrupting the target cluster.

- If exposing any portworx service, please check if this is allowed in your environment. This step should not be done for prod or stage environment without further security considerations

- If a temporary token is used to create the clusterpair object, this will eventually expire in about 24hrs or so. 

- Portworx maintenance - When a worker is upgraded, it is replaced with new worker on ROKS. Portworx on this node will be down. This requires reattaching block storage to worker and restarting portworx service to be backup. So, use atleast 2 or more replicas for your volumes to avoid downtime of apps


## Recommended Practices

- Use a Service Account instead of your personal account token for cluster pairing 
 Refer - https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/ 

- Portworx maintenance
  
  To avoid downtime of portworx and apps, use replication so that 2 or 3 volumes are created across portworx nodes. And ensure nodes are updated one by one - When a node is unavailable during upgrade, volume replicas created can be used by pods. 


## Troubleshooting

- Clusterpairing errors

  If clusterpairing is not successful for some reason, describe the object to find the error

  ```
  ▶ oc describe clusterpair remote-us-east-cluster -n pwx-async-migration
  Events:
    Type     Reason  Age               From   Message
    ----     ------  ----              ----   -------
    Warning  Error   4m (x64 over 3h)  stork  error validating credential during create:
   RequestError: send request failed
  caused by: Put http://10.241.0.4:17007/8e24f746-7db3-4a4a-bc2e-968b77520d5f: dial tcp 10.241.0.4:17007: connect: connection timed out
  ```
 
  If you see errors during cluster pairing, most likely, network connectivity is blocked b/w clusters. Open the required ports to establish connection. See the [prerequisites](#prerequisites) section if using VPC infra.
  

- Unable to mount volumes
  After migration onto the target cluster, if pods are not running, describe the pods to find more details. 
   ```
   ▶ oc describe pod test-stateful-0
   Events:
     Type     Reason       Age                 From                  Message
     ----     ------       ----                ----                  -------
     Warning  FailedMount  13m (x78 over 20h)  kubelet, 10.241.64.4  Unable to attach or mount volumes: unmounted volumes=[datelog], unattached volumes=[default-token-h2mxm datelog]: timed out waiting for the condition
     Warning  FailedMount  9m (x745 over 20h)  kubelet, 10.241.64.4  (combined from similar events): MountVolume.SetUp failed for volume "pvc-af95ef54-fd95-40cd-b487-f6a0d4a808be" : rpc error: code = Internal desc = failed  to attach volume: Unable to get secret for key [533582546501677402]. kp.Error: correlation_id='c1aa5cb1-43d1-4b66-8144-6ea5249fd044', msg='Bad Request: Action could not be performed on key: Please see `reasons` for more details (UNDEFINED_REASON)', reasons='[UNDEFINED_REASON: decrypt failed with error: encrypted data is invalid]'
     Warning  FailedMount  4m (x316 over 20h)  kubelet, 10.241.64.4  Unable to attach or mount volumes: unmounted volumes=[datelog], unattached volumes=[datelog default-token-h2mxm]: timed out waiting for the condition
   ```

    If you see something like the above, it is possible that portworx is not able to decrypt the volumes that are migrated because portworx on source px cluster is using a different key and target cluster is using a different one. For portworx to decrypt volumes and schedule pods, both should be setup with same key. To achieve this, you can point both clusters to use same KP instance and key or if you are planning for a regiona failover, you can have two KP instance in two different region and import the same key (BYOK) to two KP instances to configure with portworx

  - Unauthorized errors

    If a migration is not successful, describe the migration object to find the reason
    
    ```
    ▶ oc describe migration failovermigrationschedule-interval-2020-09-24-0632
    <...>
    Events:
      Type     Reason      Age               From   Message
      ----     ------      ----              ----   -------
      Normal   Successful  16m               stork  Volume pvc-6b055324-391f-45bf-a799-4559edbd916e migrated successfully
      Normal   Successful  16m               stork  Volume pvc-66992a0b-83c0-4070-89a7-a4d36c67faa6 migrated successfully
      Warning  Failed      16m               stork  Error migrating volumes: Unauthorized
      Warning  Failed      5m (x3 over 16m)  stork  Error applying resource: Unauthorized
      Warning  Failed      5m (x2 over 10m)  stork  Error migrating resources: Unauthorized
    ```
    If you notice unauthorized errors, you might have used a temporary token from ROKS for cluserpairing is now expired. For a POC setup, you can re-create the clusterpair object on source cluster with a new token from destination cluster
  
    Long term solution would be use a service account token that does not expire.


## References

https://docs.portworx.com/portworx-install-with-kubernetes/disaster-recovery/async-dr/#enable-load-balancing-on-cloud-clusters

https://www.ibm.com/support/producthub/icpdata/docs/content/SSQNUZ_current/cpd/admin/disaster-recovery-config-target.html

https://cloud.ibm.com/docs/openshift?topic=openshift-portworx

https://docs.portworx.com/portworx-install-with-kubernetes/migration/






