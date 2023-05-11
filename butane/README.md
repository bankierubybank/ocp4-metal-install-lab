### Installing Butane
```
curl https://mirror.openshift.com/pub/openshift-v4/clients/butane/latest/butane --output butane

chmod +x butane

echo $PATH

cp butane /usr/bin/

butane --version
```

### Creating a MachineConfig object by using Butane
```
butane 99-worker-chrony.bu -o 99-worker-chrony.yaml

butane 99-master-chrony.bu -o 99-master-chrony.yaml
```

### Configuring chrony time service
```
oc apply -f ./99-worker-chrony.yaml

oc apply -f ./99-master-chrony.yaml
```

Ref: https://docs.openshift.com/container-platform/4.10/installing/install_config/installing-customizing.html?extIdCarryOver=true&intcmp=701f20000012k6EAAQ&sc_cid=701f2000001OH7JAAW#installation-special-config-chrony_installing-customizing
