# openshift-mapr - MapR PACC on OpenShift

##  Introduction
Below example shows deploying the MapR PACC docker container on Red Hat Openshift leveraging the MapR Volume Driver Plugin to persist the storage of the container. For demo purposes only.

It consists of three pases as we are using static provisioning (dynamic provisioning is also supported):

[Phase 1: create a volume on the MapR cluster for static provisioning.](#phase-1---create-a-volume-on-the-mapr-cluster-for-static-provisioning)

[Phase 2: create a custom MapR PACC container.](#phase-2---create-a-custom-mapr-pacc-container)

[Phase 3: deploy the custom MapR PACC container on Openshift.](#phase-3---Deploy-the-custom-mapr-pacc-container-on-openshift)

Always check the latest MapR documentation on:  
https://mapr.com/docs/home/PersistentStorage/kdf_plan_and_install.html

## Prerequisites
Red Hat Openshift version 3.9.40 or later  
MapR Converged Data Platform v6.0.1 or later

##  Phase 1 - create a volume on the MapR cluster for static provisioning
As we will be using static provisioning (dynamic is also supported), we will create a volume on the MapR cluster first:
```
maprcli volume create -name mapr-pacc-volume -path /mapr-pacc-volume
```

##  Phase 2 - Create a custom MapR PACC container

Download and run the MapR PACC setup.sh script:
```
wget http://package.mapr.com/releases/installer/mapr-setup.sh

sudo bash mapr-setup.sh docker client
```

IMPORTANT: make sure to answer 'No' to the question on adding the POSIX (FUSE) client. We will not be leveraging a FUSE client in the PACC container (as that would require privileged access for the Docker container), but instead we will use a K8S PVC.
```
# Do NOT install the FUSE client:
Add POSIX (FUSE) client to container? (y/n) [y]: n

# Do install the Hadoop and Spark client
Install Hadoop YARN client (y/n) [n]: y
Add Spark client to container? (y/n) [n]: y
```

Complete example of creating a PACC:
```
Build MapR client image? (y/n) [y]: y
Image OS class (centos7, ubuntu16) [centos7]:
Docker FROM base image name:tag [centos:centos7]:
MapR core version [6.1.0]: 6.0.1
MapR MEP version [6.0.0]: 5.0.0
Install Hadoop YARN client (y/n) [n]: y
Add POSIX (FUSE) client to container? (y/n) [y]: n
Add HBase client to container? (y/n) [n]: n
Add Hive client to container? (y/n) [n]: n
Add Pig client to container? (y/n) [n]: n
Add Spark client to container? (y/n) [n]: y
Add Streams clients to container? (y/n) [y]: n
MapR client image tag name [maprtech/pacc:6.1.0_6.0.0_centos7_yarn_spark]:
```

Once the PACC image has been created, either push it to a Docker registry or make sure to use a node selector to run the pod on Openshift on the node where the custom PACC has been created.

##  Phase 3 - Deploy the custom MapR PACC container on Openshift

#### Configure the MapR ticket secret

Set the mapr-ticket-secret to allow the pod to authenticate with the MapR cluster:
```
vi mapr-k8s-pacc-secure-static.yaml

# Set the mapr-ticket-secret

# To create a Ticket, login onto the MapR cluster and execute following:
# 1. maprlogin password -user mapr
# 2. echo -n $(cat /tmp/maprticket_####) | base64 -w 0
# 3. Copy the base64 encoded ticket into the CONTAINER_TICKET line, eg:
  CONTAINER_TICKET: ZGVtby5tYXByLmNvbSBxSkxrVEhoeGtFRlUxU2p3a29NcUN4ZVhra1hPS2JwTVphNllTQ3FpaENnYlRhVkQyOEUrTTJhSng4dWljdlp1aHozR1pOS2pCNW8wRmFjRlVWRGVvVEZYVzhXdElTUG5DOEp2Q01zZG1PcEFIZ2V6eWdrekU5V1ZwaGVoT2RMcWFyaVdGVmtZSjEwVngzNG85RFFzM0U5YmdFWFZ0bVJNQ2JiREd6THpJbzVvVDBpTkU5OUlhT2dySnN3RE9SYmd6bFRBRjBzVVlHK05iL09mUkVWNUV1SFpKZk13M3NxMUY3MjI1bjJHN3hBZkhCQXFGb0dDSGhoNnhvVm45MmNEZHZJTGk4anVkU1ZMSzd0SFpFZzRZUFJXazdZUU0rdz0=

```

#### MapR Cluster details
Set the MapR cluster details to reflect your MapR cluster deployment at two places:

1. Configure the MapR cluster details for the PVC:
```
vi mapr-k8s-pacc-secure-static.yaml

      cluster: "demo.mapr.com"
      cldbHosts: "172.16.4.233 172.16.4.234 172.16.4.235"
      volumePath: "/mapr-pacc-volume"
      securityType: "secure"
      ticketSecretName: "mapr-ticket-secret"
      ticketSecretNamespace: "mapr-apps-pacc"
```

2. Configure the MapR cluster details for the Hadoop, Spark Hive etc client(s) inside the PACC container:
```
vi mapr-k8s-pacc-secure-static.yaml

    env:
      - name: MAPR_CLUSTER
        value: "demo.mapr.com"
      - name: MAPR_CLDB_HOSTS
        value: "172.16.4.233,172.16.4.234,172.16.4.235"
      - name: MAPR_CONTAINER_USER
        value: "mapr"
      - name: MAPR_CONTAINER_GROUP
        value: "mapr"
      - name: MAPR_CONTAINER_UID
        value: "5000"
      - name: MAPR_CONTAINER_GID
        value: "5000"
      - name: MAPR_TICKETFILE_LOCATION
        value: "/tmp/maprticket/CONTAINER_TICKET"
```

#### Configure the Container Image name
Depending on the PACC customization in Phase 2, you might need to change the container image name:
```
    image: maprtech/pacc:6.0.1_5.0.0_centos7_yarn_spark
```

If not using a Docker registry for the Docker image, please make sure to configure a node selector as well to run the pod on the node where the PACC has been created.

#### Launch the busybox pod to test the mapr-apps-scc
```
oc create -f mapr-k8s-pacc-secure-static.yaml
```

Continue with [connecting to the pod](#connect-to-the-pod-and-create-data-on-the-mapr-filesystem) and write data to the MapR cluster from inside the pod.


#### Connect to the pod and create data on the MapR Filesystem
The below steps will leverage the K8S PVC (and not the PACC FUSE client) to connect to the MapR cluster.
```

# From inside the container:

# Connect to the container
oc exec -it mapr-apps-pacc bash -n mapr-apps-pacc

# Create a folder and file and validate the uid/gid
# The uid/gid should match the uid/gid in the MapR ticket
$ touch /mapr/hello_from_pod
$ 
$ ls -al /mapr/
total 1
drwxr-xr-x. 2 mapr mapr   2 Nov 27 11:14 .
drwxr-xr-x. 1 root root 124 Nov 27 11:45 ..
-rw-r--r--. 1 mapr mapr   0 Nov 27 10:43 hello-from-cluster
-rw-r--r--. 1 mapr mapr   0 Nov 27 10:55 hello_from_pod

# On the MapR Cluster:
$ ls -al /mapr/demo.mapr.com/mapr-pacc-volume/
total 1
-rw-r--r--.  1 mapr mapr  0 Nov 27 10:43 hello-from-cluster
-rw-r--r--.  1 mapr mapr  0 Nov 27 10:55 hello_from_pod
```

As a final step we can also test submitting jobs to the cluster which will leverage the PACC functionality.
```
# Test if we can connect to the cluster leveraging the Hadoop client (installed in the PACC as part of phase 2)
hadoop fs -ls /

# Submit a YARN job to the cluster:
yarn jar /opt/mapr/hadoop/hadoop-2.7.0/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.0-mapr-*.jar pi 1 1
```

#### Optional: cleanup and remove pod
```
# Remove the pod
oc delete -f mapr-k8s-pacc-secure-static.yaml
```
