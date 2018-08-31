# openshift-mapr

##  Introduction
Below example shows deploying a docker container on Openshift leveraging the MapR Volume Driver Plugin to persist the storage of the container. For demo purposes only.

Always check the latest MapR documentation on:
https://mapr.com/docs/home/PersistentStorage/kdf_plan_and_install.html

##  Phase 1 - Deploy the MapR Volume Driver Plugin


#### Download MapR Volume Driver Plugin files
```
mkdir ~/mapr-kdf
cd ~/mapr-kdf
wget http://archive.mapr.com/tools/KubernetesDataFabric/v1.0.2/kdf-namespace.yaml
wget http://archive.mapr.com/tools/KubernetesDataFabric/v1.0.2/kdf-provisioner.yaml
wget http://archive.mapr.com/tools/KubernetesDataFabric/v1.0.2/kdf-plugin-openshift.yaml
wget http://archive.mapr.com/tools/KubernetesDataFabric/v1.0.2/kdf-openshift-sa.yaml
wget http://archive.mapr.com/tools/KubernetesDataFabric/v1.0.2/kdf-openshift-scc.yaml
wget http://archive.mapr.com/tools/KubernetesDataFabric/v1.0.2/kdf-openshift-cr.yaml
```

#### Configure MapR Volume Driver Plugin to point to Kubernetes Service
```
# Set IP in KUBERNETES_SERVICE_LOCATION, eg:
# - name : KUBERNETES_SERVICE_LOCATION
#   value: "172.16.4.183:6443"

vi kdf-plugin-openshift.yaml

- name : KUBERNETES_SERVICE_LOCATION
  value: "changeme!:6443"
```

#### Set the FLEXVOLUME_PLUGIN_PATH and hostPath applicable to your Openshift install

When running Openshift installed using RPM's:
```
vi kdf-plugin-openshift.yaml

# Set the following:
  - name : FLEXVOLUME_PLUGIN_PATH
    value: "/usr/libexec/kubernetes/kubelet-plugins/volume/exec"

  - name: plugindir
    hostPath:
      path: /usr/libexec/kubernetes/kubelet-plugins/volume/exec/
```

When running Openshift installed Containerized:
````
vi kdf-plugin-openshift.yaml

# Set the following:
  - name : FLEXVOLUME_PLUGIN_PATH
    value: "/etc/origin/kubelet-plugins/volume/exec/"

  - name: plugindir
    hostPath:
      path: /etc/origin/kubelet-plugins/volume/exec/
````

#### Enable support for enforced SELinux

```
# Add allowPrivilegedContainer: true

vi kdf-openshift-scc.yaml

  allowPrivilegedContainer: true
```

```
# Add "securityContext: privileged: true" to the containers section, eg:

vi kdf-plugin-openshift.yaml

      containers:
        - name: mapr-kdfplugin
          securityContext:
            privileged: true 
```

```
# Add "securityContext: privileged: true" to the containers section, eg:

vi kdf-provisioner.yaml

      containers:
      - name: mapr-kdfprovisioner
        securityContext:
          privileged: true 
```

#### Openshift deploy MapR Volume Driver Plugin
```
oc login -u admin -p admin https://ip-172-16-4-183.eu-west-1.compute.internal:8443/

oc create -f kdf-namespace.yaml
oc create -f kdf-openshift-sa.yaml
oc create -f kdf-openshift-scc.yaml
oc adm policy add-scc-to-user maprkdf-scc system:serviceaccount:mapr-system:maprkdf
oc create -f kdf-openshift-cr.yaml
oc adm policy add-cluster-role-to-user mapr:kdf system:serviceaccount:mapr-system:maprkdf
oc create -f kdf-plugin-openshift.yaml
oc create -f kdf-provisioner.yaml
```

#### Check status of pods
```
oc get pods --all-namespaces
```

#### OPTIONAL: Remove MapR Volume Driver Plugin
```
oc delete -f kdf-provisioner.yaml
oc delete -f kdf-openshift-sa.yaml
oc delete -f kdf-openshift-scc.yaml
oc delete -f kdf-openshift-cr.yaml
oc delete -f kdf-plugin-openshift.yaml
oc delete -f kdf-namespace.yaml
```


##  Phase 2 - Launch containers on Openshift leveraging the MapR Volume Driver Plugin


####  Clone the files from the example busybox container
```
git clone https://github.com/mkieboom/openshift-mapr
cd openshift-mapr
```

