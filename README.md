# FileNet on Containers manual installation ✍️<!-- omit in toc -->

For version 5.5.12

Installs FNCM environment.

- [Disclaimer ✋](#disclaimer-)
- [Preparing your cluster](#preparing-your-cluster)
- [Preparing a client to connect to the cluster](#preparing-a-client-to-connect-to-the-cluster)
  - [Creating an install client directly in OCP](#creating-an-install-client-directly-in-ocp)
  - [Command line preparation in install Pod](#command-line-preparation-in-install-pod)
- [Prerequisite software](#prerequisite-software)
  - [OpenLDAP](#openldap)
    - [Access info after deployment](#access-info-after-deployment)
  - [PostgreSQL](#postgresql)
    - [Access info after deployment](#access-info-after-deployment-1)
- [FileNet environment](#filenet-environment)
  - [Preparing a client to connect to the cluster](#preparing-a-client-to-connect-to-the-cluster-1)
  - [Installing the operator by running a script](#installing-the-operator-by-running-a-script)
  - [Use prerequisite script](#use-prerequisite-script)
    - [Install prereqs and change directory](#install-prereqs-and-change-directory)
    - [Gether phase - generate property files](#gether-phase---generate-property-files)
    - [Edit property files](#edit-property-files)
    - [Generate phase - generate sqls, secrets, fncm yaml](#generate-phase---generate-sqls-secrets-fncm-yaml)
    - [Apply SQLs to DB instance](#apply-sqls-to-db-instance)
    - [Modify CR file and Network POlicies](#modify-cr-file-and-network-policies)
    - [Validate phase - validate connectivity and apply artifacts](#validate-phase---validate-connectivity-and-apply-artifacts)
    - [Validating your deployment](#validating-your-deployment)
  - [License Metering](#license-metering)
- [Contacts](#contacts)
- [Notice](#notice)

## Disclaimer ✋

This is **not** an official IBM documentation.  
Absolutely no warranties, no support, no responsibility for anything.  
Use it on your own risk and always follow the official IBM documentations.  
It is always your responsibility to make sure you are license compliant when using this repository to install FileNet on Containers.  
Used OpenLDAP and PostgreSQL cannot run on Restricted OCP from CP4BA.

Please do not hesitate to create an issue here if needed. Your feedback is appreciated.

Not for production use. Suitable for Demo and PoC environments.

OpenLDAP is not a supported LDAP provider for FNCM.

## Preparing your cluster

Based on https://www.ibm.com/docs/en/filenet-p8-platform/5.5.12?topic=installing-preparing-your-cluster and  
https://www.ibm.com/docs/en/filenet-p8-platform/5.5.12?topic=cluster-getting-access-images-from-public-entitled-registry

- Empty OpenShift cluster of a supported version
- With direct internet connection
- File RWX StorageClass - in this case ocs-storagecluster-cephfs is used, feel free to find and replace
- Cluster admin user
- IBM entitlement key from https://myibm.ibm.com/products-services/containerlibrary

## Preparing a client to connect to the cluster

Based on https://www.ibm.com/docs/en/filenet-p8-platform/5.5.12?topic=cluster-preparing-client-connect
and additional utilities for script execution

Requested tooling provided in the install Pod.

- kubectl
- oc
- podman
- IBM Semeru JDK 17
- python 3.9 - should be 3.8 but it is not available in the UBI image repo
- yq
- jq

### Creating an install client directly in OCP

In you OCP cluster create Project *fncm-install* using OpenShift console.
```yaml
kind: Project
apiVersion: project.openshift.io/v1
metadata:
  name: fncm-install
  labels:
    app: fncm-install
```

Create PVC where all files used for installation would be kept if you would need to re-instantiate the install Pod.
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: install
  namespace: fncm-install
  labels:
    app: fncm-install
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: ocs-storagecluster-cephfs
  volumeMode: Filesystem
```

Assign cluster admin permissions to *fncm-install* default ServiceAccount under which the installation is performed by applying the following yaml.

This requires the logged in OpenShift user to be cluster admin.

The ServiceAccount needs to have cluster admin to be able to create all resources needed to deploy the platform.
```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: cluster-admin-fncm-install
  labels:
    app: fncm-install
subjects:
  - kind: User
    apiGroup: rbac.authorization.k8s.io
    name: "system:serviceaccount:fncm-install:default"
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
```

Create a pod from which the install will be performed with the following definition.  
It is ready when the message *Install pod - Ready* is in its Log.
```yaml
kind: Pod
apiVersion: v1
metadata:
  name: install
  namespace: fncm-install
  labels:
    app: fncm-install
spec:
  containers:
    - name: install
      securityContext:
        privileged: true
      image: ubi9/ubi:9.3
      command: ["/bin/bash"]
      args:
        ["-c","cd /usr;
          yum install podman -y;
          yum install ncurses -y;
          yum install jq -y;
          yum install python3.9 -y;
          yum install python3.9-pip -y;
          curl -LO https://github.com/ibmruntimes/semeru17-binaries/releases/download/jdk-17.0.9%2B9_openj9-0.41.0/ibm-semeru-open-jdk_x64_linux_17.0.9_9_openj9-0.41.0.tar.gz;
          tar -xvf ibm-semeru-open-jdk_x64_linux_17.0.9_9_openj9-0.41.0.tar.gz;
          ln -fs /usr/jdk-17.0.9+9/bin/java /usr/bin/java;
          ln -fs /usr/jdk-17.0.9+9/bin/keytool /usr/bin/keytool;
          curl -k https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable/openshift-client-linux.tar.gz --output oc.tar;
          tar -xvf oc.tar oc;
          chmod u+x oc;
          ln -fs /usr/oc /usr/bin/oc;
          curl -LO https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl;
          chmod u+x kubectl;
          ln -fs /usr/kubectl /usr/bin/kubectl;
          curl -LO https://github.com/mikefarah/yq/releases/download/v4.30.5/yq_linux_amd64.tar.gz;
          tar -xzvf yq_linux_amd64.tar.gz;
          ln -fs /usr/yq_linux_amd64 /usr/bin/yq;
          podman -v;
          java -version;
          keytool;
          oc version;
          kubectl version;
          yq --version;
          jq --version;
          python3 --version;
          pip3 --version;
          while true;
          do echo 'Install pod - Ready - Enter it via Terminal and \"bash\"';
          sleep 300; done"]
      imagePullPolicy: IfNotPresent
      volumeMounts:
        - name: install
          mountPath: /usr/install
  volumes:
    - name: install
      persistentVolumeClaim:
        claimName: install
```

### Command line preparation in install Pod

This needs to be repeated if you don't perform this in one go and you come back to resume.

Open Terminal window of the *install* pod.

Enter bash.
```sh
bash
```

## Prerequisite software

FNCM needs at least LDAP and Database. In this deployment OpenLDAP and PostgreSQL are used.

You would normally use your own production ready instances.

Please note that OpenLDAP is working but is not officially supported by CP4BA.

### OpenLDAP

Create fncm-openldap Project
```bash
echo "
kind: Project
apiVersion: project.openshift.io/v1
metadata:
  name: fncm-openldap
  labels:
    app: fncm-openldap
" | oc apply -f -
```

Add required privileged permission
```bash
echo "
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: openldap-privileged
  namespace: fncm-openldap
  labels:
    app: fncm-openldap
subjects:
  - kind: ServiceAccount
    name: default
    namespace: fncm-openldap
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:openshift:scc:privileged
" | oc apply -f -
```

Create ConfigMap for env
```bash
echo "
kind: ConfigMap
apiVersion: v1
metadata:
  name: openldap-env
  namespace: fncm-openldap
  labels:
    app: fncm-openldap
data:
  BITNAMI_DEBUG: 'true'
  LDAP_ORGANISATION: cp.internal
  LDAP_ROOT: 'dc=cp,dc=internal'
  LDAP_DOMAIN: cp.internal
" | oc apply -f -
```

Create ConfigMap for users and groups
```bash
echo "
kind: ConfigMap
apiVersion: v1
metadata:
  name: openldap-customldif
  namespace: fncm-openldap
  labels:
    app: fncm-openldap
data:
  01-sds-schema.ldif: |-
    dn: cn=sds,cn=schema,cn=config
    objectClass: olcSchemaConfig
    cn: sds
    olcAttributeTypes: {0}( 1.3.6.1.4.1.42.2.27.4.1.6 NAME 'ibm-entryUuid' DESC 
      'Uniquely identifies a directory entry throughout its life.' EQUALITY caseIgnoreMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.15 SINGLE-VALUE )
    olcObjectClasses: {0}( 1.3.6.1.4.1.42.2.27.4.2.1 NAME 'sds' DESC 'sds' SUP top AUXILIARY MUST ( cn $ ibm-entryuuid ) )
  02-default-users.ldif: |-
    # cp.internal
    dn: dc=cp,dc=internal
    objectClass: top
    objectClass: dcObject
    objectClass: organization
    o: cp.internal
    dc: cp

    # Units
    dn: ou=Users,dc=cp,dc=internal
    objectClass: organizationalUnit
    ou: Users

    dn: ou=Groups,dc=cp,dc=internal
    objectClass: organizationalUnit
    ou: Groups

    # Users
    dn: uid=cpadmin,ou=Users,dc=cp,dc=internal
    objectClass: inetOrgPerson
    objectClass: sds
    objectClass: top
    cn: cpadmin
    sn: cpadmin
    uid: cpadmin
    mail: cpadmin@cp.internal
    userpassword: Password
    employeeType: admin
    ibm-entryuuid: e6c41859-ced3-4772-bfa3-6ebbc58ec78a

    dn: uid=cpuser,ou=Users,dc=cp,dc=internal
    objectClass: inetOrgPerson
    objectClass: sds
    objectClass: top
    cn: cpuser
    sn: cpuser
    uid: cpuser
    mail: cpuser@cp.internal
    userpassword: Password
    ibm-entryuuid: 40324128-84c8-48c3-803d-4bef500f84f1

    # Groups
    dn: cn=cpadmins,ou=Groups,dc=cp,dc=internal
    objectClass: groupOfNames
    objectClass: sds
    objectClass: top
    cn: cpadmins
    ibm-entryuuid: 53f96449-2b7e-4402-a58a-9790c5089dd0
    member: uid=cpadmin,ou=Users,dc=cp,dc=internal

    dn: cn=cpusers,ou=Groups,dc=cp,dc=internal
    objectClass: groupOfNames
    objectClass: sds
    objectClass: top
    cn: cpusers
    ibm-entryuuid: 30183bb0-1012-4d23-8ae2-f94816b91a75
    member: uid=cpadmin,ou=Users,dc=cp,dc=internal
    member: uid=cpuser,ou=Users,dc=cp,dc=internal
" | oc apply -f -
```

Create Secret for password
```bash
echo "
kind: Secret
apiVersion: v1
metadata:
  name: openldap
  namespace: fncm-openldap
  labels:
    app: fncm-openldap
stringData:
  LDAP_ADMIN_PASSWORD: Password
" | oc apply -f -
```

Create PVC for data
```bash
echo "
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: openldap-data
  namespace: fncm-openldap
  labels:
    app: fncm-openldap
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: ocs-storagecluster-cephfs
  volumeMode: Filesystem
" | oc apply -f -
```

Create Deployment
```bash
echo "
kind: Deployment
apiVersion: apps/v1
metadata:
  name: openldap
  namespace: fncm-openldap
  labels:
    app: fncm-openldap
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fncm-openldap
  template:
    metadata:
      labels:
        app: fncm-openldap
    spec:
      containers:
        - name: openldap
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          startupProbe:
            tcpSocket:
              port: ldap-port
            timeoutSeconds: 1
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 30
          readinessProbe:
            tcpSocket:
              port: ldap-port
            initialDelaySeconds: 60
            timeoutSeconds: 1
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 10
          livenessProbe:
            tcpSocket:
              port: ldap-port
            initialDelaySeconds: 60
            timeoutSeconds: 1
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 10
          terminationMessagePath: /dev/termination-log
          ports:
            - name: ldap-port
              containerPort: 1389
              protocol: TCP
          image: 'bitnami/openldap:2.6.5'
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: data
              mountPath: /bitnami/openldap/
            - name: custom-ldif-files
              mountPath: /ldifs/02-default-users.ldif
              subPath: 02-default-users.ldif
            - name: custom-ldif-files
              mountPath: /schemas/01-sds-schema.ldif
              subPath: 01-sds-schema.ldif
          terminationMessagePolicy: File
          envFrom:
            - configMapRef:
                name: openldap-env
            - secretRef:
                name: openldap
          securityContext:
            privileged: true
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: openldap-data
        - name: custom-ldif-files
          configMap:
            name: openldap-customldif
            defaultMode: 420
" | oc apply -f -
```

Create Service
```bash
echo "
kind: Service
apiVersion: v1
metadata:
  name: openldap
  namespace: fncm-openldap
  labels:
    app: fncm-openldap
spec:
  ports:
    - name: ldap-port
      protocol: TCP
      port: 389
      targetPort: ldap-port
  type: NodePort
  selector:
    app: fncm-openldap
" | oc apply -f -
```

Wait for pod in *fncm-openldap* Project to become Ready 1/1.
```bash
oc get pod -n fncm-openldap -w
```

#### Access info after deployment

OpenLDAP
- See Project fncm-openldap, Service openldap for assigned NodePort 
- cn=admin,dc=cp,dc=internal / Password
- In OpenLDAP Pod terminal `ldapsearch -x -b "dc=cp,dc=internal" -H ldap://localhost:1389 -D 'cn=admin,dc=cp,dc=internal' -W "objectclass=*"`

### PostgreSQL

Create fncm-postgresql Project
```bash
echo "
kind: Project
apiVersion: project.openshift.io/v1
metadata:
  name: fncm-postgresql
  labels:
    app: fncm-postgresql
" | oc apply -f -
```

Add required privileged permission
```bash
echo "
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: postgresql-privileged
  namespace: fncm-postgresql
  labels:
    app: fncm-postgresql
subjects:
  - kind: ServiceAccount
    name: default
    namespace: fncm-postgresql
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:openshift:scc:privileged
" | oc apply -f -
```

Create Secret for config
```bash
echo "
kind: Secret
apiVersion: v1
metadata:
  name: postgresql-config
  namespace: fncm-postgresql
  labels:
    app: fncm-postgresql
stringData:
  POSTGRES_DB: postgresdb
  POSTGRES_USER: cpadmin
  POSTGRES_PASSWORD: Password
" | oc apply -f -
```

Create PVC for data
```bash
echo "
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: postgresql-data
  namespace: fncm-postgresql
  labels:
    app: fncm-postgresql
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: ocs-storagecluster-cephfs
  volumeMode: Filesystem
" | oc apply -f -
```

Create PVC for table spaces as they should not be in PGDATA
```bash
echo "
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: postgresql-tablespaces
  namespace: fncm-postgresql
  labels:
    app: fncm-postgresql
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: ocs-storagecluster-cephfs
  volumeMode: Filesystem
" | oc apply -f -
```

Create StatefulSet
```bash
echo "
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql
  namespace: fncm-postgresql
spec:
  serviceName: postgresql
  replicas: 1
  selector:
    matchLabels:
      app: fncm-postgresql
  template:
    metadata:
      labels:
        app: fncm-postgresql
    spec:
      containers:
        - name: postgresql
          args:
            - '-c'
            - max_prepared_transactions=500
            - '-c'
            - max_connections=500
          resources:
            limits:
              memory: 4Gi
            requests:
              memory: 4Gi
          livenessProbe:
            exec:
              command:
                - /bin/sh
                - -c
                - exec pg_isready -U \$POSTGRES_USER -d \$POSTGRES_DB
            failureThreshold: 6
            initialDelaySeconds: 30
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 5
          readinessProbe:
            exec:
              command:
                - /bin/sh
                - -c
                - exec pg_isready -U \$POSTGRES_USER -d \$POSTGRES_DB
            failureThreshold: 6
            initialDelaySeconds: 5
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 5
          startupProbe:
            exec:
              command:
                - /bin/sh
                - -c
                - exec pg_isready -U \$POSTGRES_USER -d \$POSTGRES_DB
            failureThreshold: 18
            periodSeconds: 10
          securityContext:
            privileged: true
          image: postgres:14.7-alpine3.17
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 5432
          envFrom:
            - secretRef:
                name: postgresql-config
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: postgresql-data
            - mountPath: /pgsqldata
              name: postgresql-tablespaces
      volumes:
        - name: postgresql-data
          persistentVolumeClaim:
            claimName: postgresql-data
        - name: postgresql-tablespaces
          persistentVolumeClaim:
            claimName: postgresql-tablespaces
" | oc apply -f -
```

Create Service
```bash
echo "
kind: Service
apiVersion: v1
metadata:
  name: postgresql
  namespace: fncm-postgresql    
  labels:
    app: fncm-postgresql
spec:
  type: NodePort
  ports:
    - port: 5432
  selector:
    app: fncm-postgresql
" | oc apply -f -
```

Wait for pod in *fncm-postgresql* Project to become Ready 1/1.
```bash
oc get pod -n fncm-postgresql -w
```

#### Access info after deployment

PostgreSQL
- See Project fncm-postgresql, Service postgresql for assigned NodePort 
- cpadmin / Password
- In PostgreSQL Pod terminal `psql postgresql://cpadmin@localhost:5432/postgresdb`

## FileNet environment

Based on https://www.ibm.com/docs/en/filenet-p8-platform/5.5.12?topic=using-containers

### Preparing a client to connect to the cluster

Based on https://www.ibm.com/docs/en/filenet-p8-platform/5.5.12?topic=cluster-preparing-client-connect

CASE packages repository https://github.com/IBM/cloud-pak/tree/master/repo/case/ibm-cp-fncm-case

```bash
# Create directory for fncm
mkdir /usr/install/fncm

# Download the package
curl https://raw.githubusercontent.com/IBM/cloud-pak/master/repo/case/\
ibm-cp-fncm-case/1.8.0/ibm-cp-fncm-case-1.8.0.tgz \
--output /usr/install/fncm/ibm-cp-fncm-case-1.8.0.tgz

# Extract the package
tar xzvf /usr/install/fncm/ibm-cp-fncm-case-1.8.0.tgz -C /usr/install/fncm/

# Extract cert-kubernetes
tar xvf /usr/install/fncm/ibm-cp-fncm-case/inventory/\
fncmOperator/files/deploy/crs/container-samples-5.5.12.tar \
-C /usr/install/fncm/ibm-cp-fncm-case/inventory/\
fncmOperator/files/deploy/crs/
```

### Installing the operator by running a script

Based on https://www.ibm.com/docs/en/filenet-p8-platform/5.5.12?topic=deployment-installing-operator-by-running-script

Initiate cluster admin setup
```bash
/usr/install/fncm/ibm-cp-fncm-case/inventory/\
fncmOperator/files/deploy/crs/container-samples/scripts/deployOperator.sh
```

```text
You need to read the International Program License Agreement before start

IMPORTANT: Review the license information for the product bundle you are deploying. 

IBM FileNet Content Manager license information here: https://ibm.biz/CPE_FNCM_License_5_5_12 

IBM Content Foundation license information here: https://ibm.biz/CPE_ICF_License_5_5_12 

IBM Content Platform Engine Software Notices here: https://ibm.biz/CPE_FNCM_ICF_Notices_5_5_12 

Press any key to continue

Do you accept the International Program License (Yes/No, default: No): Yes

Starting to Install the IBM FileNet Standalone Operator...

Select the cloud platform to deploy: 
1) RedHat OpenShift Kubernetes Service (ROKS) - Public Cloud
2) Openshift Container Platform (OCP) - Private Cloud
3) Other (Certified Kubernetes Cloud Platform / CNCF)
Enter a valid option [1 to 3]: 2 # Based on your platform

This script prepares the environment for the deployment of some FileNet Content Management capabilities 


Do you want to add a non-administrator user to manage the namespace (Yes/No, default: No): No # Yes if you have non-c,luster admin user available

Enter the name for a new project or an existing project (namespace): fncm
Using project fncm...
Podman is installed.


Follow the instructions on how to get your Entitlement Key: 
https://www.ibm.com/docs/SSNW2F_5.5.12/com.ibm.dba.install/op_topics/tsk_images_enterp_entitled.html 

Do you have an IBM Entitlement Registry key (Yes/No, default: No): Yes

Enter your Entitlement Registry key: # it is OK that when you paste nothing is seen - it is for security.
Verifying the Entitlement Registry key...
Login Succeeded!
Entitlement Registry key is valid.
Creating docker-registry secret for Entitlement Registry key in project fncm...
secret/ibm-entitlement-key created
Done

Creating the custom resource definition (CRD) and a service account that has the permissions to manage the resources... Done!
Creating ibm-fncm-operator role ... Done!
Creating ibm-fncm-operator role binding ...Done!

Label the default namespace to allow network policies to open traffic to the ingress controller using a namespaceSelector...namespace/default labeled
Done!
catalogsource.operators.coreos.com/ibm-fncm-operator-catalog created
IBM FileNet Content Manager Operator Catalog source created!
Waiting for IBM FileNet Content Manager Operator Catalog pod initialization
Waiting for IBM FileNet Content Manager Operator Catalog pod initialization
IBM FileNet Content Manager Operator Catalog is running ibm-fncm-operator-catalog-ml6wc         1/1   Running   0     31s
operatorgroup.operators.coreos.com/ibm-fncm-operator-catalog-group created
IBM FileNet Content Manager Operator Group Created!
subscription.operators.coreos.com/ibm-fncm-operator-catalog-subscription created
IBM FileNet Content Manager Operator Subscription Created!

Waiting for IBM FileNet Content Manager Operator pod initialization
No resources found in fncm namespace.
IBM FileNet Content Manager Operator is running ibm-fncm-operator-7d67546bdd-prgqd   1/1   Running   0     2m16s


Label the default namespace to allow network policies to open traffic to the ingress controller using a namespaceSelector...namespace/default not labeled
Done

Storage classes are needed by the CR file when deploying FNCM Standalone.   You will be asked for three (3) storage classes to meet the slow, medium, and fast storage for the configuration of FNCM components.  If you don't have three (3) storage classes, you can use the same one for slow, medium, or fast.  Note that you can get the existing storage class(es) in the environment by running the following command: oc get storageclass. Take note of the storage classes that you want to use for deployment. 
NAME                                  PROVISIONER                             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
ocs-storagecluster-ceph-rbd           openshift-storage.rbd.csi.ceph.com      Delete          Immediate              true                   4d2h
ocs-storagecluster-ceph-rgw           openshift-storage.ceph.rook.io/bucket   Delete          Immediate              false                  4d2h
ocs-storagecluster-cephfs (default)   openshift-storage.cephfs.csi.ceph.com   Delete          Immediate              true                   4d2h
openshift-storage.noobaa.io           openshift-storage.noobaa.io/obc         Delete          Immediate              false                  4d2h
thin                                  kubernetes.io/vsphere-volume            Delete          Immediate              false                  4d2h
thin-csi                              csi.vsphere.vmware.com                  Delete          WaitForFirstConsumer   true                   4d2h

*******************************************************
                    Summary of input                   
*******************************************************
1. Cloud platform to deploy: OCP 4.X
2. Project to deploy: fncm
3. User selected: 
*******************************************************
```

Wait until the script finishes.

Wait until all Operators in Project fncm are in *Succeeded* state.
```bash
oc get csv -n fncm -w
```

### Use prerequisite script

Based on https://www.ibm.com/docs/en/filenet-p8-platform/5.5.12?topic=configuring-generating-simple-custom-resource-deployment-files

#### Install prereqs and change directory

Install necessary Python packages
```bash
python3 -m pip install -r /usr/install/fncm/ibm-cp-fncm-case/inventory/\
fncmOperator/files/deploy/crs/container-samples/scripts/prerequisites/requirements.txt
```

Change directory is needed as prerequisites.py script generated output in current working directory.
```bash
cd /usr/install/fncm/ibm-cp-fncm-case/inventory/\
fncmOperator/files/deploy/crs/container-samples/scripts/prerequisites
```

#### Gether phase - generate property files

```bash
python3 prerequisites.py gather
```

```text
╭─ FileNet Content Manager Deployment Prerequisites CLI ─╮
│ Version: 2.4.2                                         │
│ Mode: Gather                                           │
╰────────────────────────────────────────────────────────╯

╭─────────╮
│ Version │
╰─────────╯

Which version of FNCM S do you want to deploy?
1. 5.5.8
2. 5.5.11
3. 5.5.12
Enter a valid option [1 and 3]: 3

╭─────────╮
│ License │
╰─────────╯

╭────────────────────────────────────────────────────────────────────────────────────────────────╮
│ IMPORTANT: Review the license  information for the product bundle you are deploying.           │
│                                                                                                │
│ IBM FileNet Content Manager license information here: https://ibm.biz/CPE_FNCM_License_5_5_12  │
│ IBM Content Foundation license information here: https://ibm.biz/CPE_ICF_License_5_5_12        │
│ IBM Content Platform Engine Software Notices here: https://ibm.biz/CPE_FNCM_ICF_Notices_5_5_12 │
╰────────────────────────────────────────────────────────────────────────────────────────────────╯

Do you accept the International Program License? [y/n]: y

Select a License Type
1. ICF
2. FNCM
3. CP4BA
Enter a valid option [1 and 3]: 3 # Based on yor license

Select a License Metric
1. NonProd
2. Prod
3. User
Enter a valid option [1 and 3]: 1

╭──────────╮
│ Platform │
╰──────────╯

Select a Platform Type
1. OCP
2. ROKS
3. CNCF
Enter a valid option [1 and 3]: 1 # Based on your platform

╭─────────────────────╮
│ Authentication Type │
╰─────────────────────╯

Your Authentication Type determines how users login and where they are stored.

Select an Authentication Type
1. LDAP
2. LDAP + IDP
3. SCIM + IDP
Enter a valid option [1 and 3]: 1 #as we have OpenLDAP only

╭────────────────────────────────────────────────╮
│ FIPS (Federal Information Processing Standard) │
╰────────────────────────────────────────────────╯

FIPS is a U.S. government computer security standard.
Make sure your K8s Cluster has FIPS enabled nodes, before you enable FIPS on your deployment.

Do you want to configure a FIPS enabled deployment [y/n]: n

╭────────────────────────────╮
│ Restricted Internet Access │
╰────────────────────────────╯

Restricted Internet Access is a security feature that restricts outbound network access.

Do you want to enable Restricted Internet Access [y/n]: y

╭────────────╮
│ Components │
╰────────────╯

Select zero or more FileNet Content Management Components
Enter [0] to finish selection
1. CPE ✔
2. GraphQL ✔
3. Navigator ✔
4. CSS ✔
5. CMIS ✔
6. Task Manager ✔
Enter a valid option [1 and 6]: 4, 5, 6 and then 0

╭───────────────────╮
│ Component Options │
╰───────────────────╯

Add Java SendMail support for IBM Content Navigator? [y/n]: n

Add IBM Content Collector support for IBM Content Search Services? [y/n]: n

Add custom groups and users for IBM Task Manager? [y/n]: y

╭──────────╮
│ Database │
╰──────────╯

Select a Database Type
1. IBM Db2
2. IBM Db2 HADR
3. Microsoft SQL Server
4. PostgreSQL
5. Oracle
Enter a valid option [1 and 5]: 4

How many Object Stores do you want to deploy? (1): 1

Do you want to enable SSL for your database selection? [y/n]: n

╭──────╮
│ LDAP │
╰──────╯

How many LDAP's do you want to configure? (1): 1

╭───────────────╮
│ LDAP ID: ldap │
╰───────────────╯

Select a LDAP Type
1. Microsoft Active Directory
2. IBM Security Directory Server
3. NetIQ eDirectory
4. Oracle Internet Directory
5. Oracle Directory Server Enterprise Edition
6. Oracle Unified Directory
7. CA eTrust
Enter a valid option [1 and 7]: 2 #Later will be modified to OpenLDAP

Do you want to enable SSL for this LDAP? [y/n]: n

╭───────────────────────────────╮
│ Initialize and Verify Content │
╰───────────────────────────────╯

Do you want to initialize content? [y/n]: y

Do you want to verify content? [y/n]: y

╭────────────╮                                                                                                       
│ Next Steps │                                                                                                       
╰────────────╯                                                                                                       
╭────────────────────────────────────────────────────────────────────────────────────────────────────────────╮
│ 1. Review the toml files in the propertyFiles folder                                                       │
│ 2. Fill the <Required> values                                                                              │
│ 3. If SSL is enabled, add the certificate to ./propertyFile/ssl-certs                                      │
│ 4. If ICC for email was enabled, then make sure masterkey.txt file has been added under ./propertyFile/icc │
│ 5. If trusted certificates are needed, add them to ./propertyFile/trusted-certs                            │
│ 6. All SSL and trusted certificates need to be in PEM (Privacy Enhanced Mail) format                       │
│ 7. Run the following command to generate SQL, secrets and CR file                                          │
│                                                                                                            │
╰────────────────────────────────────────────────────────────────────────────────────────────────────────────╯
╭───────────────────────────────────╮  
│ python3 prerequisites.py generate │  
╰───────────────────────────────────╯  
╭────────────────────────────╮ 
│ Selection Summary          │ 
│ ├── License Model          │ 
│ │   └── CP4BA.NonProd      │ 
│ â── Platform               │ 
│ │   └── OCP                │ 
│ ├── Components             │ 
│ │   ├── cpe                │ 
│ │   ├── graphql            │ 
│ │   ├── ban                │ 
│ │   ├── css                │ 
│ │   ├── cmis               │ 
│ │   └── tm                 │ 
│ ├── Content Initialization │ 
│ │   └── True               │ 
│ └── Content Verification   │ 
│     └── True               │ 
╰────────────────────────────╯ 
                                                                                                                                        ╭──────────────────────────╮                                                                                                            
│ Property Files Structure │                               
╰──────────────────────────╯                               
   propertyFiles                                          
├──   ssl-certs                                           
│   └──   trusted-certs                                   
├──  fncm_components_options.toml (599 bytes)             
├──  fncm_db_server.toml (2.6 kB)                         
├──  fncm_deployment.toml (1.4 kB)                        
├──  fncm_ldap_server.toml (1.5 kB)                       
└──  fncm_user_group.toml (1.7 kB)                        
╭──────────────────────────────────────────────────╮       
│                Database Selection                │       
│ ┏━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━┓ │       
│ ┃       Type ┃ No. Object Stores ┃ SSL Enabled ┃ │       
│ ┡━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━┩ │       
│ │ postgresql │ 1                 │       False │ │       
│ └────────────┴───────────────────┴─────────────┘ │       
╰──────────────────────────────────────────────────╯       
╭────────────────────────────────────────────────────────╮ 
│                     LDAP Selection                     │ 
│ ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━┳━━━━━━━━━━━━━┓ │ 
│ ┃                          Type ┃ ID   ┃ SSL Enabled ┃ │ 
│ ┡━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━╇━━━━━━━━━━━━━┩ │ 
│ │ IBM Security Directory Server │ ldap │       False │ │ 
│ └───────────────────────────────┴──────┴─────────────┘ │ 
╰────────────────────────────────────────────────────────╯ 
              
```

#### Edit property files

Update values for fncm_components_options.toml

```bash
# Backup generated file
cp /usr/install/fncm/ibm-cp-fncm-case/inventory/\
fncmOperator/files/deploy/crs/container-samples/scripts/\
prerequisites/propertyFile/fncm_components_options.toml \
/usr/install/fncm/ibm-cp-fncm-case/inventory/\
fncmOperator/files/deploy/crs/container-samples/scripts/\
prerequisites/propertyFile/fncm_components_options.toml.bak

# Update generated file with real values
sed -i \
-e 's/taskAdmins/cpadmins/g' \
-e 's/taskUsers/cpusers/g' \
-e 's/taskAuditors/cpadmins/g' \
/usr/install/fncm/ibm-cp-fncm-case/inventory/\
fncmOperator/files/deploy/crs/container-samples/scripts/\
prerequisites/propertyFile/fncm_components_options.toml
```

Update values for fncm_db_server.toml

```bash
# Backup generated file
cp /usr/install/fncm/ibm-cp-fncm-case/inventory/\
fncmOperator/files/deploy/crs/container-samples/scripts/\
prerequisites/propertyFile/fncm_db_server.toml \
/usr/install/fncm/ibm-cp-fncm-case/inventory/\
fncmOperator/files/deploy/crs/container-samples/scripts/\
prerequisites/propertyFile/fncm_db_server.toml.bak
```

```bash
# Update generated file with real values
sed -i \
-e 's/DATABASE_SERVERNAME = "<Required>"/'\
'DATABASE_SERVERNAME = "postgresql.fncm-postgresql.svc.cluster.local"/g' \
-e '0,/DATABASE_NAME = "<Required>"/s//DATABASE_NAME = "devgcd"/' \
-e '0,/DATABASE_USERNAME = """<Required>"""/s//DATABASE_USERNAME = """devgcd"""/' \
-e '0,/DATABASE_PASSWORD = """<Required>"""/s//DATABASE_PASSWORD = """Password"""/' \
-e '0,/DATABASE_NAME = "<Required>"/s//DATABASE_NAME = "devos1"/' \
-e '0,/DATABASE_USERNAME = """<Required>"""/s//DATABASE_USERNAME = """devos1"""/' \
-e '0,/DATABASE_PASSWORD = """<Required>"""/s//DATABASE_PASSWORD = """Password"""/' \
-e '0,/DATABASE_NAME = "<Required>"/s//DATABASE_NAME = "devicn"/' \
-e '0,/DATABASE_USERNAME = """<Required>"""/s//DATABASE_USERNAME = """devicn"""/' \
-e '0,/DATABASE_PASSWORD = """<Required>"""/s//DATABASE_PASSWORD = """Password"""/' \
-e '0,/TABLESPACE_NAME = "ICNDB"/s//TABLESPACE_NAME = "devicn_tbs"/' \
-e '0,/SCHEMA_NAME = "ICNDB"/s//SCHEMA_NAME = "devicn"/' \
/usr/install/fncm/ibm-cp-fncm-case/inventory/\
fncmOperator/files/deploy/crs/container-samples/scripts/\
prerequisites/propertyFile/fncm_db_server.toml
```

Update values for fncm_deployment.toml

```bash
# Backup generated file
cp /usr/install/fncm/ibm-cp-fncm-case/inventory/\
fncmOperator/files/deploy/crs/container-samples/scripts/\
prerequisites/propertyFile/fncm_deployment.toml \
/usr/install/fncm/ibm-cp-fncm-case/inventory/\
fncmOperator/files/deploy/crs/container-samples/scripts/\
prerequisites/propertyFile/fncm_deployment.toml.bak
```

```bash
# Update generated file with real values
sed -i \
-e 's/SLOW_FILE_STORAGE_CLASSNAME = "<Required>"/'\
'SLOW_FILE_STORAGE_CLASSNAME = "ocs-storagecluster-cephfs"/g' \
-e 's/MEDIUM_FILE_STORAGE_CLASSNAME = "<Required>"/'\
'MEDIUM_FILE_STORAGE_CLASSNAME = "ocs-storagecluster-cephfs"/g' \
-e 's/FAST_FILE_STORAGE_CLASSNAME = "<Required>"/'\
'FAST_FILE_STORAGE_CLASSNAME = "ocs-storagecluster-cephfs"/g' \
/usr/install/fncm/ibm-cp-fncm-case/inventory/\
fncmOperator/files/deploy/crs/container-samples/scripts/\
prerequisites/propertyFile/fncm_deployment.toml
```

Update values for fncm_ldap_server.toml

```bash
# Backup generated file
cp /usr/install/fncm/ibm-cp-fncm-case/inventory/\
fncmOperator/files/deploy/crs/container-samples/scripts/\
prerequisites/propertyFile/fncm_ldap_server.toml \
/usr/install/fncm/ibm-cp-fncm-case/inventory/\
fncmOperator/files/deploy/crs/container-samples/scripts/\
prerequisites/propertyFile/fncm_ldap_server.toml.bak
```

```bash
# Update generated file with real values
sed -i \
-e 's/LDAP_SERVER = "<Required>"/LDAP_SERVER = "openldap.fncm-openldap.svc.cluster.local"/g' \
-e 's/LDAP_BASE_DN = "<Required>"/LDAP_BASE_DN = "dc=cp,dc=internal"/g' \
-e 's/LDAP_GROUP_BASE_DN = "<Required>"/LDAP_GROUP_BASE_DN = "ou=Groups,dc=cp,dc=internal"/g' \
-e 's/LDAP_BIND_DN = """<Required>"""/LDAP_BIND_DN = """cn=admin,dc=cp,dc=internal"""/g' \
-e 's/LDAP_BIND_DN_PASSWORD = """<Required>"""/LDAP_BIND_DN_PASSWORD = """Password"""/g' \
-e 's/LDAP_USER_NAME_ATTRIBUTE = "\*:uid"/LDAP_USER_NAME_ATTRIBUTE = "*:cn"/g' \
-e 's/LC_USER_FILTER = "(\&(uid=%v)(objectclass=person))"/'\
'LC_USER_FILTER = "(\&(uid=%v)(objectclass=inetOrgPerson))"/g' \
-e 's/LC_GROUP_FILTER = "(\&(cn=%v)(|(objectclass=groupofnames)(objectclass=groupofuniquenames)'\
'(objectclass=groupofurls)))"/'\
'LC_GROUP_FILTER = "(\&(cn=%v)(|(objectclass=groupofnames)(objectclass=groupofuniquenames)))"/g' \
/usr/install/fncm/ibm-cp-fncm-case/inventory/\
fncmOperator/files/deploy/crs/container-samples/scripts/\
prerequisites/propertyFile/fncm_ldap_server.toml
```

Update values for fncm_user_group.toml

```bash
# Backup generated file
cp /usr/install/fncm/ibm-cp-fncm-case/inventory/\
fncmOperator/files/deploy/crs/container-samples/scripts/\
prerequisites/propertyFile/fncm_user_group.toml \
/usr/install/fncm/ibm-cp-fncm-case/inventory/\
fncmOperator/files/deploy/crs/container-samples/scripts/\
prerequisites/propertyFile/fncm_user_group.toml.bak
```

```bash
# Update generated file with real values
sed -i \
-e 's/KEYSTORE_PASSWORD = """<Required>"""/KEYSTORE_PASSWORD = """Password"""/g' \
-e 's/LTPA_PASSWORD = """<Required>"""/LTPA_PASSWORD = """Password"""/g' \
-e 's/FNCM_LOGIN_USER = """<Required>"""/FNCM_LOGIN_USER = """cpadmin"""/g' \
-e 's/FNCM_LOGIN_PASSWORD = """<Required>"""/FNCM_LOGIN_PASSWORD = """Password"""/g' \
-e 's/ICN_LOGIN_USER = """<Required>"""/ICN_LOGIN_USER = """cpadmin"""/g' \
-e 's/ICN_LOGIN_PASSWORD = """<Required>"""/ICN_LOGIN_PASSWORD = """Password"""/g' \
-e 's/GCD_ADMIN_USER_NAME = \["""<Required>"""\]/GCD_ADMIN_USER_NAME = \["""cpadmin"""\]/g' \
-e 's/GCD_ADMIN_GROUPS_NAME = \["""<Required>"""\]/GCD_ADMIN_GROUPS_NAME = \["""cpadmins"""\]/g' \
-e 's/CPE_OBJ_STORE_OS_ADMIN_USER_GROUPS = \["""<Required>"""\]/'\
'CPE_OBJ_STORE_OS_ADMIN_USER_GROUPS = \["""cpadmin""","""cpadmins"""\]/g' \
/usr/install/fncm/ibm-cp-fncm-case/inventory/\
fncmOperator/files/deploy/crs/container-samples/scripts/\
prerequisites/propertyFile/fncm_user_group.toml
```

#### Generate phase - generate sqls, secrets, fncm yaml

```bash
python3 prerequisites.py generate
```

```text
╭─ FileNet Content Manager Deployment Prerequisites CLI ─╮
│ Version: 2.4.2                                         │
│ Mode: Generate                                         │
╰────────────────────────────────────────────────────────╯

╭──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╮
│                                                                                                                         Files Generated Successfully                                                                                                                         │
│                                                                                                                                                                                                                                                                              │
╰──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯
╭────────────╮                                                                            ╭───────────────────────────╮                                                                                                                                                         
│ Next Steps │                                                                            │ Generated Files Structure │                                                                                                                                                         
╰────────────╯                                                                            ╰───────────────────────────╯                                                                                                                                                         
╭───────────────────────────────────────────────╮                                            generatedFiles                                                                                                                                                                    
│ 1. Review the Generated files:                │                                         ├──   database                                                                                                                                                                       
│   - Database SQL files                        │                                         │   ├──  createGCD.sql (1.1 kB)                                                                                                                                                      
│   - Deployment Secrets                        │                                         │   ├──  createICN.sql (1.1 kB)                                                                                                                                                      
│   - SSL Certs in yaml format                  │                                         │   └──  createos.sql (1.1 kB)                                                                                                                                                       
│   - Custom Resource (CR) file                 │                                         ├──   secrets                                                                                                                                                                        
│ 2. Use the SQL files to create the databases  │                                         │   ├── 󱃾 ibm-ban-secret.yaml (285 bytes)                                                                                                                                             
│ 3. Run the following command to validate      │                                         │   ├── 󱃾 ibm-fncm-secret.yaml (331 bytes)                                                                                                                                            
│                                               │                                         │   └── 󱃾 ldap-bind-secret.yaml (177 bytes)                                                                                                                                           
╰───────────────────────────────────────────────╯                                         ├──   ssl                                                                                                                                                                            
╭───────────────────────────────────╮                                                     └── 󱃾 ibm_fncm_cr_production.yaml (18.1 kB)                                                                                                                                           
│ python3 prerequisites.py validate │                                                                                                                                                                                                                                           
╰───────────────────────────────────╯                                                                                                                                                                                                                                           
                                      
```

#### Apply SQLs to DB instance

Update ICN tablespace and schema names (TODO needed on 2024-02-05 for FNCM 5.5.12 for CASE package 1.8.0)
```bash
sed -i \
-e 's/tablespace ICNDB/tablespace devicn_tbs/g' \
-e 's/SCHEMA IF NOT EXISTS ICNDB/SCHEMA IF NOT EXISTS devicn/g' \
-e 's/search_path TO ICNDB/search_path TO devicn/g' \
/usr/install/fncm/ibm-cp-fncm-case/inventory/\
fncmOperator/files/deploy/crs/container-samples/scripts/\
prerequisites/generatedFiles/database/createICN.sql
```

Copy create scripts to PostgreSQL instance
```bash
oc cp /usr/install/fncm/ibm-cp-fncm-case/inventory/\
fncmOperator/files/deploy/crs/container-samples/scripts/prerequisites/generatedFiles/database \
fncm-postgresql/$(oc get pods --namespace fncm-postgresql -o name | cut -d"/" -f2):/usr/dbscript
```

Execute create scripts with table space directory creation
```bash
# Navigator
oc --namespace fncm-postgresql exec statefulset/postgresql -- /bin/bash -c \
'mkdir /pgsqldata/devicn; chown postgres:postgres /pgsqldata/devicn;'
oc --namespace fncm-postgresql exec statefulset/postgresql -- /bin/bash -c \
'psql postgresql://cpadmin@localhost:5432/postgresdb \
--file=/usr/dbscript/createICN.sql'

# FNCM OS
oc --namespace fncm-postgresql exec statefulset/postgresql -- /bin/bash -c \
'mkdir /pgsqldata/devos1; chown postgres:postgres /pgsqldata/devos1;'
oc --namespace fncm-postgresql exec statefulset/postgresql -- /bin/bash -c \
'psql postgresql://cpadmin@localhost:5432/postgresdb \
--file=/usr/dbscript/createos.sql'

# FNCM GCD
oc --namespace fncm-postgresql exec statefulset/postgresql -- /bin/bash -c \
'mkdir /pgsqldata/devgcd; chown postgres:postgres /pgsqldata/devgcd;'
oc --namespace fncm-postgresql exec statefulset/postgresql -- /bin/bash -c \
'psql postgresql://cpadmin@localhost:5432/postgresdb \
--file=/usr/dbscript/createGCD.sql'
```

#### Modify CR file and Network POlicies

Change LDAP type to Custom as we have OpenLDAP
```bash
# Backup generated file
cp /usr/install/fncm/ibm-cp-fncm-case/inventory/\
fncmOperator/files/deploy/crs/container-samples/scripts/\
prerequisites/generatedFiles/ibm_fncm_cr_production.yaml \
/usr/install/fncm/ibm-cp-fncm-case/inventory/\
fncmOperator/files/deploy/crs/container-samples/scripts/\
prerequisites/generatedFiles/ibm_fncm_cr_production.yaml.bak
```

```bash
# Change LDAP type
sed -i \
-e 's/lc_selected_ldap_type: IBM Security Directory Server/lc_selected_ldap_type: Custom/g' \
-e 's/tds:/custom:/g' \
/usr/install/fncm/ibm-cp-fncm-case/inventory/\
fncmOperator/files/deploy/crs/container-samples/scripts/\
prerequisites/generatedFiles/ibm_fncm_cr_production.yaml
```

Add permissive network policy to enable the deployment to reach to LDAP and DB (TODO needed on 2024-02-05 for FNCM 5.5.12 for CASE package 1.8.0)
```bash
echo "
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: custom-permit-db-egress
  namespace: fncm
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector: {}
          namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: fncm-postgresql
" | oc apply -f -
echo "
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: custom-permit-ldap-egress
  namespace: fncm
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector: {}
          namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: fncm-openldap
" | oc apply -f -
```

If you want to see all debug output in operator logs add the following properties (Not for production use, can leak sensitive info!)
```bash
yq -i '.spec.shared_configuration.no_log = false' \
/usr/install/fncm/ibm-cp-fncm-case/inventory/\
fncmOperator/files/deploy/crs/container-samples/scripts/\
prerequisites/generatedFiles/ibm_fncm_cr_production.yaml
yq -i '.spec.shared_configuration.show_sensitive_log = true' \
/usr/install/fncm/ibm-cp-fncm-case/inventory/\
fncmOperator/files/deploy/crs/container-samples/scripts/\
prerequisites/generatedFiles/ibm_fncm_cr_production.yaml
```

#### Validate phase - validate connectivity and apply artifacts

Change project
```bash
oc project fncm
```

Run the validation script which validates the connectivity, applies artifacts including CR
```bash
python3 prerequisites.py validate
```

```text

╭─ FileNet Content Manager Deployment Prerequisites CLI ─╮
│ Version: 2.4.2                                         │
│ Mode: Validate                                         │
╰────────────────────────────────────────────────────────╯

╭────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── FNCM Standalone Operator ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╮
│ ╭───────────────────────────────── Hint ──────────────────────────────────╮ ╭─────────────────────────────── Command ────────────────────────────────╮                                                                                                                       │
│ │ - Run the validation from the FNCM Standalone Operator                  │ │ cd ..                                                                  │                                                                                                                       │
│ │ - All tools and libraries are installed                                 │ │ export OPERATOR=$(kubectl get pods | grep operator | awk '{print $1}') │                                                                                                                       │
│ │ - Validation from within the your cluster can test private connections  │ │ kubectl cp prerequisites $OPERATOR:/opt/ansible                        │                                                                                                                       │
│ │ - See the below command to copy the folder and run the validation.      │ │ kubectl exec -it $OPERATOR -- bash                                     │                                                                                                                       │
│ ╰─────────────────────────────────────────────────────────────────────────╯ │ cd /opt/ansible                                                        │                                                                                                                       │
│                                                                             │ python3 prerequisites.py validate                                      │                                                                                                                       │
│                                                                             ╰────────────────────────────────────────────────────────────────────────╯                                                                                                                       │
╰───────â──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯

[09:08:18] ╭─────────────────────────────────────────────────────╮                                                                                                                                                                                              validate.py:1095
           │ Validating storage class: ocs-storagecluster-cephfs │                                                                                                                                                                                                              
           ╰─────────────────────────────────────────────────────╯                                                                                                                                                                                                              
[09:08:19]                                                                                                                                                                                                                                                      validate.py:1148
           Verification for Storage Class: "ocs-storagecluster-cephfs" PASSED!                                                                                                                                                                                  validate.py:1149
                                                                                                                                                                                                                                                                                
                                                                                                                                                                                                                                                                validate.py:1181
           Sample PVC created with storage class: ocs-storagecluster-cephfs                                                                                                                                                                                     validate.py:1182
                                                                                                                                                                                                                                                                validate.py:1105
           Checking for fncm-test-pvc liveness - Attempt 1/30                                                                                                                                                                                                                   
                                                                                                                                                                                                                                                                                
[09:08:20]                                                                                                                                                                                                                                                      validate.py:1114
           "fncm-test-pvc" not yet found, waiting 10 seconds to retry                                                                                                                                                                                                           
[09:08:30]                                                                                                                                                                                                                                                      validate.py:1105
           Checking for fncm-test-pvc liveness - Attempt 2/30                                                                                                                                                                                                                   
                                                                                                                                                                                                                                                                                
                                                                                                                                                                                                                                                                validate.py:1123
           Verification for PVC: "fncm-test-pvc" PASSED!                                                                                                                                                                                                        validate.py:1124
                                                                                                                                                                                                                                                                                
           ╭────────────────────────────────────────────────────────────────────────────────────────────╮                                                                                                                                                        validate.py:271
           │ Ensure Postgresql Max Transactions has been configured.                                    │                                                                                                                                                                       
           │ Please see https://www.ibm.com/docs/SSNW2F_5.5.12/com.ibm.p8.performance.doc/p8ppi308.htm. │                                                                                                                                                                       
           ╰────────────────────────────────────────────────────────────────────────────────────────────╯                                                                                                                                                                       
                                                                                                                                                                                                                                                                 validate.py:272
           ╭────────────────────────────────────╮                                                                                                                                                                                                                validate.py:299
           │ Validating GCD Database Connection │                                                                                                                                                                                                                               
           ╰────────────────────────────────────╯                                                                                                                                                                                                                               
                                                                                                                                                                                                                                                                 validate.py:300
           Validating Server "postgresql.fncm-postgresql.svc.cluster.local" Reachability                                                                                                                                                                         validate.py:993
[09:08:31]                                                                                                                                                                                                                                                       validate.py:994
                                                                                                                                                                                                                                                                validate.py:1019
           Reachability to "postgresql.fncm-postgresql.svc.cluster.local" succeeded!                                                                                                                                                                                            
                                                                                                                                                                                                                                                                                
                                                                                                                                                                                                                                                                validate.py:1020
[09:08:33]                                                                                                                                                                                                                                                       validate.py:489
           Checked DB connection for "gcd" on database server "postgresql.fncm-postgresql.svc.cluster.local", PASSED!                                                                                                                                                           
                                                                                                                                                                                                                                                                                
                                                                                                                                                                                                                                                                 validate.py:490
           Detected Connection Latency: 56.33ms                                                                                                                                                                                                                 validate.py:1064
           Potential Failure Latency Range: > 30ms                                                                                                                                                                                                              validate.py:1065
                                                                                                                                                                                                                                                                validate.py:1066
           ╭───────────────────────────────────╮                                                                                                                                                                                                                 validate.py:304
           │ Validating OS Database Connection │                                                                                                                                                                                                                                
           ╰───────────────────────────────────╯                                                                                                                                                                                                                                
                                                                                                                                                                                                                                                                 validate.py:305
           Validating Server "postgresql.fncm-postgresql.svc.cluster.local" Reachability                                                                                                                                                                         validate.py:993
                                                                                                                                                                                                                                                                 validate.py:994
[09:08:34]                                                                                                                                                                                                                                                      validate.py:1019
           Reachability to "postgresql.fncm-postgresql.svc.cluster.local" succeeded!                                                                                                                                                                                            
                                                                                                                                                                                                                                                                                
                                                                                                                                                                                                                                                                validate.py:1020
[09:08:37]                                                                                                                                                                                                                                                       validate.py:489
           Checked DB connection for "os" on database server "postgresql.fncm-postgresql.svc.cluster.local", PASSED!                                                                                                                                                            
                                                                                                                                                                                                                                                                                
                                                                                                                                                                                                                                                                 validate.py:490
           Detected Connection Latency: 24.01ms                                                                                                                                                                                                                 validate.py:1064
           Performance Degradation Latency Range: 10ms - 30ms                                                                                                                                                                                                   validate.py:1065
                                                                                                                                                                                                                                                                validate.py:1066
           ╭────────────────────────────────────╮                                                                                                                                                                                                                validate.py:310
           │ Validating ICN Database Connection │                                                                                                                                                                                                                               
           ╰────────────────────────────────────╯                                                                                                                                                                                                                               
                                                                                                                                                                                                                                                                 validate.py:311
           Validating Server "postgresql.fncm-postgresql.svc.cluster.local" Reachability                                                                                                                                                                         validate.py:993
                                                                                                                                                                                                                                                                 validate.py:994
                                                                                                                                                                                                                                                                validate.py:1019
           Reachability to "postgresql.fncm-postgresql.svc.cluster.local" succeeded!                                                                                                                                                                                            
                                                                                                                                                                                                                                                                                
[09:08:38]                                                                                                                                                                                                                                                      validate.py:1020
[09:08:40]                                                                                                                                                                                                                                                       validate.py:489
           Checked DB connection for "icn" on database server "postgresql.fncm-postgresql.svc.cluster.local", PASSED!                                                                                                                                                           
                                                                                                                                                                                                                                                                                
                                                                                                                                                                                                                                                                 validate.py:490
           Detected Connection Latency: 24.45ms                                                                                                                                                                                                                 validate.py:1064
           Performance Degradation Latency Range: 10ms - 30ms                                                                                                                                                                                                   validate.py:1065
                                                                                                                                                                                                                                                                validate.py:1066
           ╭──────────────────────────────╮                                                                                                                                                                                                                      validate.py:613
           │ LDAP Server Validation: LDAP │                                                                                                                                                                                                                                     
           ╰──────────────────────────────╯                                                                                                                                                                                                                                     
                                                                                                                                                                                                                                                                 validate.py:614
           Validating Server "openldap.fncm-openldap.svc.cluster.local" Reachability                                                                                                                                                                             validate.py:993
                                                                                                                                                                                                                                                                 validate.py:994
                                                                                                                                                                                                                                                                validate.py:1019
           Reachability to "openldap.fncm-openldap.svc.cluster.local" succeeded!                                                                                                                                                                                                
                                                                                                                                                                                                                                                                                
                                                                                                                                                                                                                                                                validate.py:1020
           Detected Connection Latency: 25.71ms                                                                                                                                                                                                                 validate.py:1064
           Acceptable Latency Range: 0ms - 100ms                                                                                                                                                                                                                validate.py:1065
                                                                                                                                                                                                                                                                validate.py:1066
           Testing Authentication of "openldap.fncm-openldap.svc.cluster.local" with Bind DN: "cn=admin,dc=cp,dc=internal"                                                                                                                                       validate.py:795
                                                                                                                                                                                                                                                                 validate.py:796
           Successfully authenticated with "cn=admin,dc=cp,dc=internal"                                                                                                                                                                                          validate.py:802
                                                                                                                                                                                                                                                                 validate.py:803
           ╭────────────────────────────────────────╮                                                                                                                                                                                                            validate.py:652
           │ LDAP Users and Groups Validation Check │                                                                                                                                                                                                                           
           ╰────────────────────────────────────────╯                                                                                                                                                                                                                           
                                                                                                                                                                                                                                                                 validate.py:653
           Searching LDAP: "openldap.fncm-openldap.svc.cluster.local"                                                                                                                                                                                            validate.py:661
                                                                                                                                                                                                                                                                 validate.py:662
[09:08:41] ╭─ Users Search Results ─╮                                                                                                                                                                                                                            validate.py:675
           │      Users Found       │                                                                                                                                                                                                                                           
           │ ┏━━━━━━━━━┳━━━━━━━━━━┓ │                                                                                                                                                                                                                                           
           │ ┃ User    ┃ Found in ┃ │                                                                                                                                                                                                                                           
           │ ┡━━━━━━━━━╇━━━━━━━━━━┩ │                                                                                                                                                                                                                                           
           │ │ cpadmin │ LDAP     │ │                                                                                                                                                                                                                                           
           │ └─────────┴──────────┘ │                                                                                                                                                                                                                                           
           ╰────────────────────────╯                                                                                                                                                                                                                                           
           ╭─ Groups Search Results ─╮                                                                                                                                                                                                                                          
           │      Groups Found       │                                                                                                                                                                                                                                          
           │ ┏━━━━━━━━━━┳━━━━━━━━━━┓ │                                                                                                                                                                                                                                          
           │ ┃ Group    ┃ Found in ┃ │                                                                                                                                                                                                                                          
           │ ┡━━━━━━━━━━╇━━━━━━━━━━┩ │                                                                                                                                                                                                                                          
           │ │ cpadmins │ LDAP     │ │                                                                                                                                                                                                                                          
           │ │ cpusers  │ LDAP     │ │                                                                                                                                                                                                                                          
           │ └──────────┴──────────┘ │                                                                                                                                                                                                                                          
           ╰─────────────────────────╯                                                                                                                                                                                                                                          
           ╭──────────────────────────────────────╮                                                                                                                                                                                                                             
           │ ✅ All users and groups where found! │                                                                                                                                                                                                                             
           ╰──────────────────────────────────────╯                                                                                                                                                                                                                             
                                                                                                                                                                                                                                                                 validate.py:676
  Validate Storage Class         ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% 1/1 0:00:11
  Validate Database              ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% 3/3 0:00:21
  Validate LDAP                  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% 1/1 0:00:21
  Validate LDAP Users and Groups ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% 1/1 0:00:00

╭─────────────────────────────────╮
│ All prerequisites are validated │ #Everything should validate
╰─────────────────────────────────╯

Do you want to apply the SSL & Secrets? [y/n]: y
╭───────────────────────────────────╮
│ secret/ibm-fncm-secret configured │
╰───────────────────────────────────╯
╭──────────────────────────────────╮
│ secret/ibm-ban-secret configured │
╰──────────────────────────────────╯
╭────────────────────────────────────╮
│ secret/ldap-bind-secret configured │
╰────────────────────────────────────╯

Do you want to apply the CR? [y/n]: y

╭─────────────────────────────────────────────╮
│ fncmcluster.fncm.ibm.com/fncmdeploy created │
╰─────────────────────────────────────────────╯
```

#### Validating your deployment
 
Based on https://www.ibm.com/docs/en/filenet-p8-platform/5.5.12?topic=resource-validating-your-deployment

Wait for the deployment to be completed. This can be determined by looking in Project fncm in Kind FNCMCluster, instance named fncmdeploy to have the following components in the desired state:
```bash
oc get -n fncm FNCMCluster fncmdeploy -o jsonpath='{.status.components}' | jq
```

```json
{
  "ban_initialization": {
    "banAddDefaultDesktop": "Ready",
    "banAddDefaultPlugins": "Ready",
    "banAddRepository": "Ready"
  },
  "ban_verification": {
    "banCreateFolder": "Ready",
    "banDeleteDoc": "Ready",
    "banDeleteFolder": "Ready",
    "conditions": {
      "lastTransitionTime": "2024-02-05T14:36:48Z",
      "message": "",
      "reason": "",
      "type": ""
    }
  },
  "cmis": {
    "cmisDeployment": "NotReady",
    "cmisRoute": "NotReady",
    "cmisService": "NotReady",
    "cmisStorage": "NotReady",
    "lastTransitionTime": "2024-02-05T15:36:27Z",
    "message": "",
    "reason": ""
  },
  "content_initialization": {
    "cpeDomainInitialization": "Ready",
    "cpeObjectStoreInitializationObjectstoreName": "Ready",
    "cssIndexCreation": "Ready"
  },
  "content_verification": {
    "cmisVerification": "Ready",
    "conditions": {
      "lastTransitionTime": "2024-02-05T15:19:01Z",
      "message": "Verification Done",
      "reason": "Successful",
      "status": "True",
      "type": "Ready"
    },
    "cpeVerification": "Ready",
    "cssIndexVerification": "Ready"
  },
  "cpe": {
    "cpeDeployment": "Ready",
    "cpeJDBCDriver": "Ready",
    "cpeRoute": "Ready",
    "cpeService": "Ready",
    "cpeStorage": "Ready",
    "lastTransitionTime": "2024-02-05T15:36:27Z",
    "message": "",
    "reason": ""
  },
  "css": {
    "cssDeployment": "Ready",
    "cssService": "Ready",
    "cssStorage": "Ready",
    "lastTransitionTime": "2024-02-05T15:36:27Z",
    "message": "",
    "reason": ""
  },
  "extshare": {
    "extshareDeployment": "NotInstalled",
    "extshareRoute": "NotInstalled",
    "extshareService": "NotInstalled",
    "extshareStorage": "NotInstalled",
    "lastTransitionTime": "2024-02-05T15:19:01Z",
    "message": "",
    "reason": ""
  },
  "graphql": {
    "graphqlDeployment": "Ready",
    "graphqlRoute": "Ready",
    "graphqlService": "Ready",
    "graphqlStorage": "Ready",
    "lastTransitionTime": "2024-02-05T15:19:01Z",
    "message": "",
    "reason": ""
  },
  "navigator": {
    "lastTransitionTime": "2024-02-05T15:19:01Z",
    "message": "",
    "navigatorDeployment": "Ready",
    "navigatorService": "Ready",
    "navigatorStorage": "Ready",
    "reason": ""
  },
  "tm": {
    "lastTransitionTime": "2024-02-05T15:19:01Z",
    "message": "",
    "reason": "",
    "tmDeployment": "Ready",
    "tmRoute": "Ready",
    "tmService": "Ready",
    "tmStorage": "Ready"
  }
}
```

### License Metering

License Service can be installed following the documentation at https://www.ibm.com/docs/en/cloud-paks/foundational-services/4.4?topic=ils-installing-license-service-openshift-container-platform-version-410-later

## Contacts

Jan Dušek  
jdusek@cz.ibm.com  
Business Automation Partner Technical Specialist  
IBM Czech Republic

## Notice

© Copyright IBM Corporation 2024.
