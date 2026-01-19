# Nested OpenShift Sandbox Labs
## Lesson 4 - Compact Three Node OpenShift Deployment

## Overview
OpenShift is typically deployed as a cluster of multiple nodes of different types. A standard configuration you'll typically see in a production or multi-node cluster will have:

- Control Plane Nodes: These host all of the management and control plane components such as the Kubernetes/OpenShift API Server, an etcd key-value pair database, etc... In a production environment, we'll typically deploy three of these so that we have availability for control plane components in case of a single node failure, and workloads (containers/applications/VMs) are not scheduled to run on them. For a vSphere parallel, you can think of the Control Plane nodes as hosting many of the services you would find on a vCenter - plus some.

- Worker Nodes: These are the cluster nodes that will typically run workloads in the cluster, and interact with the control plane nodes for scheduling of the workloads running on the cluster. The parallel to a vSphere environment here would be ESXi nodes in an HA/DRS cluster that run VM workloads.

## Compact Three Node Cluster Overview
The configuration we're deploying in this lesson (Compact Three Node) is another small footprint deployment pattern for OpenShift. It is comprised of three "full" OpenShift nodes that run both control plane and worker functions.

In contrast to the Two Node Arbiter configuration in lesson 3, instead of having a third smaller arbiter node for etcd and storage quorum, the third node participates as a full control plane and worker node.

In this lab lesson you'll have two choices for storage deployment. You can either deploy Portworx Enterprise like we did for the TNA cluster, or you can deploy OpenShift Data Foundation (ODF), which is a ceph-based software defined storage solution. Compared to the Single Node OpenShift (SNO) deployment in lesson two, you'll again be able to play around with a couple of things you weren't able to in the SNO lab:

- Shared Storage: Both ODF and Portworx Enterprise give us the capability to deploy up to three replicas of our VM disks within OpenShift Virtualization when using three nodes. This effectively gives us the equivalent of a FTT=2 vSAN configuration for our VMs, and allows OpenShift Virtualization to restart VMs on surviving nodes if we have a full node failure.
- vMotion/Live Migration Capability: Unlike the LVM Storage configuration in the SNO lab lesson, both ODF and Portworx Enterprise support ReadWriteMany (RWX) block volume deployment for our VMs. This provides the underlying storage capability for us to be able to Live Migrate VMs between the OpenShift nodes that will be running the VM workloads in our lab.

*IMPORTANT:* If you have deployed the SNO VM node or the TNA VM nodes previously, please ensure they are shutdown and powered off if you only have 64GB of RAM allocated to your `fedora-kvm` VM - as the Three Node Cluster deployment will consume up to 61GB of memory once fully deployed. You can do this by running the following command in a terminal on the `fedora-kvm` host before executing the procedures in this lab lesson:  

If you've deployed the SNO node from lesson 2 previously:  
```
sudo virsh shutdown ocp-0-s
```

If you've deployed the Two Node Arbiter (TNA) nodes from lesson 3 previously:  
```
sudo virsh shutdown ocp-0-t
sudo virsh shutdown ocp-1-t
sudo virsh shutdown arbiter-t
```

## Architecture Details
![Nested OpenShift Sandbox Labs - Three Node Cluster Architecture](/lab-lessons/Lesson-4_3nc_deploy/images/Fedora-Three-Node-Compact-Cluster.png)  

Virtualization solutions require the triad of compute, network, and storage in order to pass these abstracted components onto the VMs for them to operate properly.

OpenShift uses what we call "Operators" to provide functionality to extend the base Kubernetes capabilities of the cluster. 

For the compute/virtualization capability, we'll be installing the OpenShift Virtualization Operator that is built upon the upstream KubeVirt project. This will allow KVM VMs to run on the OpenShift nodes.

For networking capabilities that you might be used to in vSphere, we'll install the NMState Operator, which will allow us to create a dedicated bridge - so you can plumb in tagged or untagged network segments for virtual machines to communicate on the network.

And for storage for this three node compact cluster, as noted above - you'll have the choice of deploying OpenShift Data Foundation (ODF) or Portworx Enterprise. This gives us the capability to use local storage attached to the OpenShift nodes and provide us persistent storage capabilities within OpenShift to house our VM disks.

Since you'll be using the `fedora-kvm` host as a bastion to deploy and access the OpenShift environment, you may find it useful to open this page in Firefox on your `fedora-kvm` host - that way you can easily copy and paste commands from the GitHub repo and don't have to type everything in by hand.

Let's get started!

