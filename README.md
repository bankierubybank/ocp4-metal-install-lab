# OpenShift 4 Bare Metal Install - User Provisioned Infrastructure (UPI)

## Architecture Diagram

###### Information
- Cluster Name: UAT
- Base Domain: ocp.nsth.demo

![Architecture Diagram](./diagram/uat.ocp.nsth.demo.png)

## Download Software

1. Download [CentOS 8 Stream](https://mirrors.bangmod.cloud/centos/8-stream/isos/x86_64/)
   Other Mirror [CentOS Mirror](https://www.centos.org/download/mirrors/)
1. Login to [RedHat OpenShift Cluster Manager](https://cloud.redhat.com/openshift)
1. Select 'Create Cluster' from the 'Clusters' navigation menu
1. Select 'RedHat OpenShift Container Platform'
1. Select 'Run on Bare Metal'
1. Download the following files:

   - Openshift Installer for Linux
   - Pull secret
   - Command Line Interface for Linux and your workstations OS (OC)
   - Red Hat Enterprise Linux CoreOS (RHCOS)
     - rhcos-X.X.X-x86_64-metal.x86_64.raw.gz
     - rhcos-X.X.X-x86_64-installer.x86_64.iso (or rhcos-X.X.X-x86_64-live.x86_64.iso for newer versions)

1. Or from here [RHOCP Installer & Client](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/) [RHCOS](https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/)

###### Filename format (Replace x.xx.x to version you want to install)
   - OpenShift Installer: openshift-install-linux-x.xx.x.tar.gz
   - OpenShift Client: openshift-client-linux-x.xx.x.tar.gz
   - RHCOS: rhcos-x.xx.x-x86_64-metal.x86_64.raw.gz

## Prepare the 'Bare Metal' environment

1. Copy the CentOS 8 Stream iso to an ESXi datastore
1. Create a new Port Group called 'OCP' under Networking

### OCP VMs
| VM | CPU | Memory | Disk | Image | NICs |
| ------ | ------ | ------ | ------ | ------ | ------ |
| bastion | 4 | 8GB | 200GB | centos_8.iso | 2 NIC - Attach to Internal network and OCP Port Group |
| master-# | 4 | 12GB | 200GB | rhcos-X.X.X-x86_64-installer.x86_64.iso | 1 NIC - Attach to OCP Port Group |
| worker-# | 4 | 8GB | 200GB | rhcos-X.X.X-x86_64-installer.x86_64.iso | 1 NIC - Attach to OCP Port Group |
| bootstrap | 8 | 16GB | 200GB | rhcos-X.X.X-x86_64-installer.x86_64.iso | 1 NIC - Attach to OCP Port Group |

## Configure Environmental Services

1. Install CentOS8 on the ocp-svc host

   - Remove the home dir partition and assign all free storage to '/'
   - Optionally you can install the 'Guest Tools' package to have monitoring and reporting in the VMware ESXi dashboard
   - Enable the nsth.demo NIC only to obtain a DHCP address from the nsth.demo network and make note of the IP address (ocp-svc_IP_address) assigned to the vm

1. Boot the ocp-svc VM

1. Move the files downloaded from the RedHat Cluster Manager site to the ocp-svc node

   ```bash
   scp ~/Downloads/openshift-install-linux.tar.gz ~/Downloads/openshift-client-linux.tar.gz ~/Downloads/rhcos-metal.x86_64.raw.gz root@{ocp-svc_IP_address}:/root/
   ```

1. SSH to the ocp-svc vm

   ```bash
   ssh root@{ocp-svc_IP_address}
   ```

1. Extract Client tools and copy them to `/usr/local/bin`

   ```bash
   tar xvf openshift-client-linux.tar.gz
   mv oc kubectl /usr/local/bin
   ```

1. Confirm Client Tools are working

   ```bash
   kubectl version
   oc version
   ```

1. Update CentOS and install required dependencies

   ```bash
   dnf update
   dnf install -y git bind bind-utils dhcp-server httpd haproxy nfs-utils
   ```

1. Extract the OpenShift Installer

   ```bash
   tar xvf openshift-install-linux.tar.gz
   ```

1. Download [config files](https://github.com/bankierubybank/ocp4-metal-install-lab) for each of the services

   ```bash
   git clone https://github.com/bankierubybank/ocp4-metal-install-lab
   ```

1. OPTIONAL: Create a file '~/.vimrc' and paste the following (this helps with editing in vim, particularly yaml files):

   ```bash
   cat <<EOT >> ~/.vimrc
   syntax on
   set nu et ai sts=0 ts=2 sw=2 list hls
   EOT
   ```

   Update the preferred editor

   ```bash
   export OC_EDITOR="vim"
   export KUBE_EDITOR="vim"
   ```

1. Set a Static IP for OCP network interface `nmtui-edit ens224` or edit `/etc/sysconfig/network-scripts/ifcfg-ens444`

   - **Address**: 192.168.44.1
   - **DNS Server**: 127.0.0.1
   - **Search domain**: ocp.nsth.demo
   - Never use this network for default route
   - Automatically connect

   > If changes arent applied automatically you can bounce the NIC with `nmcli connection down ens224` and `nmcli connection up ens224`

1. Setup firewalld

   Create **internal** and **external** zones

   ```bash
   firewall-cmd --zone=external --change-interface=ens192
   firewall-cmd --zone=internal --change-interface=ens224
   ```

   Ref: [firewall-cmd](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/security_guide/sec-working_with_zones)

   View zones:

   ```bash
   firewall-cmd --get-active-zones
   ```

   Set masquerading (source-nat) on the both zones.

   So to give a quick example of source-nat - for packets leaving the external interface, which in this case is ens192 - after they have been routed they will have their source address altered to the interface address of ens192 so that return packets can find their way back to this interface where the reverse will happen.

   ```bash
   firewall-cmd --zone=external --add-masquerade --permanent
   firewall-cmd --zone=internal --add-masquerade --permanent
   ```

   Reload firewall config

   ```bash
   firewall-cmd --reload
   ```

   Check the current settings of each zone

   ```bash
   firewall-cmd --list-all --zone=internal
   firewall-cmd --list-all --zone=external
   ```

   When masquerading is enabled so is ip forwarding which basically makes this host a router. Check:

   ```bash
   cat /proc/sys/net/ipv4/ip_forward
   ```

1. Configure BIND DNS

   Apply configuration

   ```bash
   cp ~/ocp4-metal-install-lab/dns/named.conf /etc/named.conf
   cp -R ~/ocp4-metal-install-lab/dns/zones /etc/named/
   ```

   Configure the firewall for DNS

   ```bash
   firewall-cmd --add-port=53/udp --zone=internal --permanent
   # for OCP 4.9 and later 53/tcp is required
   firewall-cmd --add-port=53/tcp --zone=internal --permanent
   firewall-cmd --reload
   ```

   Enable and start the service

   ```bash
   systemctl enable named
   systemctl start named
   systemctl status named
   ```

   Change the nsth.demo nic (ens192) to use 127.0.0.1 for DNS AND ensure `Ignore automatically Obtained DNS parameters` is ticked

   ```bash
   nmtui-edit ens192
   ```

   Restart Network Manager

   ```bash
   systemctl restart NetworkManager
   ```

   Confirm dig now sees the correct DNS results by using the DNS Server running locally

   ```bash
   dig ocp.nsth.demo
   # The following should return the answer ocp-bootstrap.uat.ocp.nsth.demo from the local server
   dig -x 192.168.44.2
   ```

1. Configure DHCP

   ```bash
   dnf install dhcp-server -y
   ```

   Edit dhcpd.conf from the cloned git repo to have the correct mac address for each host and copy the conf file to the correct location for the DHCP service to use

   ```bash
   cp ~/ocp4-metal-install-lab/dhcpd.conf /etc/dhcp/dhcpd.conf
   ```

   Configure the Firewall

   ```bash
   firewall-cmd --add-service=dhcp --zone=internal --permanent
   firewall-cmd --reload
   ```

   Enable and start the service

   ```bash
   systemctl enable dhcpd
   systemctl start dhcpd
   systemctl status dhcpd
   ```

1. Configure Apache Web Server

   Change default listen port to 8080 in httpd.conf

   ```bash
   sed -i 's/Listen 80/Listen 0.0.0.0:8080/' /etc/httpd/conf/httpd.conf
   ```

   Configure the firewall for Web Server traffic

   ```bash
   firewall-cmd --add-port=8080/tcp --zone=internal --permanent
   firewall-cmd --reload
   ```

   Enable and start the service

   ```bash
   systemctl enable httpd
   systemctl start httpd
   systemctl status httpd
   ```

   Making a GET request to localhost on port 8080 should now return the default Apache webpage

   ```bash
   curl localhost:8080
   ```

1. Configure HAProxy

   Copy HAProxy config

   ```bash
   cp ~/ocp4-metal-install-lab/haproxy.cfg /etc/haproxy/haproxy.cfg
   ```

   Configure the Firewall

   > Note: Opening port 9000 in the external zone allows access to HAProxy stats that are useful for monitoring and troubleshooting. The UI can be accessed at: `http://{ocp-svc_IP_address}:9000/stats`

   ```bash
   firewall-cmd --add-port=6443/tcp --zone=internal --permanent # kube-api-server on control pane nodes
   firewall-cmd --add-port=6443/tcp --zone=external --permanent # kube-api-server on control pane nodes
   firewall-cmd --add-port=22623/tcp --zone=internal --permanent # machine-config server
   firewall-cmd --add-service=http --zone=internal --permanent # web services hosted on worker nodes
   firewall-cmd --add-service=http --zone=external --permanent # web services hosted on worker nodes
   firewall-cmd --add-service=https --zone=internal --permanent # web services hosted on worker nodes
   firewall-cmd --add-service=https --zone=external --permanent # web services hosted on worker nodes
   firewall-cmd --add-port=9000/tcp --zone=external --permanent # HAProxy Stats
   firewall-cmd --reload
   ```

   Enable and start the service

   ```bash
   setsebool -P haproxy_connect_any 1 # SELinux name_bind access
   systemctl enable haproxy
   systemctl start haproxy
   systemctl status haproxy
   ```

1. Configure NFS for the OpenShift Registry. It is a requirement to provide storage for the Registry, emptyDir can be specified if necessary.

   Create the Share

   Check available disk space and its location `df -h`

   ```bash
   mkdir -p /shares/registry
   chown -R nobody:nobody /shares/registry
   chmod -R 777 /shares/registry
   ```

   Export the Share

   ```bash
   echo "/shares/registry 192.168.44.0/24(rw,sync,root_squash,no_subtree_check,no_wdelay)" > /etc/exports
   exportfs -rv
   ```

   Set Firewall rules:

   ```bash
   firewall-cmd --zone=internal --add-service mountd --permanent
   firewall-cmd --zone=internal --add-service rpc-bind --permanent
   firewall-cmd --zone=internal --add-service nfs --permanent
   firewall-cmd --reload
   ```

   Enable and start the NFS related services

   ```bash
   systemctl enable nfs-server rpcbind
   systemctl start nfs-server rpcbind nfs-mountd
   ```

## Generate and host install files

1. Generate an SSH key pair keeping all default options

   ```bash
   ssh-keygen
   ```

1. Create an install directory

   ```bash
   mkdir ~/ocp-install
   ```

1. Copy the install-config.yaml included in the clones repository to the install directory

   ```bash
   cp ~/ocp4-metal-install-lab/install-config.yaml ~/ocp-install
   ```

1. Update the install-config.yaml with your own pull-secret and ssh key.

   - Line 23 should contain the contents of your pull-secret.txt
   - Line 24 should contain the contents of your '~/.ssh/id_rsa.pub'

   ```bash
   vim ~/ocp-install/install-config.yaml
   ```

1. Generate Kubernetes manifest files

   ```bash
   ~/openshift-install create manifests --dir ~/ocp-install
   ```

   > A warning is shown about making the Control Pane nodes schedulable. It is up to you if you want to run workloads on the Control Pane nodes. If you dont want to you can disable this with:
   > `sed -i 's/mastersSchedulable: true/mastersSchedulable: false/' ~/ocp-install/manifests/cluster-scheduler-02-config.yml`.
   > Make any other custom changes you like to the core Kubernetes manifest files.

   Generate the Ignition config and Kubernetes auth files

   ```bash
   ~/openshift-install create ignition-configs --dir ~/ocp-install/
   ```

1. Create a hosting directory to serve the configuration files for the OpenShift booting process

   ```bash
   mkdir /var/www/html/ocp4
   ```

1. Copy all generated install files to the new web server directory

   ```bash
   cp -R ~/ocp-install/* /var/www/html/ocp4
   ```

1. Move the Core OS image to the web server directory (later you need to type this path multiple times so it is a good idea to shorten the name)

   ```bash
   mv ~/rhcos-X.X.X-x86_64-metal.x86_64.raw.gz /var/www/html/ocp4/rhcos
   ```

1. Change ownership and permissions of the web server directory

   ```bash
   chcon -R -t httpd_sys_content_t /var/www/html/ocp4/
   chown -R apache: /var/www/html/ocp4/
   chmod 755 /var/www/html/ocp4/
   ```

1. Confirm you can see all files added to the `/var/www/html/ocp4/` dir through Apache

   ```bash
   curl localhost:8080/ocp4/
   ```

## Deploy OpenShift

1. Power on the ocp-bootstrap host and ocp-cp-\# hosts and select 'Tab' to enter boot configuration. Enter the following configuration:

Ref: https://docs.openshift.com/container-platform/4.11/installing/installing_bare_metal/installing-bare-metal.html#installation-user-infra-machines-advanced_installing-bare-metal at Table 13. coreos.inst boot options

   ```bash
   # Bootstrap Node - ocp-bootstrap
   coreos.inst.install_dev=sda coreos.inst.image_url=http://192.168.44.1:8080/ocp4/rhcos coreos.inst.insecure=yes coreos.inst.ignition_url=http://192.168.44.1:8080/ocp4/bootstrap.ign
   ```

   ```bash
   # Each of the Control Pane Nodes - ocp-cp-\#
   coreos.inst.install_dev=sda coreos.inst.image_url=http://192.168.44.1:8080/ocp4/rhcos coreos.inst.insecure=yes coreos.inst.ignition_url=http://192.168.44.1:8080/ocp4/master.ign
   ```

1. Power on the ocp-w-\# hosts and select 'Tab' to enter boot configuration. Enter the following configuration:

   ```bash
   # Each of the Worker Nodes - ocp-w-\#
   coreos.inst.install_dev=sda coreos.inst.image_url=http://192.168.44.1:8080/ocp4/rhcos coreos.inst.insecure=yes coreos.inst.ignition_url=http://192.168.44.1:8080/ocp4/worker.ign
   ```

## Monitor the Bootstrap Process

1. You can monitor the bootstrap process from the ocp-svc host at different log levels (debug, error, info)

   ```bash
   ~/openshift-install --dir ~/ocp-install wait-for bootstrap-complete --log-level=debug
   ```

1. Once bootstrapping is complete the ocp-boostrap node [can be removed](#remove-the-bootstrap-node)

## Remove the Bootstrap Node

1. Remove all references to the `ocp-bootstrap` host from the `/etc/haproxy/haproxy.cfg` file

   ```bash
   # Two entries
   vim /etc/haproxy/haproxy.cfg
   # Restart HAProxy - If you are still watching HAProxy stats console you will see that the ocp-boostrap host has been removed from the backends.
   systemctl reload haproxy
   ```

1. The ocp-bootstrap host can now be safely shutdown and deleted from the VMware ESXi Console, the host is no longer required

## Wait for installation to complete

> IMPORTANT: if you set mastersSchedulable to false the [worker nodes will need to be joined to the cluster](#join-worker-nodes) to complete the installation. This is because the OpenShift Router will need to be scheduled on the worker nodes and it is a dependency for cluster operators such as ingress, console and authentication.

1. Collect the OpenShift Console address and kubeadmin credentials from the output of the install-complete event

   ```bash
   ~/openshift-install --dir ~/ocp-install wait-for install-complete
   ```

1. Continue to join the worker nodes to the cluster in a new tab whilst waiting for the above command to complete

## Join Worker Nodes

1. Setup 'oc' and 'kubectl' clients on the ocp-svc machine

   ```bash
   export KUBECONFIG=~/ocp-install/auth/kubeconfig
   # Test auth by viewing cluster nodes
   oc get nodes
   ```

1. View and approve pending CSRs

   > Note: Once you approve the first set of CSRs additional 'kubelet-serving' CSRs will be created. These must be approved too.
   > If you do not see pending requests wait until you do.

   ```bash
   # View CSRs
   oc get csr
   # Approve all pending CSRs
   oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve
   # Wait for kubelet-serving CSRs and approve them too with the same command
   oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve
   ```

1. Watch and wait for the Worker Nodes to join the cluster and enter a 'Ready' status

   > This can take 5-10 minutes

   ```bash
   watch -n5 oc get nodes
   ```

## Configure storage for the Image Registry

> A Bare Metal cluster does not by default provide storage so the Image Registry Operator bootstraps itself as 'Removed' so the installer can complete. As the installation has now completed storage can be added for the Registry and the operator updated to a 'Managed' state.

1. Create the 'image-registry-storage' PVC by updating the Image Registry operator config by updating the management state to 'Managed' and adding 'pvc' and 'claim' keys in the storage key:

   ```bash
   oc edit configs.imageregistry.operator.openshift.io
   ```

   ```yaml
   managementState: Managed
   ```

   ```yaml
   storage:
     pvc:
       claim: # leave the claim blank
   ```

1. Confirm the 'image-registry-storage' pvc has been created and is currently in a 'Pending' state

   ```bash
   oc get pvc -n openshift-image-registry
   ```

1. Create the persistent volume for the 'image-registry-storage' pvc to bind to

   ```bash
   oc create -f ~/ocp4-metal-install-lab/manifest/registry-pv.yaml
   ```

1. After a short wait the 'image-registry-storage' pvc should now be bound

   ```bash
   oc get pvc -n openshift-image-registry
   ```

## Create the first Admin user

1. Apply the `oauth-htpasswd.yaml` file to the cluster

   > This will create a user 'admin' with the password 'password'. To set a different username and password substitue the htpasswd key in the '~/ocp4-metal-install-lab/manifest/oauth-htpasswd.yaml' file with the output of `htpasswd -n -B -b <username> <password>`

   ```bash
   oc apply -f ~/ocp4-metal-install-lab/manifest/oauth-htpasswd.yaml
   ```

1. Assign the new user (admin) admin permissions

   ```bash
   oc adm policy add-cluster-role-to-user cluster-admin admin
   ```

## Access the OpenShift Console

1. Wait for the 'console' Cluster Operator to become available

   ```bash
   oc get co
   ```

1. Append the following to your local workstations `/etc/hosts` file:

   > From your local workstation
   > If you do not want to add an entry for each new service made available on OpenShift you can configure the ocp-svc DNS server to serve externally and create a wildcard entry for \*.apps.uat.ocp.nsth.demo

   ```bash
   # Open the hosts file
   sudo vi /etc/hosts

   # Append the following entries:
   172.16.68.201 bastion api.uat.ocp.nsth.demo console-openshift-console.apps.uat.ocp.nsth.demo oauth-openshift.apps.uat.ocp.nsth.demo downloads-openshift-console.apps.uat.ocp.nsth.demo alertmanager-main-openshift-monitoring.apps.uat.ocp.nsth.demo grafana-openshift-monitoring.apps.uat.ocp.nsth.demo prometheus-k8s-openshift-monitoring.apps.uat.ocp.nsth.demo thanos-querier-openshift-monitoring.apps.uat.ocp.nsth.demo
   ```

1. Navigate to the [OpenShift Console URL](https://console-openshift-console.apps.uat.ocp.nsth.demo) and log in as the 'admin' user

   > You will get self signed certificate warnings that you can ignore
   > If you need to login as kubeadmin and need to the password again you can retrieve it with: `cat ~/ocp-install/auth/kubeadmin-password`

## Troubleshooting

1. DHCP troubleshooting
   ```
   cat /var/log/messages | grep dhcp
   journalctl -u dhcpd
   ```

1. Apache troubleshooting
   ```
   cat /var/log/httpd/access_log | grep -iE '*.ign'
   ```

1. You can collect logs from all cluster hosts by running the following command from the 'ocp-svc' host:

   ```bash
   ./openshift-install gather bootstrap --dir ocp-install --bootstrap=192.168.44.2 --master=192.168.44.11 --master=192.168.44.12 --master=192.168.44.13
   ```

1. Modify the role of the Control Pane Nodes

   If you would like to schedule workloads on the Control Pane nodes apply the 'worker' role by changing the value of 'mastersSchedulable' to true.

   If you do not want to schedule workloads on the Control Pane nodes remove the 'worker' role by changing the value of 'mastersSchedulable' to false.

   > Remember depending on where you host your workloads you will have to update HAProxy to include or exclude the Control Pane nodes from the ingress backends.

   ```bash
   oc edit schedulers.config.openshift.io cluster
   ```

   ## Reference

   1. https://docs.openshift.com/container-platform/4.11/installing/installing_bare_metal/installing-bare-metal.html#installation-requirements-user-infra_installing-bare-metal
   1. https://access.redhat.com/documentation/en-us/openshift_container_platform/4.7/html/support/troubleshooting#installation-bootstrap-gather_troubleshooting-installations