#### Create mapr-apps-scc
```
oc create -f mapr-apps-scc.yaml

# List the scc's and validate that the mapr-apps-scc exists:
oc get scc
```

#### Add mapr-apps-scc scc to serviceaccount mapr-apps:mapr-apps-sa
```
oc adm policy add-scc-to-user mapr-apps-scc system:serviceaccount:mapr-apps:mapr-apps-sa

# Validate that the user got added to the ssc
oc edit scc mapr-apps-scc
```

#### Configure the ticket secret
Below steps are mandatory to allow the pod to authenticate with the MapR platform 

Set the mapr-provisioner-secrets to allow dynamic provisioning:
```
vi mapr-k8s-busybox-secure-dynamic-part1-volumedriver.yaml

# Set the mapr-provisioner-secrets for dynamic provisioning of MapR Volumes
# base64 encoding: "echo -n '<mapr username/mapr password>' | base64" eg:
# echo -n 'mapr' | base64
  MAPR_CLUSTER_USER: "bWFwcg=="
  MAPR_CLUSTER_PASSWORD: "bWFwcg=="
```

Set the mapr-ticket-secret to allow the pod to authenticate with the MapR cluster:
```
vi mapr-k8s-busybox-secure-dynamic-part1-volumedriver.yaml

# Set the mapr-ticket-secret

# To create a Ticket, login onto the MapR cluster and execute following:
# 1. maprlogin password -user mapr
# 2. echo -n $(cat /tmp/maprticket_####) | base64
# 3. combine 6 lines base64 output in single CONTAINER_TICKET line, eg:
  CONTAINER_TICKET: ZGVtby5tYXByLmNvbSBxSkxrVEhoeGtFRlUxU2p3a29NcUN4ZVhra1hPS2JwTVphNllTQ3FpaENnYlRhVkQyOEUrTTJhSng4dWljdlp1aHozR1pOS2pCNW8wRmFjRlVWRGVvVEZYVzhXdElTUG5DOEp2Q01zZG1PcEFIZ2V6eWdrekU5V1ZwaGVoT2RMcWFyaVdGVmtZSjEwVngzNG85RFFzM0U5YmdFWFZ0bVJNQ2JiREd6THpJbzVvVDBpTkU5OUlhT2dySnN3RE9SYmd6bFRBRjBzVVlHK05iL09mUkVWNUV1SFpKZk13M3NxMUY3MjI1bjJHN3hBZkhCQXFGb0dDSGhoNnhvVm45MmNEZHZJTGk4anVkU1ZMSzd0SFpFZzRZUFJXazdZUU0rdz0=

```

Set the MapR cluster details to reflect your MapR cluster deployment:
```
vi mapr-k8s-busybox-secure-dynamic-part1-volumedriver.yaml

  restServers: "172.18.14.32:8443"
  cldbHosts: "172.18.14.32"
  cluster: "demo.mapr.com"
  securityType: "secure"
  ticketSecretName: "mapr-ticket-secret"
  ticketSecretNamespace: "mapr-apps"
  maprSecretName: "mapr-provisioner-secrets"
  maprSecretNamespace: "mapr-apps"
  namePrefix: "busybox"
  mountPrefix: "/busybox"
  reclaimPolicy: "Retain"
  advisoryquota: "100M"
  type: "rw"
  mount: "1"
```

#### Launch the busybox pod to test the mapr-apps-scc
```
oc create -f mapr-k8s-busybox-secure-dynamic-part1-volumedriver.yaml
oc create -f mapr-k8s-busybox-secure-dynamic-part2-container.yaml
```

#### Connect to the pod and create data on the MapR Filesystem
```
# Connect to the container
oc exec -it mapr-k8s-busybox -n mapr-apps -- sh

# Check the uid/gid in the container and create files/folder on MapR
# The uid/gid should reflect the uid/gid from the mapr user on the MapR cluster (5000/5000)
id
ls -al /mapr
touch /mapr/123
mkdir /mapr/mydir
ls -al /mapr
```

#### Optional: cleanup and remove pod and scc
```
# Remove the pod
oc delete -f mapr-k8s-busybox-secure-dynamic-part2-container.yaml
oc delete -f mapr-k8s-busybox-secure-dynamic-part1-volumedriver.yaml

# Cleanup
oc adm policy remove-scc-from-user mapr-apps-scc system:serviceaccount:mapr-apps:mapr-apps-sa
oc delete -f mapr-apps-scc.yaml
```
