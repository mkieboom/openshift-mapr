# openshift-mapr


##  Phase 1 - Deploy the MapR Volume Driver Plugin

#### Disable SELinux temporarily:
```
echo 0 > /sys/fs/selinux/enforce
```

#### Download MapR Volume Driver Plugin files
```
# TODO: Needs update to v1.0.2 once released
mkdir ~/mapr-kdf
cd ~/mapr-kdf
wget http://archive.mapr.com/tools/KubernetesDataFabric/v1.0.1/kdf-namespace.yaml
wget http://archive.mapr.com/tools/KubernetesDataFabric/v1.0.1/kdf-provisioner.yaml
wget http://archive.mapr.com/tools/KubernetesDataFabric/v1.0.1/kdf-plugin-openshift.yaml
wget http://archive.mapr.com/tools/KubernetesDataFabric/v1.0.1/kdf-openshift-sa.yaml
wget http://archive.mapr.com/tools/KubernetesDataFabric/v1.0.1/kdf-openshift-scc.yaml
wget http://archive.mapr.com/tools/KubernetesDataFabric/v1.0.1/kdf-openshift-cr.yaml
```

#### Configure MapR Volume Driver Plugin to point to Kubernetes Service
```
vi kdf-plugin-openshift.yaml
#Set IP in KUBERNETES_SERVICE_LOCATION, eg:
- name : KUBERNETES_SERVICE_LOCATION
  value: "changeme!:6443"
```

#### Openshift deploy MapR Volume Driver Plugin
```
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

#### Remove MapR Volume Driver Plugin
```
oc delete -f kdf-provisioner.yaml
oc delete -f kdf-openshift-sa.yaml
oc delete -f kdf-openshift-scc.yaml
oc delete -f kdf-openshift-cr.yaml
oc delete -f kdf-plugin-openshift.yaml
oc delete -f kdf-namespace.yaml
```