## Deploying a Three Node OpenShift Cluster Using Ansible
**Why are we doing this?**  
We could go through and configure our OpenShift cluster by hand, using the Agent-Based Installer to deploy OpenShift. But we'd have to be documenting MAC addresses of our VM network adapters, generating YAML files for the installer, manually mounting an ISO to our VM nodes, powering them on, etc...

The Ansible role we are using automates all of the processes we would have to go through manually, reducing human error and giving us a usable OpenShift installation quickly so you can get to learning.

The Ansible role will:
- Perform some pre-flight and configuration checks to make sure DNS is properly setup, checks for existing VMs that might conflict with our three node cluster deployment, and verifies that critical variables are set for installation
- Create the root disk for the OpenShift cluster nodes, and create virtual/emulated NVMe devices that we'll use for the ODF or Portworx Enterprise backing devices
- Create the OpenShift node VMs with the proper CPU/Memory/Storage configurations that the OpenShift Agent-Based Installer expects
- Create the OpenShift agent-config and install-config YAML files that the installer consumes, which define the desired OpenShift configuration to be installed
- Create the custom OpenShift installation ISO, which is customized with the IPs, DNS names, and node configurations for your specific install
- Mount the custom installation ISO to the OpenShift node VMs and powers on the VMs, which will start the OpenShift installation process
- Monitor the OpenShift Agent-Based Installer for completion or errors, and perform various post-installation cleanup

**How to do it:**  
We'll be deploying our three node cluster using an Ansible role that I wrote, which utilizes the OpenShift Agent-Based Installer under the hood.

Feel free to look through the Ansible role tasks that are executed to perform the install and configuration - you can find these in the tarball located in this repo at `ansible/roles/ocp-vm-build.tar.gz`. I've intentionally written this role to use CLI commands instead of API calls so it's easier for you to follow what we're doing in the automation - so if you want to dig deeper into the "how does this work?" - feel free!

It's critical that you log into the `fedora-kvm` server console and NOT use the Cockpit terminal to run the Ansible playbook. The reason for this is that the Cockpit terminal will likely time out since this step takes about an hour to execute; we want to ensure that the playbook runs fully to completion. 

Open the console of your `fedora-kvm` VM, login as your user account, and start a terminal session by clicking on the terminal icon in the dash.

First, let's make sure that you've got the latest and greatest code from this repo, since sometimes we're tweaking things for kaizen. From your terminal session in the VM console, type:

```
cd ~/working/nested-openshift-sandbox-labs/
```

Pull the latest codebase:
```
git pull
```

Change to your local ansible working directory:
```
cd ~/ansible
```

Then copy over the latest playbooks for setup:
```
cp ~/working/nested-openshift-sandbox-labs/ansible/{kvm_host_perf_tune.yml,ocp_compact_install.yml} .
```

Next, let's execute the playbook to deploy OpenShift:

```
ansible-navigator run -m stdout ocp_compact_install.yml --enable-prompts --ask-vault-pass -e local_auth_folder=/home/$(whoami)/ansible/
```
You'll notice a couple of extra arguments that we're passing ansible-navigator here.

One is `--ask-vault-pass` - since the playbook is using variables stored in the Ansible vault file `sens_vars.yml` that we created, you're going to have to type in your vault password from when you created the vault file previously.

The second additional argument is `-e`, which is setting the `local_auth_folder` variable to your `~/ansible` directory by executing `whoami` to get your username.

You'll be prompted twice for a password - once for the `BECOME password` (your normal user account password) - and once for your vault password. Enter them and the playbook should begin execution:
![Executing the Three Node Cluster Ansible Playbook](/lab-lessons/Lesson-4_3nc_deploy/images/Compact-Deploy-Playbook-Start.png)

This is going to take a while - a typical OpenShift Agent-based Install will take up to 45-60 minutes, and we do a few things like certificate rotation after the install is complete so you can suspend/resume your OpenShift cluster.

So, relax - go grab a cup of coffee! While we wait for the OpenShift installer to do it's thing, you will see the Ansible playbook displaying debug `FAILED - RETRYING:` messages on the screen:
![Three Node Cluster Ansible Retry Warnings](/lab-lessons/Lesson-4_3nc_deploy/images/Compact-Deploy-Ansible-Warnings.png)

Don't be alarmed by this - the playbook is simply in a retry loop looking for completion of the OpenShift installation. Just be patient and let the playbook complete.

Once the install is successful and the Ansible playbook is complete, you should see a play recap displayed from ansible-navigator:
![Three Node Cluster Ansible Play Recap](/lab-lessons/Lesson-4_3nc_deploy/images/Compact-Deploy-Ansible-Completed.png)

