
# Air-gapped installation of IBM Cloud Pak for Integration

This document contains notes collected during the installation of IBM Cloud Pak for Integration (CP4I) version 2021.4.1 in the existing demo environment with the Red Hat OpenShift cluster version 4.8.35.

Please see the following chapter from the documentation for the description of possible air-gapped scenarios:
https://www.ibm.com/docs/en/cloud-paks/cp-integration/2021.4?topic=installing-adding-catalog-sources-air-gapped-openshift-cluster

The air-gapped installation of CP4I is based on the Container Application Software for Enterprises (CASE) specification (https://github.com/ibm/case) 
The repository with all IBM cloud paks CASE-es is available here: https://github.com/IBM/cloud-pak/tree/master/repo/case and the repository with IBM CP4I CASE-es can be found here: https://github.com/IBM/cloud-pak/tree/master/repo/case/ibm-cp-integration


**Table of contents:**

- [1. Setup local registry](#registry-setup) 
- [2. Command line tools](#cli-tools)
- [3. Setup environment variables](#environment-variables)
- [4. Download CASE-es and mirror the images](#downloading-and-mirroring)
- [5. Update cluster configuration](#update-cluster)
- [6. Finalize cloud pak installation](#finalize-installation)



<a name="registry-setup"></a>
## 1. Setup local registry

A Docker V2 registry must be available and accessible from the OpenShift Container Platform cluster nodes. In the test installation described in this document, we didn't have an available container images registry. Since we had enough space, we decided to set it up on the bastion node. This chapter describes the setup steps. Please see the already mentioned documentation chapter (https://www.ibm.com/docs/en/cloud-paks/cp-integration/2021.4?topic=installing-adding-catalog-sources-air-gapped-openshift-cluster) for the local registry possibilities.


Check the hostname:
```
hostname
```
Result:
```
bastion.ocp48.tec.uk.ibm.com
```  

Available storage:
```
lsblk
```
Result:
```
NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda             8:0    0  108G  0 disk 
|-sda1          8:1    0    1G  0 part /boot
`-sda2          8:2    0  107G  0 part 
  |-rhel-root 253:0    0   95G  0 lvm  /
  |-rhel-swap 253:1    0 10.8G  0 lvm  [SWAP]
  `-rhel-home 253:2    0  1.2G  0 lvm  /home
sdb             8:16   0    1T  0 disk /nfs
sr0            11:0    1 1024M  0 rom   
```

Based on the above result, we decided to create a directory for all air-gapped artifacts as a subdirectory of `/nfs`.

Set environment variables:
```
export REGISTRY_SERVER=bastion.ocp48.tec.uk.ibm.com
export REGISTRY_PORT=5000
export LOCAL_REGISTRY="${REGISTRY_SERVER}:${REGISTRY_PORT}"
export REGISTRY_USER="admin"
export REGISTRY_PASSWORD="passw0rd"

export AIRGAP_DIR=/nfs/airgap
```

Prepare registry directories:
```
mkdir -p ${AIRGAP_DIR}/registry/{auth,certs,data,images}
```

Generate certificate:
```
cd ${AIRGAP_DIR}/registry/certs

openssl req -newkey rsa:4096 -nodes -sha256 -keyout registry.key -x509 -days 365 -out registry.crt -subj "/C=US/ST=/L=/O=/CN=$REGISTRY_SERVER"
```

Create a password for the registry:
```
htpasswd -bBc ${AIRGAP_DIR}/registry/auth/htpasswd $REGISTRY_USER $REGISTRY_PASSWORD
```

Download registry image:
```
podman pull docker.io/library/registry:2
podman save -o ${AIRGAP_DIR}/registry/images/registry-2.tar docker.io/library/registry:2
```

Download the NFS provisioner image:
```
podman pull quay.io/external_storage/nfs-client-provisioner:latest
podman save -o ${AIRGAP_DIR}/registry/images/nfs-client-provisioner.tar quay.io/external_storage/nfs-client-provisioner:latest
```

Create registry pod:
```
podman run --name mirror-registry --publish $REGISTRY_PORT:5000 \
   --detach \
   --volume ${AIRGAP_DIR}/registry/data:/var/lib/registry:z \
   --volume ${AIRGAP_DIR}/registry/auth:/auth:z \
   --volume ${AIRGAP_DIR}/registry/certs:/certs:z \
   --env "REGISTRY_AUTH=htpasswd" \
   --env "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
   --env REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
   --env REGISTRY_HTTP_TLS_CERTIFICATE=/certs/registry.crt \
   --env REGISTRY_HTTP_TLS_KEY=/certs/registry.key \
   docker.io/library/registry:2 
```

Add the certificate to the trust store:
```
cp -f ${AIRGAP_DIR}/registry/certs/registry.crt /etc/pki/ca-trust/source/anchors/

update-ca-trust
```

Test the connection to the registry:
```
curl -u $REGISTRY_USER:$REGISTRY_PASSWORD https://${LOCAL_REGISTRY}/v2/_catalog
```

Expected output:
```
{"repositories":[]}
```

>**Note:** If you restart the bastion node, don't forget to start the registry again:
>```
>podman start mirror-registry
>```

<a name="cli-tools"></a>
## 2. Command line tools

The following tools are needed:
- IBM Cloud Pak CLI: https://github.com/IBM/cloud-pak-cli
- Red Hat OpenShift CLI: https://docs.openshift.com/container-platform/4.8/cli_reference/openshift_cli/getting-started-cli.html
- Skopeo: https://github.com/containers/skopeo/blob/master/install.md
- OpenSSL: https://www.openssl.org/

The OCP CLI and OpenSSL are usually already installed on the bastion node:
```
oc version

# In our demo environment it was:
# Client Version: 4.8.35
# Server Version: 4.8.35
# Kubernetes Version: v1.21.8+ee73ea2

openssl version

# In our demo environment it was:
# OpenSSL 1.1.1k  FIPS 25 Mar 2021
```
You will need to install the IBM Cloud Pak CLI and Skopeo.


#### Install Cloud Pak CLI

Make sure to download the latest version: https://github.com/IBM/cloud-pak-cli/releases/latest
At the moment of writing this document the latest version was v3.17.0
```
wget https://github.com/IBM/cloud-pak-cli/releases/download/v3.17.0/cloudctl-linux-amd64.tar.gz
tar -xf cloudctl-linux-amd64.tar.gz
chmod 755 cloudctl-linux-amd64
mv cloudctl-linux-amd64 /usr/local/bin/cloudctl
cloudctl version
```

#### Install Skopeo

Install CLI 0.2.0 or higher: https://github.com/containers/skopeo/blob/master/install.md
```
sudo dnf -y install skopeo
```
<a name="environment-variables"></a>
## 3. Setup environment variables

>Please read the following description:

Some of the environment variables used in this example are already defined above during the registry setup. For the sake of consistency, we are defining them once again. Please make sure that the values of the **REGISTRY_...** variables reflect your local registry setup.

The variable **ENTITLEMENT_KEY** contains a key needed for pulling the container images from the IBM registry. The key can be obtained here: https://myibm.ibm.com/products-services/containerlibrary

For the offline installation, we are using **CASE** bundles. The variable **CASE_ARCHIVE** refers to the CASE file. Please make sure that it contains the correct version. You can check the Cloud Pak for Integration CASE versions here: https://github.com/IBM/cloud-pak/tree/master/repo/case/ibm-cp-integration At the moment of writing this document, the latest version was *3.0.0*

The variable **OFFLINE_DIR** contains the location where CASE files will be stored. In your environment, it could be, of course, different.

```
export REGISTRY_SERVER=bastion.ocp48.tec.uk.ibm.com
export REGISTRY_PORT=5000
export LOCAL_REGISTRY="${REGISTRY_SERVER}:${REGISTRY_PORT}"
export REGISTRY_USER="admin"
export REGISTRY_PASSWORD="passw0rd"
export OFFLINE_DIR=/nfs/airgap/artifacts
export CASE_ARCHIVE=ibm-cp-integration-3.0.0.tgz
export CASE_INVENTORY_SETUP=operator
export NAMESPACE=cp4i
export ENTITLEMENT_KEY= # ...your entitlement key...
```

>**IMPORTANT NOTE:** We specify here the "top-level" CASE archive (*ibm-cp-integration-3.0.0.tgz*) that will cause downloading the CASE-es of all components and later mirroring all of the images related to the CP4I. Instead, it is possible to select the individual CASEs just for the needed components. Please see the section **3.1 Mirroring individual capabilities** in the following chapter of the CP4I documentation: https://www.ibm.com/docs/en/cloud-paks/cp-integration/2021.4?topic=cluster-adding-catalog-sources-bastion-host and the content of the CASE repository: https://github.com/IBM/cloud-pak/tree/master/repo/case <br>
In this case, you should repeat each of the commands described below where the variable $CASE_ARCHIVE appears for each of the individual products. You should also correct the CASE_INVENTORY_SETUP variable.
For example, if you want to install just Platform Navigator, MQ and App Connect, the values for the latest versions are:
>- Platform Navigator
>   ```
>   export CASE_ARCHIVE=ibm-integration-platform-navigator-1.6.0.tgz
>   export CASE_INVENTORY_SETUP=platformNavigatorOperator
>   ```
>- IBM App Connect
>   ```
>   export CASE_ARCHIVE=ibm-appconnect-4.0.0.tgz
>   export CASE_INVENTORY_SETUP=ibmAppconnect
>   ```
>- IBM MQ
>   ```
>   export CASE_ARCHIVE=ibm-mq-1.8.0.tgz
>   export CASE_INVENTORY_SETUP=ibmMQOperator
>   ```



<a name="downloading-and-mirroring"></a>
## 4. Download CASE-es and mirror the images

If it does not already exist, create the directory where the CASE files will be stored:
```
mkdir $OFFLINE_DIR
```

Download CASE files:
```
cloudctl case save \
  --case https://github.com/IBM/cloud-pak/raw/master/repo/case/$CASE_ARCHIVE \
  --outputdir $OFFLINE_DIR/ 
```

Check the content of the offline directory:
```
ls -l $OFFLINE_DIR 
```
It must be similar to the following example:
```
-rw-r--r-- 1 root root     785 Apr 13 07:48 caseDependencyMapping.csv
drwxr-xr-x 2 root root       6 Apr 13 07:47 charts
-rw-r--r-- 1 root root      32 Apr 13 07:48 ibm-ai-wmltraining-1.1.1-charts.csv
-rw-r--r-- 1 root root    1460 Apr 13 07:48 ibm-ai-wmltraining-1.1.1-images.csv
-rw-r--r-- 1 root root   83215 Apr 13 07:48 ibm-ai-wmltraining-1.1.1.tgz
-rw-r--r-- 1 root root      32 Apr 13 07:48 ibm-apiconnect-3.0.6-charts.csv
-rw-r--r-- 1 root root  124950 Apr 13 07:48 ibm-apiconnect-3.0.6-images.csv
-rw-r--r-- 1 root root  819521 Apr 13 07:47 ibm-apiconnect-3.0.6.tgz
-rw-r--r-- 1 root root      32 Apr 13 07:48 ibm-appconnect-4.0.0-charts.csv
-rw-r--r-- 1 root root  253993 Apr 13 07:48 ibm-appconnect-4.0.0-images.csv
-rw-r--r-- 1 root root 1604016 Apr 13 07:48 ibm-appconnect-4.0.0.tgz

...

```

Configure credentials for the IBM Entitled Registry:
```
cloudctl case launch \
  --case $OFFLINE_DIR/$CASE_ARCHIVE \
  --inventory $CASE_INVENTORY_SETUP \
  --action configure-creds-airgap \
  --args "--registry cp.icr.io --user cp --pass $ENTITLEMENT_KEY --inputDir $OFFLINE_DIR"    
```
The JSON file with base64 encoded credentials is stored in `$HOME/.airgap/secrets` directory.


Configure credentials for your local container images registry:
```
cloudctl case launch \
  --case $OFFLINE_DIR/$CASE_ARCHIVE \
  --inventory $CASE_INVENTORY_SETUP \
  --action configure-creds-airgap \
  --args "--registry $LOCAL_REGISTRY --user $REGISTRY_USER --pass $REGISTRY_PASSWORD"
```
The JSON file with base64 encoded credentials is stored in `$HOME/.airgap/secrets` directory.


Start mirroring images:
>**Note:** The following process copies all images defined in the downloaded CASE files from IBM to the local registry. It can take a lot of time before it is completed. Consider using `screen` or any similar solution to avoid problems if you lose the SSH session.

```
cloudctl case launch \
  --case $OFFLINE_DIR/$CASE_ARCHIVE \
  --inventory $CASE_INVENTORY_SETUP \
  --action mirror-images \
  --args "--registry $LOCAL_REGISTRY --inputDir $OFFLINE_DIR"
```

<a name="update-cluster"></a>
## 5. Update cluster configuration

We have to inform OpenShift to pull the images from the mirrored registry instead of the original one. In order to do that we have to create *ImageContentSourcePolicy*.

Log in to the OpenShift if not already:
```
oc login -u <your_ocp_admin_username> -p <your_ocp_admin_password> ...
```

The *case* command requires a namespace parameter despite nothing is actually created in that namespace. Let's create an OpenShift project using previously defined *NAMESPACE* variable (in our case the project name is *cp4i*):
```
oc new-project $NAMESPACE
```

Create an *ImageContentSourcePolicy*:
```
cloudctl case launch \
  --case $OFFLINE_DIR/$CASE_ARCHIVE \
  --inventory $CASE_INVENTORY_SETUP \
  --action configure-cluster-airgap \
  --namespace $NAMESPACE \
  --args "--registry $LOCAL_REGISTRY --inputDir $OFFLINE_DIR"
```

Verify that ImageContentSourcePolicy is created
```
oc get ImageContentSourcePolicy
```
The response should be similar to the following:
```
NAME                 AGE
ibm-cp-integration   3m26s
```

Optional step: if necessary add the local registry to the *insecureRegistries* list:
```
oc patch image.config.openshift.io/cluster --type=merge \
  -p '{"spec":{"registrySources":{"insecureRegistries":["'${LOCAL_REGISTRY}'"]}}}'
```


**!!! VERY IMPORTANT !!!:** The nodes update will start. During that process, the nodes will show a *SchedulingDisabled* state. It is not necessary that the process starts immediately. Wait until the first node shows that state and then **wait until all nodes are back in the *Ready* state**. The whole process takes around 30 minutes.

You can watch the nodes' states with the following command:
```
watch -n 10 oc get nodes
```

While the update is going on, you will see the result similar to the following example:

```
NAME                            STATUS                     ROLES    AGE   VERSION
master-1.ocp48.tec.uk.ibm.com   Ready                      master   12d   v1.21.8+ee73ea2
master-2.ocp48.tec.uk.ibm.com   Ready                      master   12d   v1.21.8+ee73ea2
master-3.ocp48.tec.uk.ibm.com   Ready                      master   12d   v1.21.8+ee73ea2
worker-1.ocp48.tec.uk.ibm.com   Ready,SchedulingDisabled   worker   12d   v1.21.8+ee73ea2
worker-2.ocp48.tec.uk.ibm.com   Ready                      worker   12d   v1.21.8+ee73ea2
worker-3.ocp48.tec.uk.ibm.com   Ready                      worker   12d   v1.21.8+ee73ea2
worker-4.ocp48.tec.uk.ibm.com   Ready,SchedulingDisabled   worker   12d   v1.21.8+ee73ea2
worker-5.ocp48.tec.uk.ibm.com   Ready                      worker   12d   v1.21.8+ee73ea2
worker-6.ocp48.tec.uk.ibm.com   Ready                      worker   12d   v1.21.8+ee73ea2
```

The alternative is to watch the Machine Config Pools state:
```
oc get mcp
```

They show the UPDATING is True:
```
NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master   rendered-master-1a1eedc4be6d0681776102ec0b4170a0   False     True       False      3              2                   2                     0                      12d
worker   rendered-worker-206a8cfe7891428623f98b1ee2b5c884   False     True       False      6              4                   4                     0                      12d
```


<a name="finalize-installation"></a>
## 6. Finalize cloud pak installation

Import catalog sources
```
cloudctl case launch \
  --case $OFFLINE_DIR/$CASE_ARCHIVE \
  --inventory $CASE_INVENTORY_SETUP \
  --action install-catalog \
  --namespace $NAMESPACE \
  --args "--registry $LOCAL_REGISTRY --inputDir $OFFLINE_DIR --recursive"
```

Verify:
```
oc get catalogsource -n openshift-marketplace
```

Sample response:
```
NAME                                           DISPLAY                                      TYPE   PUBLISHER     AGE
appconnect-operator-catalogsource              IBM App Connect operator                     grpc   IBM           60s
aspera-operators                               Aspera Operators                             grpc   IBM           58s
certified-operators                            Certified Operators                          grpc   Red Hat       12d
community-operators                            Community Operators                          grpc   Red Hat       12d
couchdb-operator-catalog                       Couchdb Operator Catalog                     grpc   IBM           63s
ibm-ai-wmltraining-operator-catalog            WML Core Training                            grpc   IBM           69s
ibm-apiconnect-catalog                         IBM APIConnect catalog                       grpc   IBM           68s
ibm-automation-foundation-core-catalog         IBM Automation Foundation Core Operators     grpc   IBM           74s
ibm-cloud-databases-redis-operator-catalog     ibm-cloud-databases-redis-operator-catalog   grpc   IBM           58s
ibm-cp-integration-catalog                     IBM Cloud Pak for Integration                grpc   IBM           38s
ibm-datapower-operator-catalog                 DataPower Operator                           grpc   IBM Content   71s
ibm-eventstreams                               Event Streams Operators                      grpc   IBM           55s
ibm-integration-asset-repository-catalog       IBM CP4I Asset Repository                    grpc   IBM           46s
ibm-integration-operations-dashboard-catalog   IBM CP4I Operations Dashboard                grpc   IBM           39s
ibm-integration-platform-navigator-catalog     IBM CP4I Platform Navigator                  grpc   IBM           41s
ibmmq-operator-catalogsource                   IBM MQ                                       grpc   IBM           52s
opencloud-operators                            IBMCS Operators                              grpc   IBM           61s
redhat-marketplace                             Red Hat Marketplace                          grpc   Red Hat       12d
redhat-operators                               Red Hat Operators                            grpc   Red Hat       12d
```

Verify pods in *openshift-marketplace*, wait until all of them are up and running (it could happen that they temporary show *ImagePullBackOff* states - it usually means the nodes update described above is not fully completed):
```
watch -n 10 oc get pods -n openshift-marketplace
```

From this point on, you can continue in the same way as in the case of the online installation. Create subscriptions, an instance of the Platform Navigator, capabilities instances, etc...
