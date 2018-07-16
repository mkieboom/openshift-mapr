# openshift-mapr


##  Phase 1 - Deploy the MapR Volume Driver Plugin

#### Disable SELinux temporarily:
```
echo 0 > /sys/fs/selinux/enforce
```

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
#   value: "172.16.4.225:6443"

vi kdf-plugin-openshift.yaml

- name : KUBERNETES_SERVICE_LOCATION
  value: "changeme!:6443"
```

#### When MapR Plugin v1.0.2 is not yet publically available
```
# Check if v1.0.2 is available on docker hub. If so, skip this step.
# https://hub.docker.com/r/maprtech/kdf-plugin/tags/

vi kdf-plugin-openshift.yaml

  containers:
    - name: mapr-kdfplugin
      imagePullPolicy: Never
      image: docker.artifactory/maprtech/kdf-plugin:1.0.2_001_centos7
```

#### Openshift deploy MapR Volume Driver Plugin
```
oc login -u admin -p admin https://ip-172-16-4-225.eu-west-1.compute.internal:8443/

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

#### Enable SELinux:
```
echo 1 > /sys/fs/selinux/enforce
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
git clone https://github.com/mkieboom/mapr-k8s-busybox
cd mapr-k8s-busybox
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

#### Launch the busybox pod to test the mapr-apps-scc
```
kubectl create -f mapr-k8s-busybox-secure-dynamic-part1-volumedriver.yaml
kubectl create -f mapr-k8s-busybox-secure-dynamic-part2-container.yaml
```

#### Connect to the pod and create data on the MapR Filesystem
```
# Connect to the container
kubectl exec -it mapr-k8s-busybox -n mapr-apps -- sh

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
kubectl delete -f mapr-k8s-busybox-secure-dynamic-part2-container.yaml
kubectl delete -f mapr-k8s-busybox-secure-dynamic-part1-volumedriver.yaml

# Cleanup
oc adm policy remove-scc-from-user mapr-apps-scc system:serviceaccount:mapr-apps:mapr-apps-sa
oc delete -f mapr-apps-scc.yaml
```