**More Information:**  
For additional information on the installation process that was automated and for three node cluster information, take a look at the following resources:
- [OpenShift Agent-Based Installer](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/installing_an_on-premise_cluster_with_the_agent-based_installer/preparing-to-install-with-agent-based-installer)
- [Three Node Compact OpenShift Cluster Resource Planning](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/installing_an_on-premise_cluster_with_the_agent-based_installer/preparing-to-install-with-agent-based-installer#agent-based-installer-recommended-resources_preparing-to-install-with-agent-based-installer)

## Configuring OpenShift Authentication, OpenShift Data Foundation or Portworx Enterprise Storage, Networking, and Virtualization
Now that we have OpenShift deployed, let's make sure we can access the cluster. If you've ever worked on a Kubernetes cluster before, you may have used the `kubectl` command. OpenShift can be administered using `kubectl`, but for OpenShift-specific administration, Red Hat ships the `oc` command which is very similar to `kubectl`. `oc` has been installed on the `fedora-kvm` machine as part of the OpenShift install playbook, and the playbook has also copied our kubeconfig to the proper directory and file (`~/.kube/config`) so that you and the Ansible automation can use `oc`.

Run the following from your terminal to ensure you can use the `oc` commands:
```
oc get nodes
```
You should see your `ocp-0-c.compact.ocp.lab.example.com`, `ocp-1-c.compact.ocp.lab.example.com`, and `ocp-2-c.compact.ocp.lab.example.com` nodes, and their statuses should be `Ready`.

## Configuring htpasswd OpenShift Authentication
**Why are we doing this?**  
A fresh OpenShift install comes with one temporary account named `kubeadmin` to access the cluster with. However, best practice is to configure a more permanent method/Identity Provider to authenticate to the cluster with for administration and access.

We'll be using the htpasswd Identity Provider for these lab lessons, which will create an `admin` user with a password of `sandbox`.

**How to do it:**  
Let's run another Ansible playbook to configure a proper Identity Provider for the cluster and give a simple username/password for you to login to the Console UI (web interface) of the OpenShift cluster:
```
cp ~/working/nested-openshift-sandbox-labs/ansible/ocp_auth_setup.yml ~/ansible/
```
```
cd ~/ansible
```
```
ansible-navigator run -m stdout ocp_auth_setup.yml --enable-prompts
```

**More Information:**  
For additional information on OpenShift Identity Providers, take a look at the documentation [here](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/authentication_and_authorization/understanding-identity-provider)

## Storage Options  
Our OpenShift cluster doesn't have any persistent storage right now. Since we have multiple nodes that will be hosting virtual machines, we need some type of shared storage. You have two options to deploy storage for OpenShift Virtualization in this lab lesson - please choose **only one!** The configurations for both ODF and Portworx have been slightly modified from a default installation for both storage solutions.  

The OpenShift Data Foundation configuration in this lab will deploy an ODF cluster with RBD Block and Noobaa Object storage. A typical ODF installation would also include a CephFS deployment to give block, file, and object storage for workloads running on OpenShift. CephFS was excluded because RBD Block support RWX volumes to enable Live Migration of VMs; typically we deploy VMs on block storage as opposed to any filesystem-based storage to ensure performance - and excluding it in these labs gives us a bit more extra CPU and memory for test VMs to consume.

The Portworx Enterprise configuration in this lab uses Portworx' `StoreV1` storage backend, which uses BTRFS as the backend and has lower resource requirements as opposed to the new `StoreV2` storage backend, which uses DM as the storage backend. This does not change any volume mode capabilities within Portworx, but like the change to the ODF configuration - gives us a bit more extra CPU and memory for the test VMs to consume. Like ODF, Portworx provides RWX Block capabilities for the Live Migration capability we need to migrate VMs between nodes while the VMs are running.

Again, please choose only one storage option to deploy in the lab lesson - both use the same emulated NVMe drives attached to the OpenShift nodes, and the storage deployment will fail if you try to deploy both.  

## STORAGE OPTION 1: Configuring OpenShift Data Foundation  
**Why are we doing this?**  
If you're using Proxmox right now, you may already be familiar with ceph as a storage solution. OpenShift Data Foundation is a storage solution built upon ceph. As mentioned above, it provides true unified storage (block, file, and object), providing a storage foundation for practically any workload that is container or VM-based that you need to run on OpenShift.

Instead of a standard ceph configuration that uses backing storage devices directly, ODF uses the `Local Storage Operator` (LSO) - which creates `PersistentVolumes` within Kubernetes that map to the physical storage drives - and then OpenShift Data Foundation uses those PVs as it's backing devices to create the ODF StorageCluster.

In this set of configuration tasks, we'll install and configure the LSO first, and then deploy OpenShift Data Foundation after that in order to have storage available for our VMs later.

**How to do it:**  
Again, let's use Ansible to install and configure everything. Copy over the necessary playbooks by typing:
```
cp ~/working/nested-openshift-sandbox-labs/ansible/{ocp_lso_setup.yml,ocp_odf_setup.yml} ~/ansible/
```
```
cd ~/ansible
```

First, let's install and configure the Local Storage Operator:
```
ansible-navigator run -m stdout ocp_lso_setup.yml --enable-prompts
```

Once completed, we can then install OpenShift Data Foundation:
```
ansible-navigator run -m stdout ocp_odf_setup.yml --enable-prompts
```

Once this playbook completes, we now have highly available persistent storage for our VMs and containers on our OpenShift nodes. Skip storage option 2 (Portworx Enterprise), and continue with the `Configuring OVS Networking for VMs` tasks below.

**More Information:**  
For additional information about OpenShift Data Foundation storage, check out the official [documentation](https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/4.20).  

## STORAGE OPTION 2: Configuring Portworx Enterprise Operator and StorageCluster
**Why are we doing this?**  
Portworx Enterprise provides both `Filesystem` and `Block` persistent storage capabilities for OpenShift workloads, but lacks native object storage capabilities and is a fully commercial offering (not based on open source) in contrast to ODF. We will use Portworx Enterprise to create a `StorageCluster` resource using the virtual NVMe drives attached to the full node VMs as backing storage for the StorageCluster.

In this set of configuration tasks, the Ansible playbook will install the Portworx Enterprise Operator, and then deploy a Portworx StorageCluster (STC) operating in a 30-day trial mode that will provide storage for our VMs.

**How to do it:**  
Let's again copy over the necessary playbook and run it to install and configure Portworx. Type:
```
cp ~/working/nested-openshift-sandbox-labs/ansible/ocp_px_3nc_setup.yml ~/ansible/
```
```
cd ~/ansible
```
```
ansible-navigator run -m stdout ocp_px_3nc_setup.yml --enable-prompts
```

Once this playbook completes, we now have highly available persistent storage for our VMs and containers on our OpenShift nodes.

**More Information:**  
For more information about Portworx Enterprise, check out the documentation [here](https://docs.portworx.com/).

## Configuring OVS Networking for VMs
**Why are we doing this?**  
So far, we've just been operating our full nodes with a single network adapter assigned to a Linux bond interface (`bond0`), and the bond is assigned to the default bridge in OpenShift (`br-ex`).

The default `br-ex` bridge in OpenShift uses NAT and masquerading to provide egress capability for the pods and VMs in OpenShift to access resources outside the cluster. But VMs are typically available directly via their IP in a virtualization environment - so we need to configure the network on our OpenShift cluster to achieve this. In order to separate our Kubernetes/OpenShift management and NAT traffic from VMs that should live directly on external networks, we'll create another bridge (`ovs-br1`), using the unused `enp10s0` interface that is assigned to another bond (`bond1`) on the cluster node. 

We'll then map the bridge to a new `localnet` network in OVS that will give the VMs we'll deploy access to the `192.168.122.0/24` network. This is very similar to using a distributed port group in vSphere.


To do this, we will:
- Install the `NMState` operator in OpenShift
- Create a NetworkNodeConfigurationPolicy (`NNCP`)
- Create a NetworkAttachmentDefinition (`NAD`) for the namespace in OpenShift that our VMs will reside in

**How to do it:**  
Again, let's run some Ansible to do this!
```
cp ~/working/nested-openshift-sandbox-labs/ansible/ocp_nmstate_setup.yml ~/ansible/
```
```
cd ~/ansible
```
```
ansible-navigator run -m stdout ocp_nmstate_setup.yml --enable-prompts
```

Once this playbook completes, we've now plumbed our network so our VMs can get IP addresses from the libvirt default network (`192.168.122.0/24`) running on the `fedora-kvm` host through our secondary interface on the OpenShift nodes.

We can either let the DHCP server on the default network assign an IP, or we can statically assign an IP - and the VM will be able to communicate on the network directly as well as out to the Internet!

**More Information:**  
To learn more about the networking configuration used in this step, see the following resources:
- [Kubernetes NMState Operator](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html-single/networking_operators/index#k8s-nmstate-about-the-k8s-nmstate-operator)
- [Node Network State and Configuration](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/kubernetes_nmstate/k8s-nmstate-updating-node-network-config)

## Configuring OpenShift Virtualization
**Why are we doing this?**  
The OpenShift Virtualization Operator essentially enables the OpenShift cluster to have capabilities from the upstream KubeVirt project to allow management of virtual machines via Kubernetes constructs.

It also provides us a nice interface via the OpenShift Console UI to manage and operate our VMs on OpenShift.

**How to do it:**  
Like the other Operators, we'll use Ansible to install and configure the OpenShift Virtualization Operator:
```
cp ~/working/nested-openshift-sandbox-labs/ansible/ocp_hco_setup.yml ~/ansible/
```
```
cd ~/ansible
```
```
ansible-navigator run -m stdout ocp_hco_setup.yml --enable-prompts
```
Once this playbook completes, OpenShift Virtualization is installed and ready to run virtual machines! 

**More Information:**  
You can find additional information about OpenShift Virtualization and KubeVirt in these links:
- [OpenShift Virtualization Documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/virtualization/about)
- [KubeVirt Project](https://kubevirt.io/)

## Accessing the Compact Three Node OpenShift Cluster
All of our Operators are installed at this point, and now you should be able to login to the cluster via the OpenShift Console. From the console of the `fedora-kvm` machine, open Firefox and visit:
```
https://console-openshift-console.apps.compact.ocp.lab.example.com
```
You'll need to acknowledge the self-signed SSL warnings (there should be two of them - one for the console URL, and one for OAuth), and then you'll be presented with a screen where you can choose the Identity Provider you want to login with. Select the `ocp_htpasswd_provider` at the bottom:
![Selecting the htpasswd IdP for Login](/lab-lessons/Lesson-4_3nc_deploy/images/Compact-Console-Select-IDP.png)

When asked for a username and password, login with a username of `admin` and a password of `sandbox`:
![Logging Into the Console Using Admin Account](/lab-lessons/Lesson-4_3nc_deploy/images/Compact-Console-htpasswd-Login.png)

Once you login, you'll be dropped into the default "Administrator" view of the OpenShift Console:
![Logged Into the Console Successfully](/lab-lessons/Lesson-4_3nc_deploy/images/Compact-Console-Logged-In.png)

Feel free to browse around and get familiar with the console. There's probably alot that you may not be familiar with - but please be careful here as you are logged in with a cluster-admin role, and can do some damage. But, since this is a VM-based install of OpenShift, you can always easily delete your VMs and redeploy following the instructions in this repo!

## Prepping a Small Bootable Volume for VM Deployment
**Why are we doing this?**  
Typically when you install the OpenShift Virtualization Operator, one of the first things the Operator will do is download several bootable volume templates for you to use as sources for deploying common VMs. These take a while to download, and consume disk space on your OpenShift cluster.

Since these labs are nested, have fewer resources than a standard OpenShift install, and have minimal disk space - I've disabled this initial download of bootable volume templates.

So, we need to find a nice and compact disk image to download that we can then deploy VMs from to learn and test with in this environment.

In this step, we will create a new bootable volume using a Fedora Cloud image and limit it's size to 5GB instead of the typical 30GB volumes that are pre-created during a normal OpenShift Virtualization install.

**How to do it:**  
You should already be logged into the OpenShift Console from the previous step. If not,log in using the directions above.

Navigate to the `Virtualization` section in the left pane of the OpenShift console, and select the `Bootable Volumes` item under the Virtualization heading. Click the "Add Volume" dropdown in the middle of the page, and select "With form":
![Starting the Add Volume Form Wizard](/lab-lessons/Lesson-4_3nc_deploy/images/Compact-Add-Bootable-Volume-Form.png)

The New Volume wizard will appear. Configure the bootable volume with the following parameters:
- **Source type**: URL
- **Image URL**: ```https://download.fedoraproject.org/pub/fedora/linux/releases/43/Cloud/x86_64/images/Fedora-Cloud-Base-Generic-43-1.6.x86_64.qcow2```
- **Storage Class**: ocs-storagecluster-ceph-rbd (if using ODF) **or** px-rwx-block-kubevirt-3nc (if using Portworx)
- **Volume Mode**: Block
- **Access Mode**: Shared access (RWX)
- **Disk size**: 5 GiB
- **Volume name**: fedora-cloud-5gb
- **Destination project**: default
- **Preference**: linux

Once you create the volume with the parameters above, you will see it appear in the "Bootable Volumes" page and it will indicate "Clone in progress""
![Cloning of Bootable Volume in Progress](/lab-lessons/Lesson-4_3nc_deploy/images/Compact-Bootable-Volume-Cloning.png)

During this time, OpenShift Virtualization is downloading the qcow2 image and populating what we call a `DataVolume`. A DataVolume is an abstraction used by the Containerized Data Importer (CDI) that abstracts the Kubernetes PersistentVolumeClaim (PVC) resource. We can either use it as a base disk within a VM definition, or use it as a source to clone from to deploy a virtual machine.

You can check on the status of the Fedora Cloud qcow2 image import by navigating to the terminal on your `fedora-kvm` machine and type:
```
oc get dv fedora-cloud-5gb
```

Once the image is done downloading and the DataVolume creation is complete, you should see it with a status of "Succeeded", with Progress of 100.0%:

![Fedora Cloud qcow2 Image Successfully Downloaded](/lab-lessons/Lesson-4_3nc_deploy/images/Compact-Fedora-Cloud-DV-Succeeded.png)

**More Information:**  
You can find additional information about the steps we just performed below:
- [CDI DataVolumes](https://github.com/kubevirt/containerized-data-importer/blob/main/doc/datavolumes.md)
- [Containerized Data Importer](https://kubevirt.io/user-guide/storage/containerized_data_importer/)

## Deploying Your First OpenShift Virtualization VM  
**Why are we doing this?**  
There are multiple methods of deploying a VM in OpenShift Virtualization, and we'll cover more of them in later lab lessons - but we'll use the Console UI to do our first one. This method allows you to select through the web interface the DataVolume to use as the template for your VM, enter some details about how you want the destination VM customized, and deploys the VM with those customizations.

In this step, we will:
- Select our `fedora-cloud-5gb` bootable volume as a template to clone from
- Create an SSH key to inject into all VMs we deploy into the default namespace
- Modify the destination VM vNIC to use the `ovs-br1` bridge so it gets an IP on the external default libvirt network hosted by the `fedora-kvm` host
- Allow the VM to boot and customize itself
- Test network connectivity and verify the customization was successful

**How to do it:**  
Within the OpenShift Console UI, navigate to `Virtualization->Catalog` - you should be greeted with the "Create new VirtualMachine" page.

In step 1 of the wizard on the page, you should see the `fedora-cloud-5gb` bootable volume that we created in the previous step. Click on it so that it is highlighted (surrounded by a grey background) and scroll down to step 2 so we can select the InstanceType for the VM we want to deploy:
![Selecting the Fedora Cloud Bootable Volume](/lab-lessons/Lesson-4_3nc_deploy/images/Compact-Selecting-Bootable-Volume-Deploy-VM.png)

Next, find the `General Purpose - U Series` InstanceType and select `nano: 1 CPUs, 512 MiB Memory` InstanceType:
![Selecting the InstanceType](/lab-lessons/Lesson-4_3nc_deploy/images/Compact-Selecting-Instance-Type-Deploy-VM.png)

Scroll down to step 3, and modify the VM name to be `fedora-myfirst-vm`. Next, click on the pencil icon next to the text `Public SSH Key   Not configured`:
![Start SSH Key Config Dialog](/lab-lessons/Lesson-4_3nc_deploy/images/Compact-Start-SSH-Public-Key-Dialog-Deploy-VM.png)

This will present the Public SSH key dialog, where you can inject your public SSH key from the `fedora-kvm` bastion host into the VM so you can log into it after deployment. 

Select the "Add new" radio button, and select the "Automatically apply this key...." checkbox. Provide a name of `<username>-fedora-kvm`, where `<username>` is your user that you are logged onto the fedora-kvm host as.

Switch to your terminal on the `fedora-kvm` host, and type:
```
cat ~/.ssh/id_ed25519.pub
```
Copy the resultant text to your clipboard, switch back to the OpenShift Console web interface, and paste in your SSH key in the middle box of the dialog, then click "Save":
![Adding Public SSH Key](/lab-lessons/Lesson-4_3nc_deploy/images/Compact-Create-SSH-Key-Secret-Deploy-VM.png)

Finally, click on the "Customize VirtualMachine" button so we can map the vNIC to the appropriate network. On the left side of the customization interface, click on "Network". You'll see that the default is to have the VM connect to the pod network where it will not get an IP address on our `192.168.122.0/24` network:
![Network Customization Default View](/lab-lessons/Lesson-4_3nc_deploy/images/Compact-Customize-Network-Default-Settings-Deploy-VM.png)

To change this, click on the kebab icon on the far right of the page within the same row as the default network adapter, and select Edit:
![Editing the Default Network Adapter](/lab-lessons/Lesson-4_3nc_deploy/images/Compact-Customize-Network-Edit-Adapter-Deploy-VM.png)

This starts the "Edit network interface" dialog - within the `Network` section of the edit dialog, using the dropdown select the `default/br1-vm` bridge binding and click Save:
![Selecting the Bridge Binding](/lab-lessons/Lesson-4_3nc_deploy/images/Compact-Customize-Network-Assign-Bridge-Deploy-VM.png)

You should now see your VMs vNIC attached to the `br1-vm` bridge and the "Type" changed to `Bridge` instead of `Masquerade`.

Click the `Create VirtualMachine` button, and OpenShift Virtualization will begin deploying and customizing your first VM:
![First VM Provisioning](/lab-lessons/Lesson-4_3nc_deploy/images/Compact-Provisioning-Deploy-VM.png)

You should see your VM go from Provisioning status to Running status after a short time - it may take a few minutes to boot and customize, especially since we are running it as a triple-nested instance.

If you scroll down on your VirtualMachine detail page, on the right side about halfway down - you should see `Network (1)` - this is your VMs vNIC, and you should see an IPv4 IP address listed that is in the `192.168.122.0/24` subnet. This means your VM got an IP address from the DHCP service on the default libvirt network running on your `fedora-kvm` host:
![Viewing Virtual Machine IP Address Assignment](/lab-lessons/Lesson-4_3nc_deploy/images/Compact-Identify-VM-IP.png)

Since we had your SSH key injected to the VMs configuration at provisioning time, you should be able to SSH into your newly provisioned VM. Open a terminal on the `fedora-kvm` host, and type:

```
ssh cloud-user@<IP Address>
```

Where `<IP Address>` is the IP you saw assigned to your VM in the OpenShift Console UI. You'll be prompted if you want to continue connecting - type `yes`, and you should be connected to your `fedora-myfirst-vm` machine and have a bash prompt to work from:

![Connecting to fedora-myfirst-vm via SSH](/lab-lessons/Lesson-4_3nc_deploy/images/Compact-SSH-To-First-VM.png)

Go ahead and try pinging a host on the Internet (i.e., www.redhat.com) and it should succeed. You should also be able to ping your `fedora-kvm.lab.example.com` host as well as your `ocp-0-c.compact.ocp.lab.example.com`, `ocp-1-c.compact.ocp.lab.example.com`, or `ocp-2-c.compact.ocp.lab.example.com` host IPs on the `192.168.122.0/24` subnet. If you can, then everything is working and you did great deploying and configuring OpenShift Virtualization, and deploying your first VM onto an external network!

To exit your SSH session to your VM, simply type:
```
exit
```

**More Information:**  
You can read more about creating a VM from a template using OpenShift Virtualization [here](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/virtualization/creating-a-virtual-machine#virt-creating-vm-from-template_virt-creating-vms-from-templates).

## Performing Your First Live Migration  
**Why are we doing this?**  
Live Migration (the equivalent of vMotion using KubeVirt) is a critical capability in a virtualization solution - so we're going to test it out!

The requirements for Live Migration are pretty simple. You must have more than one worker node that can run VMs, and you must be using a CSI storage solution that provides ReadWriteMany (RWX) PersistentVolume capabilities where two hosts can access the VMs disk at a single point in time. All of your VMs disks that are used for persistent storage must be located on RWX volumes - and in this lab lesson, ODF or Portworx Enterprise give us RWX block volumes that you've just deployed your first VM onto.

The process is very similar to how a vMotion functions in a vSphere environment, and this capability has been available in KVM for many years - so it's just a reliable as vMotion in vSphere. Essentially KVM and QEMU work together to transfer the CPU state, initial memory snapshot, and dirty pages to the destination host, and perform a final cutover of the memory and VM state. Similar to vMotion, you may see a short (less than a second) drop of the VM when the final memory cutover occurs.

Let's test it out!

**How to do it:**  
First, let's login via SSH to your `fedora-myfirst-vm` virtual machine you created in the last step. Open a terminal on the `fedora-kvm` host, and type:

```
ssh cloud-user@<IP Address>
```

Where `<IP Address>` is the IP you saw assigned to your VM in the OpenShift Console UI. You should be logged in immediately since we accepted the new SSH fingerprint when testing our login in the last step.

Once logged in, start a ping to `www.redhat.com` by typing the following:

```
ping www.redhat.com
```

This will start a ping to the Internet that we'll monitor as the VM gets migrated between your OpenShift nodes in the compact cluster:

![Establishing an outbound ping to the Internet](/lab-lessons/Lesson-4_3nc_deploy/images/Compact-Ping-To-Internet-Pre-LM.png)

Next, let's open another terminal session window so we can watch the VM migration occur using the `oc` CLI utility. Click on the `+` symbol in your Gnome terminal to open another terminal window on the `fedora-kvm` host, then right-click on the new terminal tab and select `Detach Terminal`:

![Detaching the new terminal](/lab-lessons/Lesson-4_3nc_deploy/images/Compact-Detach-New-Terminal.png) 

This should open another separate terminal window - go ahead and drag it underneath the one with the ping to www.redhat.com so you can see both with the OpenShift Console UI web interface in the background where you can click the `Actions` dropdown on your `fedora-myfirst-vm` virtual machine in the background. Next, start a watch command in the second terminal by typing:
```
watch -n1 oc get vmi -n default -o wide
```

This will start a watch command on the VirtualMachineInstances (VMI) in the default namespace on your compact cluster. You should be able to see the `fedora-myfirst-vm` VMI you just deployed listed, and the node that it is currently running on (mine is running on the `ocp-1-c` node to start with):

![Watching the VMI via oc command](/lab-lessons/Lesson-4_3nc_deploy/images/Compact-Start-Watch-Command-Live-Migrate.png)

Now, from the OpenShift Console UI, select the `Actions` dropdown for your VM and select `Migration->Compute`:

![Starting the Live Migration](/lab-lessons/Lesson-4_3nc_deploy/images/Compact-Start-Live-Migration-UI.png)

This will bring up the `Migrate VirtualMachine to a different Node` wizard. We'll leave the `Automatically selected Node` radio button selected:

![Automatic node selection for Live Migration](/lab-lessons/Lesson-4_3nc_deploy/images/Compact-Auto-Select-Migration-Node.png)

You'll need to be quick for this next step. We're going to click the `Migrate VirtualMachine` button and then immediately close the browser so that you can see the migration occur from the watch command, and ensure your SSH session to the VM that is running the ping to www.redhat.com stays alive. Click the `Migrate VirtualMachine` button to initiate the Live Migration, and observe the node that the VirtualMachineInstance transfers to (mine migrated to ocp-2-c):

![Live Migration occurring](/lab-lessons/Lesson-4_3nc_deploy/images/Compact-VM-Live-Migrated.png)

You should still have a live SSH session to your VM, and in my case, I didn't lose a single ping sequence as my `fedora-myfirst-vm` migrated to my ocp-2-c node in the compact cluster.

Go ahead and open Firefox again within the `fedora-kvm` host console, and open the OpenShift Console UI web interface again by browsing to:
```
https://console-openshift-console.apps.compact.ocp.lab.example.com
```

If needed, login with the username `admin` and a password of `sandbox`, then navigate to `Virtualization->Virtual Machines`, and click on your `fedora-myfirst-vm`. 

Feel free to issue a couple of additional Live Migrations to watch the "Status" and "Node" indicators within the Console UI as the VM migrates between nodes. After each migration, you can check your SSH/ping terminal session to see that your VM stayed up and running and that your SSH and ping sessions were not dropped or interrupted.

**More Information:**  
For additional information on Live Migration using KVM and QEMU, check out the following resources:
- [OpenShift Virtualization Live Migration](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/virtualization/live-migration)
- [KubeVirt Live Migration](https://kubevirt.io/user-guide/compute/live_migration/)

## Wrapping Up Lesson Four
If you've successfully completed this lab lesson, congrats!

We've covered the following items in this lab lesson:
- **Overview of OpenShift node types**  
- **Compact Three Node Cluster Overview and Architecture Details**  
- **Deploying a Compact Three Node Cluster as KVM VMs using Ansible automation**  
- **Configuring OpenShift Authentication, OpenShift Data Foundation or Portworx Enterprise Storage, Networking, and Virtualization using Ansible**  
- **Logging In To the OpenShift Console UI**  
- **Creating a Bootable Volume Using a URL Source**  
- **Deploying and Customizing a VM from a Bootable Volume Source**  
- **Verifying Access to a VM Deployed on an External Network Using OVS Bridges**  
- **Performing Your First Live Migration Using OpenShift Virtualization**  

I hope you've enjoyed these lab lessons so far, and keep an eye out for future lab lessons in this series!
