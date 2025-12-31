# Nested OpenShift Sandbox Labs
## Lesson 1 - Base Environment Build

### Welcome
Thanks for checking out this lab series! You probably got here from my [introduction LinkedIn post](https://www.linkedin.com/pulse/nested-openshift-sandbox-labs-introduction-tim-darnell-ea29c/), or from someone sharing the link with you - either way, welcome!

A bit of background - I started building these for a couple of reasons. I wanted to build my Ansible skills as I was taking my RHCE (EX200) exam - but I also wanted to create some automation to easily stand up environments on my laptop for my OpenShift training I'm required to go through as a specialist at Red Hat. So, I figured I would marry the two - and this became the result.

This is built to only run on libvirt/KVM - if you're looking for OpenShift deployment on other platforms, there are plenty of other things you can find that use public cloud providers or the vSphere provider. My goal here is to educate people who have lived most of their professional lives in vSphere, may not be Kubernetes or KubeVirt experts, and who haven't been exposed to native KVM much - these foundational principles should help if you choose to start looking at cloud native ways of managing VMs.

That being said - these labs aren't solely for vSphere admins - anybody can use them or contribute to them! Many vSphere folks I know haven't had strong backgrounds in Linux - so if you are someone already in the cloud native ecosystem, or are just wanting to play around - apologies in advance. We are going to cover some very basic Linux sysadmin concepts here so that we can grow this amazing community we have in cloud native. If you're one of those people, and you feel like something is misrepresented or incorrect - PLEASE call it out, file a PR, contact me. These are NOT going to be perfect - but with community contributions, they can be pretty close!

### Topics Covered in Lesson 1
This is a foundational setup that will support all of the rest of the lab lessons that will be documented in this repository - we're not even going to cover OpenShift installation in this lesson. What we will cover:

- Installation and configuration of Linux VMs
- Basic Linux environment setup and administration
- KVM and libvirt basics
- Red Hat Developer accounts
- Ansible automation basics

### Requirements
Please review the requirements in the "What Do I Need For This?" section of my [introduction LinkedIn post](https://www.linkedin.com/pulse/nested-openshift-sandbox-labs-introduction-tim-darnell-ea29c/). Again, I developed these labs on my Lenovo 21KWS laptop with 22 threads and 64GB of memory - they don't run super fast, but they are usable to learn some basics.

My "new to me" machine just arrived today (AMD Strix Halo with 128GB and 4TB of NVMe) - so I'll be testing these procedures on that as I polish up this lesson. In the future, I'll be loading Proxmox onto my secondary NVMe drive and testing everything with Proxmox as the base hypervisor host, but for now - everything has been developed with KVM as the base underlying hypervisor hosting the Fedora KVM host.

If you've not worked in nested VM environments before, just a disclaimer - sometimes things don't work the same as a level 1 VM or on bare metal. So, please - have some patience if you do run into issues. This is meant to be a learning experience, and with learning, sometimes come frustrations.

Enough of my intro blabber - let's get started!

### Sign up for a Red Hat Developer Account
First, if you don't have a Red Hat Developer account - let's get you one. Red Hat gives you access to most all of the technology you will need to learn OpenShift Virtualization, Ansible, and other Red Hat goodness - all for free to run in test environments.

Navigate to https://developers.redhat.com/ and click on the "Register for an account button":
![Red Hat Developer Account Signup](/lab-lessons/Lesson-1_base_env_build/images/RH_Dev_Account_Signup.png)

Fill out your information and check your email to validate your account. Then come back to https://developers.redhat.com/ and click the "Log in" button, and make sure you can log in successfully:
![Red Hat Developer Account Signin](/lab-lessons/Lesson-1_base_env_build/images/RH_Dev_Account_Login.png)

Keep your login information handy for later, as we'll need it when we configure Ansible and prep for our OpenShift cluster install.

### Creating the KVM host VM
We're going to use Fedora 43 Server for our KVM host to house the lab environments. Grab the ISO from https://download.fedoraproject.org/pub/fedora/linux/releases/43/Server/x86_64/iso/Fedora-Server-dvd-x86_64-43-1.6.iso.

Next, create a VM on your ESXi host, VMware workstation instance, Proxmox machine, wherever you can create a VM that has decent resource availability. Keep in mind we'll be running triple-nested VMs - so the more performant this base VM is, the better. If you feel like allocating more than the 24vCPU, 64GB of RAM, and ~700GB of NVMe/SSD-backed storage, then cool! 

The critical thing is to have the one vNIC on the VM that has a static IP on your internal network, can resolve Internet DNS names from a forwarding DNS server, and that has Internet access. What you need to bring to this lesson:
![Resource Requirements for Nested OpenShift Sandbox Labs](/lab-lessons/Lesson-1_base_env_build/images/Fedora-nested-ocp-base-environment.png)

We'll be building out the following base hypervisor VM in this lesson to host all of our nested OpenShift environments:
![Base Environment for Nested OpenShift Sandbox Labs](/lab-lessons/Lesson-1_base_env_build/images/Fedora-nested-ocp-fedora-kvm.png)


### Installing Fedora 43 Server
Mount the Fedora 43 Server ISO to the VM and boot from it to start anaconda (the graphical installer). Select your installation language and keyboard layout and click Continue:
![Anaconda Installer Welcome Screen](/lab-lessons/Lesson-1_base_env_build/images/Anaconda_Welcome_Screen.png)

The Installation Summary page appears. Click on "Installation Destination" and you will see the Device Selection page with your VMs disk already selected with a black checkmark on it. Nothing is needed to be changed here - simply click the Done button to take you back to the Installation Summary:
![Anaconda Disk Selection](/lab-lessons/Lesson-1_base_env_build/images/Anaconda_Disk_Selection.png)

Next, click on "Network and Hostname". You will see the single NIC you've assigned to your VM. In the lower left corner, enter `fedora-kvm.lab.example.com`, and click the Apply button next to the host name. 
![Anaconda Host Name Assignment](/lab-lessons/Lesson-1_base_env_build/images/Anaconda_Fedora_Hostname.png)

Click the Configure button in the lower right corner to edit the IP assignment for the NIC, and navigate to the "IPv4 Settings" tab for the interface. Change the Method from Automatic(DHCP) to Manual, and click Add in the Addresses window.

Enter an available static IP on the network your VMs vNIC is connected to, the CIDR prefix (netmask), and the default gateway for the network. Enter a DNS server that is reachable for the VM that can resolve names on the Internet. Do not enter anything within the "Search domains" section. Lastly, select the "IPv6 Settings" tab for the interface, and select "Disabled" from the dropdown.

Click the Save button which will take you back to the Network and Host Name page, then toggle the ethernet adapter off and on once to apply the change in IP settings (if you can test pinging the IP you assigned from another machine to ensure you have proper connectivity, all the better). Click Done in the upper left corner once complete:
![Anaconda Network Configuration](/lab-lessons/Lesson-1_base_env_build/images/Anaconda_Network_Configuration.png)

Click on Time and Date to set your local timezone.

Next, click on "Root Account", select the "Enable root account" radio button, set your root password, check the "Allow root SSH login with password" checkbox, then click Done:
![Anaconda Root Account Creation](/lab-lessons/Lesson-1_base_env_build/images/Anaconda_Root_Account.png)

Finally, click on "User Creation" and create yourself an account. Ensure both checkboxes are checked for wheel group membership and to require a password for the account, and set your password then click Done:
![Anaconda User Account Creation](/lab-lessons/Lesson-1_base_env_build/images/Anaconda_User_Account_Creation.png)

Click the "Begin Installation" button in the lower right corner, and wait for the install to complete. Once complete, the "Reboot System" button will appear - click it and let's get to configuring our hypervisor host!
![Anaconda Install Complete](/lab-lessons/Lesson-1_base_env_build/images/Anaconda_Reboot.png)


### Configuring Fedora 43 Server
You'll be greeted by a terminal login prompt once the VM reboots. Login with the root password that you set for the system:
![Initial Login to the Fedora System](/lab-lessons/Lesson-1_base_env_build/images/Fedora_KVM_Initial_Login.png)

Since you'll be using this VM as both the KVM hypervisor and a bastion host to access OpenShift, let's install a UI and some tools we need. Enter the following to install the necessary packages:
```
dnf install gdm gnome-terminal gnome-software firefox pip3 python3-libdnf5 virt-manager -y
```

Once the packages are installed, let's set the graphical target (UI) as the default for boot, enable the display manager service, then reboot the machine:
```
systemctl set-default graphical.target
```
```
systemctl enable gdm.service
```
```
reboot
```

You should be greeted by the display manager login screen with your user account you created at setup. Login with your password, and click "Skip" when asked if you want to take the tour:
![Initial Graphical Login to Fedora Server](/lab-lessons/Lesson-1_base_env_build/images/Fedora_KVM_Graphical_Login.png)

In the search bar at the top of the screen, start typing in "terminal" until the terminal icon appears - right click on the icon and select "Pin to Dash". This will pin the terminal shortcut to your dashboard at the bottom of the screen:
![Pinning the terminal shortcut to the Gnome Dash](/lab-lessons/Lesson-1_base_env_build/images/Fedora_Terminal_to_Dash.png)

Open the terminal by clicking on it from the dashboard. You'll probably notice the font spacing is a bit strange. To fix this, click on the hamburger menu in the upper right corner of the terminal window, and select "Preferences":
![Terminal Preferences Menu](/lab-lessons/Lesson-1_base_env_build/images/Fedora_Fix_Terminal_Font.png)

Select the "Unnamed" profile in the left pane of the Preferences dialog, and check the "Custom font" checkbox. You should see your terminal look a bit more usable now, and can close the preferences dialog:
![Setting Custom Font in Gnome Terminal](/lab-lessons/Lesson-1_base_env_build/images/Fedora_Fix_Terminal_Font_2.png)

 That's it for the basic setup to make the VM usable as a bastion host for now. Let's get to installing Ansible so that we can automate!

### Configuring Ansible on the Fedora 43 Server VM
We're going to install Ansible as an "Execution Environment" inside of a container to run on the Fedora 43 Server. You may have used Ansible in the past, having to install the Ansible packages on your control node and issuing commands such as ```ansible-playbook```. 

For this setup, we're going to run a fully self-contained Ansible node as a container via podman (similar to docker), commonly referred to as ```ansible-navigator```. If you've not used ansible-navigator or used execution environments, it would be good to get familiar with this type of configuration - it will be the future of Ansible moving forward.

To keep things clean, let's create a working directory for all of the Ansible work we're going to do in this lab. Open a terminal via the dashboard, and type:
```
mkdir ~/ansible
```

Next, let's install ansible-navigator and the python Kubernetes modules via pip3:
```
pip3 install ansible-navigator kubernetes
```

Now that we've got ansible-navigator installed, it's time to create our ansible/ansible-navigator.yml file, which will tell ansible-navigator which container image to use for the execution environment. Create this file by opening it in vi:
```
vi ~/ansible/ansible-navigator.yml
```

If you've not used vi or vim much, please learn it - it's a great editor to be familiar with. To start editing the file, press the `i` key to get into Insert mode, and populate the following (note each indent is two spaces - since this is YAML, do not use tabs - and proper indentation/spacing is critical!):
```
ansible-navigator:
  execution-environment:
    image: registry.redhat.io/ansible-automation-platform-26/ee-supported-rhel9:latest
    pull:
      policy: missing
  playbook-artifact:
    enable: false
```

Once you've entered this info, press the escape key once, then type:
```
:wq
```
followed by enter. This will save the contents of the ansible/ansible-navigator.yml file, exit vi/vim, and take you back to the terminal.

Next, we need to create our ansible/ansible.cfg file that will tell the execution environment and ansible-navigator how we want to run Ansible. Again, create the file by opening it in vi:
```
vi ~/ansible/ansible.cfg
```

Press the `i` key to get into Insert mode, and populate the following. Note that the four items in the `[defaults]` section will be specific to your user account - replace `<username>` with the user name you are logged in as:
```
[defaults]
inventory = /home/<username>/ansible/inventory
roles_path = /home/<username>/ansible/roles
collections_path = /home/<username>/ansible/my_collections:/usr/share/ansible/collections
remote_user = <username>
timeout = 60

[ssh_connection]
pipelining = true

[privilege_escalation]
become = true
become_method = sudo
become_user = root
become_ask_pass = true

[colors]
debug = color197
```

Once you've populated the file, press escape once, then type:
```
:wq
```
followed by enter to save the file and to take you back to the terminal.

You may notice that we've told Ansible we want to use an inventory file called "inventory" defined in the ansible.cfg file we just created. Let's create it and populate with our host information that we'll be managing (the Fedora 43 Server in this case). Create the file by opening it in vi:
```
vi ~/ansible/inventory
```

Press the `i` key to get into Insert mode, and populate the inventory group `kvm_target` and the host `fedora-kvm.lab.example.com` to target:
```
[kvm_target]
fedora-kvm.lab.example.com ansible_host=host.containers.internal
```
Ensure that you've added the variable `ansible_host=host.containers.internal` after the FQDN of your host - this will allow the ansible-navigator container to execute commands against the host it is running on properly (in our case, the Fedora 43 Server). Once properly populated, press escape once, and again type:
```
:wq
```
followed by enter to save the file and totake you back to the terminal.

The other items in the ansible.cfg defaults section were `roles_path` and `collections_path` - let's create those directories now. I'm sure you are familiar with PowerShell cmdlets as a vSphere admin - think of roles and collections as cmdlets in PowerShell - they are custom written automation routines that serve specific purposes for certain software or hardware automation tasks.

We'll be installing an Ansible role I wrote for deploying OpenShift on KVM VMs and some community collections to deploy the VM-based OpenShift clusters later on, and these directories are where they will live. Type:
```
mkdir ~/ansible/{roles,my_collections}
```

Ansible uses SSH to connect to managed systems and execute Ansible playbooks to manage and perform automation tasks on them. Even though we're running Ansible on a local system we're managing, we're still going to use SSH - so let's generate our SSH key for your user account. Type:
```
ssh-keygen
```
and take all of the defaults, do not enter a passphrase, and when prompted for the passphrase, just press enter.

We then need to enable passwordless login using the SSH key for Ansible since our "remote_user" in the ansible.cfg is your personal account on the Fedora 43 Server. Type:
```
ssh-copy-id <username>@fedora-kvm.lab.example.com
```
where `<username>` is your user name (i.e., tdarnell). Type `yes` when prompted if you are sure to continue, and enter your password when prompted:
![Enabling Passwordless Login via SSH](/lab-lessons/Lesson-1_base_env_build/images/Fedora_SSH_Copy_ID.png)

Finally, we're going to put that Red Hat Developer account you created to use. Since the execution-environment image defined in the ansible-navigator.yml file is provided from Red Hat, we need to login to the Red Hat Registry in order to pull down the container image. Type:
```
podman login registry.redhat.io
```
and enter your Red Hat Developer username and password when prompted. You should get a "Login Succeeded!" message when complete.

Before we get to installing configuring the rest of the Fedora 43 Server, let's install a few Ansible collections we need. Run the following commands to install the community.general, community.libvirt, community.okd, and fedora.linux_system_roles collections into our `my_collections` directory:
```
cd ~/ansible/
```
```
ansible-galaxy collection install community.general community.libvirt community.okd fedora.linux_system_roles -p my_collections/
```

### Enable libvirt on Fedora 43 Server and Prep for VMs
Ok, we finally have our hypervisor host built and our automation environment configured and ready to go - now we have to actually install the hypervisor!

This is simple - we're just going to install libvirt and enable the service, which will allow us to run virtual machines. If this is your first time using `sudo` - don't be concerned when asked for your password - sudo is elevating privileges for you in order to make administrative changes to the Fedora 43 Server:
```
sudo dnf install libvirt -y
```
```
sudo systemctl enable --now libvirtd.service
```

By default, libvirt will setup a NAT network which is where we're going to be connecting our private DNS server VM and OpenShift VMs to. However, the default network is fully DHCP by default - and the VMs we'll be deploying will have static IPs. 

In order to modify the DHCP range and give us some room on the default network for static IPs, we're going to use the `virsh` utility. This is another one of those commands similar to vi/vim that I recommend you get familiar with if you are going to be doing anything with KVM for virtualization.

From a terminal, simply type:
```
sudo virsh
```
You'll be dumped to the `virsh` shell - to look at the default network info, type:
```
net-info default
```
You'll see some details about the default network - but notice that it doesn't tell you anything about the default DHCP range that it provides. In order to modify the DHCP range, we'll need to edit the network. 

Most all components in libvirt are defined via XML - VM definitions, network definitions, etc... - so let's edit the network XML definition. Type:
```
net-edit default
```
You'll see the XML we need to edit - identify the line with the DHCP range start and end definition, and change the starting IP from `192.168.122.2` to `192.168.122.128`. This will allow us to define static IPs from `192.168.122.2` to `192.168.122.127` for the DNS Server and OpenShift VMs we will deploy. By default, `192.168.122.1` is assigned to the libvirt host on the `virbr0` bridge interface as the default NAT gateway for the VMs on the default libvirt network.

`virsh` uses vi as the editor for editing the libvirt XML elements. So, just like when we created the files for Ansible - press `i` to enter Insert mode, modify the IP start range with the proper value, press escape once to exit Insert mode, and type:
```
:wq
```
followed by an enter to save your edits to the default network XML. After editing, the DHCP range line in your XML file should look like this:
![Editing the Default libvirt NAT Network DHCP Range](/lab-lessons/Lesson-1_base_env_build/images/Virsh_Default_Network_Edit.png)

Finally, let's stop and restart the default network in order for our changes to the default DHCP range to take effect:
```
net-destroy default
```
```
net-start default
```
You'll notice the command to "stop" the network is actually `net-destroy` - don't worry, this won't delete the network - it only stops it. You'll see this terminology throughout libvirt when stopping VMs, networks, etc... - if you truly want to delete something, look into the `undefine` commands within virsh.

To verify that your changes took for the DHCP range, feel free to edit the default network XML again by typing `net-edit default` and you should see your start IP at `192.168.122.128`. 

Since we're not making any changes to the file in vi, you don't need to use `:wq` to exit the file (this stands for `write, quit` if you haven't guessed already). To exit vi/vim without making any changes to a file, simply type `:q!` followed by enter:
![Quitting vim Without Saving A Change](/lab-lessons/Lesson-1_base_env_build/images/Virsh_DHCP_Range_Edited.png)

After you've verified your changes to the network took, simply type `exit` to exit the virsh shell to get back to a bash terminal prompt.

The next thing we'll do to prep our Fedora 43 Server for hosting our VMs is to extend our root partition. The default install only creates a 15GB logical volume, even though we've allocated a larger base disk to the VM.

Libvirt will store any VM disk images in the `/var/lib/libvirt/images` directory by default, which falls under the / (root) mountpoint on a default install. To see what you've got allocated to the root partition, type:
```
df -h
```
You'll notice that the filesystem `/dev/mapper/fedora-root` only has 15GB allocated:
![Looking at the Default Root Filesystem](/lab-lessons/Lesson-1_base_env_build/images/Fedora_Default_Filesystem.png)

Let's change that.

If you are not familiar with LVM, logical volumes are carved from larger storage pools called volume groups. Let's take a look at the volume group first:
```
sudo vgdisplay
```
You should see a VG named "Fedora", and if you find the "Alloc PE / Size" entry, it will show 15GB currently allocated. Beneath that, you should see "Free PE / Size" - this is the amount of storage unallocated within the VG. Keep that number in mind as we extend our root logical volume:
![Viewing the default Volume Group](/lab-lessons/Lesson-1_base_env_build/images/Initial_Volume_Group_Display.png)

To look at the logical volume backing the `/dev/mapper/fedora-root` filesystem (/dev/fedora/root), type:
```
sudo lvdisplay
```
You can see that the "VG Name" (the volume group that the volume is carved from) is set to "fedora", which is the volume group we previously looked at - and that the "LV Size" is 15GB:
![Viewing the default Logical Volume](/lab-lessons/Lesson-1_base_env_build/images/Initial_Logical_Volume_Display.png)

Remember how many GB you had "Free" in your VG? Subtract 1GB from that and let's use that as the expansion size for the LV. In my example, I had 383GB free in my VG, so I'm going to extend my volume by 382GB. Type the following, replacing "382" with the value you want to grow your logical volume by on your system: 
```
sudo lvextend --size +382G fedora
```
Now run another `sudo lvdisplay` command, and you should see your `fedora` logical volume now has much more available space. But if you run `df -h`, you'll notice your root filesystem is still at 15GB capacity. Let's grow the filesystem to actually gain the space so it's usable:
```
sudo xfs_growfs /
```
Now that we've extended the filesystem, issue a `df -h` once again and now you should see that your root filesystem is much larger - so we can deploy our VMs without having to worry about disk space.

Finally, let's download a pre-configured cloud image for Fedora 43 Server that we'll use to deploy our internal BIND DNS server VM. Open a terminal, and let's grab the pre-configured qcow2 image:
```
wget https://download.fedoraproject.org/pub/fedora/linux/releases/43/Server/x86_64/images/Fedora-Server-Guest-Generic-43-1.6.x86_64.qcow2
```
Let the download finish, and then move the qcow2 image to the /var/lib/libvirt/boot/ directory:
```
sudo mv Fedora-Server-Guest-Generic-43-1.6.x86_64.qcow2 /var/lib/libvirt/boot/
```

Great - we've enabled libvirt, modified our DHCP address range, configured our filesystem to host our OpenShift and DNS VMs, and downloaded a Fedora pre-configured disk image to use for our DNS VM deployment. Let's move on!

### Enabling the Cockpit Web Administration Interface
Up until now, I've intentionally had you administering the Fedora 43 Server via traditional methods using the terminal. You've gotten a little taste of the `virsh` shell, which is how we've administered KVM virtual machines for years in Linux.

But this is all about learning Linux-based virtualization and some basic administration tasks before we get to OpenShift, so let's take a look at another method - Cockpit.

Cockpit is a nice web interface to give you information about your Linux server and provide a method to do some basic administration tasks. First, we'll enable cockpit and start the service:
```
systemctl enable --now cockpit.socket
```
You'll need to enter your password when prompted since you are enabling a systemd service as a non-root user (notice we didn't use `sudo` this time - when in a window manager and you need elevated privileges to modify system services, this is what you will see):
![Enabling the Cockpit Service](/lab-lessons/Lesson-1_base_env_build/images/Fedora_Enabling_Cockpit_Service.png)

Next, let's access the web interface. You can either use Firefox on the Fedora 43 Server itself, or you can use a browser from a machine that can access the IP you've assigned to the Fedora 43 Server on your network. Open a connection in the browser to:
```
https://<Fedora 43 Server IP>:9090
```
The server has generated a self-signed certificate for cockpit, so accept the self-signed message to proceed and you should be greeted with a login page to cockpit. 

Enter your username and password, then click the "Login" button. In the example below, I'm using Chrome from my workstation to access cockpit remotely from my workstation for connecting to the Fedora 43 Server:
![Accessing the Cockpit Web Interface](/lab-lessons/Lesson-1_base_env_build/images/Fedora_Accessing_Cockpit.png)

You'll be greeted by the cockpit web interface, and will see a message that the "Web console is running in limited access mode":
![Cockpit Limited Access Mode Message](/lab-lessons/Lesson-1_base_env_build/images/Fedora_Cockpit_Limited_Access_Mode.png)

This means that you have the same restrictions for administering the system as you would via the Gnome UI or terminal we've used so far - you are unable to modify the system without requesting administrative privileges via sudo.

You can click the "Turn on administrative access" to enable full administrative privileges for your account within cockpit, and will be prompted to enter your password to acknowledge you are requesting elevated access to administer the system, then click the "Authenticate" button:
![Enabling Cockpit Administrative Access](/lab-lessons/Lesson-1_base_env_build/images/Fedora_Enable_Cockpit_Admin_Access.png)

Once you have done this - keep in mind that you can now completely destroy your system if you are not careful - so be aware of this as you explore the capabilities you now have via cockpit!

There are some useful plugins for cockpit to administer certain functionalities on the Fedora 43 Server you've deployed - these are called "Applications" in cockpit. Find the "Tools" subheading in the left pane of the interface, and click on "Applications" below it:
![Browsing Cockpit Applications](/lab-lessons/Lesson-1_base_env_build/images/Fedora_Cockpit_Applications.png)

Scroll down and find the "Machines" Application, and click the "Install" button. This will enable management of libvirt and virtual machines via the cockpit interface. You should now see a "Virtual Machines" entry in the left pane of the cockpit interface. Click on it:
![The Cockpit Virtual Machines Application](/lab-lessons/Lesson-1_base_env_build/images/Fedora_Cockpit_VM_Application.png)

Sweet, now we can see things in a bit more graphical representation instead of looking at XML via the virsh shell! You'll notice that you have one network defined, so click on the "1 Network" link to see the default network entry, and expand it to view the details. You should now see the DHCP range that we edited via XML and the virsh shell that is defined within the default network on your Fedora 43 Server:
![Looking at the Default libvirt Network in Cockpit](/lab-lessons/Lesson-1_base_env_build/images/Fedora_Cockpit_VM_Network.png)

The last thing we are going to do before deploying VMs is update our Fedora 43 Server install. Within Cockpit, navigate to "Software updates" in the left pane. You should see all of the available updates for your system - go ahead and click "Install All Updates":
![Updating the Fedora 43 Server Using Cockpit](/lab-lessons/Lesson-1_base_env_build/images/Fedora_Cockpit_System_Update.png)

Updates will begin deploying to your server - you can view the updates occurring by expanding the update log in the Cockpit interface. 

You may have to log back in if Cockpit is one of the components being updated. Once all updates are complete, navigate to the Terminal within Cockpit and type:
```
sudo reboot now
```
Wait for Fedora to reboot, and then refresh Cockpit and log back in.


### Deploying a BIND9 DNS VM for Internal Name Resolution
DNS is a critical service needed for OpenShift to function. There are a few different ways we could deploy the DNS VM that we'll need for OpenShift using libvirt:

- Use the CLI and the libvirt `virt-install` command, defining our VM configuration via command line arguments passed to `virt-install`
 
- Create an XML definition for our virtual machine, register it with virsh using the `define` command, boot the VM from an installable ISO, and use the anaconda installer to configure the VM
 
- Use Cockpit and the "Machines" application to interface with libvirt, defining our VM configuration via the UI within Cockpit
 
- Use the Ansible community.libvirt collection to interface with libvirt, defining our VM configuration via arguments passed to the libvirt Ansible module

The disk image that we're using requires us to interact with the VM during its first boot to assign its IP address, set the hostname and root password, and create your personal account - so we're going to use the CLI method. 

Feel free to play around with creating a VM using Cockpit once we're done, and you'll see how to create a VM via Ansible later on when we deploy OpenShift - which uses an XML definition template and boots a VM via ISO to install CoreOS and configure OpenShift.

Instead of using the terminal within the console of the VM on your base hypervisor, navigate to Cockpit on your Fedora 43 Server VM, and select "Terminal" at the bottom of the left pane. I'm having you use this because you can copy and paste content into and out of this terminal (please don't get mad at me for having you actually type during the Ansible setup previously.) :)

The qcow2 image that you downloaded earlier using `wget` is going to be the base image we use for our DNS server VM - think of this almost like an OVA. To prepare the DNS server disk for deployment, let's make a copy of the qcow2 image we downloaded earlier. Within the Cockpit terminal, paste the following:
```
sudo cp /var/lib/libvirt/boot/Fedora-Server-Guest-Generic-43-1.6.x86_64.qcow2 /var/lib/libvirt/images/dns.qcow2
```
Next, let's use the `virt-install` utility to configure and boot our VM:
```
sudo virt-install --name dns --memory 2048 --cpu host --vcpus 1 --graphics none --os-variant fedora-unknown --import --disk /var/lib/libvirt/images/dns.qcow2,format=qcow2,bus=virtio --network network=default
```
You'll see the VM boot in the terminal - let it go until you see the configuration menu for the cloud image appear:
![Initial Config Menu for DNS VM](/lab-lessons/Lesson-1_base_env_build/images/Fedora_DNS_VM_Initial_Boot.png)

First, press the `3` key followed by enter to get into the network configuration menu. Once in the network configuration menu, let's set the hostname by pressing `1` and press enter again. Set the hostname for the VM to `dns.lab.example.com` and press enter:
![Setting the DNS VM Hostname](/lab-lessons/Lesson-1_base_env_build/images/Fedora_DNS_VM_Hostname.png)

You'll be returned to the network configuration menu after setting the hostname. Next, press the `2` key followed by enter to enter the device configuration menu:
![DNS VM Device Configuration Menu](/lab-lessons/Lesson-1_base_env_build/images/Fedora_DNS_VM_Device_Config.png)

I'm not going to walk you through the entire menu for device configuration, but you can follow the below since you're using the libvirt default network on 192.168.122.0/24 and your configuration should not need to deviate from the values I list here:
1) 192.168.122.2
2) 255.255.255.0
3) 192.168.122.1
4) ignore
5) `<no action>`
6) 8.8.8.8,8.8.4.4
7) `<no action>`
8) Checkbox selected
Once you are happy that the device configuration menu looks good, press `c` to get back to the network configuration menu, and `c` again to get back to the main configuration menu.

Press the `2` key followed by enter to configure your timezone settings, then press `1` to navigate to the timezone settings menu. Find your timezone by navigating the available regions menu until you select your timezone. Once you select your timezone, you should be taken back to the main configuration menu. 

Press `4` to enter the root password dialog, and enter the root password twice. Once complete, you'll be taken back to the main configuration menu.

Finally, press the `5` key to enter the user creation menu, and press `1` to enter the user creation dialog. You only need to modify items 2, 3, and 5 in this dialog. Ensure you use the same username as your account on the Fedora 43 Server in order to make Ansible configuration of the DNS server easier.

Once your user creation menu looks good and you've configured your user account, press `c` to be taken back to the main configuration menu:
![DNS VM Configuration Menu Complete](/lab-lessons/Lesson-1_base_env_build/images/Fedora_DNS_VM_Configured.png)

Press `c` one last time from the main configuration menu to complete the configuration of the VM, and you should see a login prompt to the DNS server:
![DNS VM Booted Post-Configuration](/lab-lessons/Lesson-1_base_env_build/images/Fedora_DNS_VM_Booted.png)

Login to the DNS VM as root, and type `reboot` to reboot the machine for all of the configurations you just modified to take effect.

To get back to your Fedora 43 Server terminal prompt after the VM has rebooted, press `Ctrl`+`]`, and you should see a `Domain creation completed.` message, and get dumped back to your fedora-kvm terminal prompt:
![DNS VM Creation Complete](/lab-lessons/Lesson-1_base_env_build/images/Fedora_DNS_VM_Install_Complete.png)

If you'd like, you can navigate to your Fedora 43 Server VM's Cockpit web interface and use the "Machines" application to look at the DNS VM that we just created. Next up, we'll configure the DNS VM using Ansible and prep our nameserver with the records necessary for our OpenShift installation.

### Configuring the DNS VM With Ansible
We've already added our Fedora 43 Server running KVM and libvirt to our Ansible inventory, and configured SSH keys for it. Now, let's do the same for our fresh DNS server so we can install and configure BIND using Ansible automation. Navigate to the `fedora-kvm` server Cockpit web interface, and select the Terminal from the left pane.

Open the ansible inventory file by typing:
```
vi ~/ansible/inventory
```
Press the `i` key to enter Insert mode, and add a new group in the file called `dns_server`, with the hostname `dns.lab.example.com` and a variable named `ansible_host` set to 192.168.122.2:
![Adding the DNS VM to Ansible Inventory](/lab-lessons/Lesson-1_base_env_build/images/Fedora_Add_DNS_To_Ansible_Inventory.png)

Once the inventory file looks good, press the escape key once and type`:wq` then enter to save the file and get back to the terminal.

Next, let's get passwordless login setup from our Ansible control node (your Fedora 43 Server) to the new DNS server VM. Type:
```
ssh-copy-id <username>@192.168.122.2
```
Ensure you replace `<username>` in the command above with your username on the `fedora-kvm` host. Type `yes` when prompted, then type your password when prompted to complete the SSH key copy:
![Performing SSH key copy for DNS VM](/lab-lessons/Lesson-1_base_env_build/images/Fedora_Add_SSH_Key_DNS_VM.png)

I'm going to make things a bit easier for you when using Ansible playbooks and roles to do the rest of the installs and configuration - and those are all in this git repository (`https://github.com/tdarnell-rh/nested-openshift-sandbox-labs`). But first, to download the files in the repo, we should install `git` on our Fedora 43 Server. From your Fedora 43 Server Cockpit terminal, paste in the following:
```
sudo dnf install git -y
```
Now you have the git tools necessary to clone all of the files from the git repo, so let's clone this repo down to the Fedora server. First, let's create a working directory for the lab files. Then from your Cockpit terminal, type:
```
mkdir ~/working
```
This creates a directory called `working` in your home directory of `/home/<username>/` (the ~/ denotes your home directory, where we've already created the `ansible` directory.)

Next, change directory into `working` and clone the lab repository:
```
cd ~/working
```
```
git clone https://github.com/tdarnell-rh/nested-openshift-sandbox-labs.git
```
This will clone the files we need down to your Fedora 43 Server - issue an `ls` from the terminal and you should see a `nested-openshift-sandbox-labs` directory.

There is an Ansible role (again, think PowerShell cmdlet) in this directory structure that we need to install using `ansible-galaxy` - let's find it and install to our local Ansible roles directory:
```
cd ~/working/nested-openshift-sandbox-labs/ansible/roles
```
```
ansible-galaxy role install fedora-bind-mgmt.tar.gz,,fedora-bind-mgmt -p ~/ansible/roles/
```
There is also an Ansible playbook we need to copy to our local Ansible directory - let's copy that over too:
```
cp ~/working/nested-openshift-sandbox-labs/ansible/dns_setup.yml ~/ansible/
```
Now we're ready to install BIND9 on the DNS VM, configure DNS forwarding and all of our forward/reverse lookup zones, and reconfigure the Fedora 43 Server and DNS VM to use the BIND9 instance as their DNS server. All through Ansible!

First, we'll change directory to our local Ansible working directory:
```
cd ~/ansible
```
Since we've rebooted the server after updates - your login to the registry where ansible-navigator will pull it's execution environment image from has expired.

Go ahead and log back into the Red Hat registry with your Red Hat Developer account credentials:
```
podman login registry.redhat.io
```
You should get a "Login Succeeded!" message indicating you have logged into the registry correctly. Next, we'll tell `ansible-navigator` that we want to run the `dns_setup.yml` Ansible playbook, and that it should prompt us for our password since we're performing automation actions that require elevated privileges:
```
ansible-navigator run -m stdout dns_setup.yml --enable-prompts
```
You'll see ansible navigator pull down the image from the Red Hat registry for the Ansible execution environment, and then it will prompt you for your `BECOME password` - enter your user password, press enter, and the playbook will begin execution:
![Executing the DNS Setup Ansible Playbook](/lab-lessons/Lesson-1_base_env_build/images/Fedora_DNS_Ansible_Playbook_Start.png)

Once this completes, you should see some output that tells you the overall execution status of the Ansible playbook, looking something like this:
![A Successful Ansible Playbook Run for DNS VM Setup](/lab-lessons/Lesson-1_base_env_build/images/Fedora_DNS_Ansible_Playbook_Complete.png)

Now for the real test - let's make sure we can resolve records that will be necessary for our OpenShift instances we will deploy. Let's see if we can resolve the DNS name `api.sno.ocp.lab.example.com` first - type:
```
dig @192.168.122.2 api.sno.ocp.lab.example.com
```
You should get a response from the DNS server VM - you are looking for the `ANSWER SECTION` and you should see an IP address of `192.168.122.10`:
![Testing DNS Resolution for OpenShift Records](/lab-lessons/Lesson-1_base_env_build/images/Fedora_DIG_Test_Results.png)

If you do see this, we're on track! This means that the zone records and configuration of the BIND server on the DNS VM were successful, and it is resolving queries sent to it for our lab domain names.

Next, let's just try a simple ping command from the `fedora-kvm` machine to ensure that the DNS client is set properly to query the DNS VM for names in our lab. Type:
```
ping api.tna.ocp.lab.example.com
```
Don't worry that you don't get a response, as there is no machine on the network yet corresponding to the IP address/name. What we're looking for here is that the name actually resolved to the IP address of `192.168.122.23`:
![Ensuring ping command resolves DNS names](/lab-lessons/Lesson-1_base_env_build/images/Fedora_Ping_Resolution_Test.png)

If it did, then congrats - all of the base infrastructure to support our labs is now complete - great job!

We'll want to reboot the `fedora-kvm` host before we continue for a couple of reasons:
- We want to make sure the DNS VM auto-starts upon boot of the host
- The DNS setup playbook made some modifications to the `fedora-kvm` host that will help with preventing sync page fault kernel panics (which can be common when running multiple nested layers of VMs if memory pressure occurs)

In a terminal on the `fedora-kvm` host, type:
```
sudo reboot now
```

Let your `fedora-kvm` host reboot, and then login to Cockpit to see if the DNS VM auto-started. If it did, we're golden - so let's move on!

### Prepping Ansible to Deploy OpenShift
Up until now, everything we've done has been basic Linux and KVM tasks - with a little bit of Ansible sprinkled in to make it easy and consistent. Hopefully you've learned a bit so far if you're not super familiar with Linux.

In these next steps, we're going to configure Ansible to be ready to deploy OpenShift in the next set of lab lessons - but we need to get what's called a "pull secret" from Red Hat in order to deploy OpenShift. 

This secret is sensitive information, and we don't want to simply store it in plaintext on our KVM node that is running Ansible - so we're going to create an encrypted vault file for Ansible to store it along with our SSH public key.

Open the Cockpit web interface on your `fedora-kvm` host, and navigate to the Terminal. To create your encrypted vault file, type:
```
ansible-vault create ~/ansible/sens_vars.yml
```
Ansible will ask you for a vault password - remember this, as it's the only way to decrypt the information within the sens_vars.yml file that we are creating to store our pull secret and SSH public key. Enter and confirm your vault password, and you will be dumped into a vi editor.

In another browser tab or window, navigate to `https://console.redhat.com`. Enter your Red Hat Developer username in the "Red Hat login" box and click next, and on the following page enter your password and click the "Log in" button.

If this is your first time logging into the Hybrid Cloud Console and you've just created your Red Hat Developer account for this lab, you may be asked for your phone number, address, and what type of account you are using  - Corporate or Personal. Fill this out and click Submit if you are requested to.

You'll then be greeted by the Red Hat Hybrid Cloud Console - this is your single place for everything Red Hat related when it comes to managing multiple technologies across different clouds or locations!

You should see "Red Hat OpenShift" in the middle top tile - click on "OpenShift ->":
![Viewing the Red Hat Hybrid Cloud Console](/lab-lessons/Lesson-1_base_env_build/images/Cloud_Console_OpenShift.png)

We are going to deploy our own self-managed cluster, so click on the "Red Hat OpenShift Container Platform" tile "Create cluster" button:
![Cloud Console OpenShift Container Platform Cluster Create](/lab-lessons/Lesson-1_base_env_build/images/Cloud_Console_OCP.png)

On the next screen, scroll down to the "Run it yourself" section, and find the "Platform agnostic (x86_64)" link and click on it:
![Selecting the Platform Agnostic Cluster Type](/lab-lessons/Lesson-1_base_env_build/images/Cloud_Console_Run_It_Yourself.png)

Next, click on "Local Agent-based":
![Selecting the Agent-Based Installer](/lab-lessons/Lesson-1_base_env_build/images/Cloud_Console_Local_Agent_Based.png)

On this page, you'll see a section called "Pull secret" - click on the "Copy pull secret" link to copy your pull secret to the clipboard of your machine:
![Copying Your Pull Secret from Cloud Console](/lab-lessons/Lesson-1_base_env_build/images/Cloud_Console_Pull_Secret.png)

That's all we need from the Red Hat Hybrid Cloud Console - navigate back to the empty vault file being edited in the Cockpit interface.

Press the `i` key to get into Insert mode in vi, and type: `ocp_pull_secret:` and then a single space. Type a single quote (`'`), and then paste in your pull secret immediately after the single quote. Once the pull secret is pasted, type another single quote (`'`) and press enter for a new line: 
![Pasting Pull Secret into Ansible Vault File](/lab-lessons/Lesson-1_base_env_build/images/Cloud_Console_Pasting_Pull_Secret.png)

Press escape to exit Insert mode, and type `:wq` followed by enter to write and exit the encrypted file to get back to a shell.

Just to prove that the file is encrypted via Ansible vault, type:
```
cat ~/ansible/sens_vars.yml
```
Sure doesn't look like what you typed in, does it? We're not finished, though - we need to add one more variable to the vault file, which is your SSH public key. In the Cockpit terminal, type:
```
cat ~/.ssh/id_ed25519.pub
```
Highlight the content of this key (beginning with `ssh-ed25519` and ending with `<username>@fedora-kvm.lab.example.com`) and copy it to your clipboard.

Open the Ansible vault file `sens_vars.yml` again by issuing the following command and typing your vault password when prompted:
```
ansible-vault edit ~/ansible/sens_vars.yml
```
Type `i` to get into Insert mode in vi, and type: `ocp_ssh_key:` and then a single space. This time, type a double quote (`"`) and paste your ssh key in immediately after the double quote. Once the ssh key is pasted, type another double quote (`"`) and press enter for a new line:
![Adding SSH Public Key to Ansible Vault File](/lab-lessons/Lesson-1_base_env_build/images/Cloud_Console_Vault_File_Complete.png)

Press escape to exit Insert mode in vi, and then type `:wq` followed by enter to write and exit the encrypted file.

We now have sensitive information that we need to use in the Ansible automation to deploy OpenShift nice and encrypted. Next, let's copy the Ansible role I wrote to deploy OpenShift on libvirt to our local Ansible working directory. 

Change directory to the directory where we previously cloned the git repo for these lab lessons, and use `ansible-galaxy` to install the `ocp-vm-build` role to our `~/ansible/roles` directory:
```
cd ~/working/nested-openshift-sandbox-labs/ansible/roles
```
```
ansible-galaxy role install ocp-vm-build.tar.gz,,ocp-vm-build -p ~/ansible/roles/
```
Finally, just like with our DNS Ansible automation - we need to copy over the playbooks that we can use to deploy the different OpenShift configurations:

```
cp ~/working/nested-openshift-sandbox-labs/ansible/ocp_{sno,tna,compact}_install.yml ~/ansible/
```

### Wrapping Up Lesson One
If you've never used Linux and native KVM heavily in the past, or only played around with it a bit - hopefully I've given you a few new skills - or at least started you out on your journey to using Linux and native KVM. 

Everything you've seen so far is really covering "legacy virtualization" concepts - and we'll get to the cloud native, or "Modern Virtualization" concepts in the next lab lessons.

We've covered the following items across several different areas so far:

- **Installation and configuration of Linux VMs**
	 - Using a standard boot ISO and the anaconda graphical installer
	 - Using a minimal, preconfigured qcow2 image and the console-based customization menu

- **Basic Linux environment setup and administration**
	 - How to install and enable Gnome Desktop UI environment
	 - The absolute basics of using the vi/vim editor
	 - Generating an SSH key for your user account
	 - Copying your SSH key to another system to enable passwordless logins
	 - How to view LVM Volume Groups and Logical Volumes
	 - How to expand a Logical Volume and its underlying filesystem
	 - Enabling the Cockpit web administration interface and installing Application plugins
	 - How to update your Linux system using Cockpit
	 - Using git to clone a repository from GitHub

- **KVM and libvirt basics**
	 - Installing and enabling libvirt
	 - Using the virsh shell to edit the default NAT network in libvirt to allow static IP use

- **Red Hat Developer accounts**
	 - Signing up for a Red Hat developer account
	 - Using podman to login to the Red Hat Image Registry
	 - Logging into the Red Hat Hybrid Cloud Console to get your pull secret for installing OpenShift

- **Ansible Automation**
	 - How to install and configure Ansible Navigator with an execution environment
	 - Creating a basic ansible.cfg file
	 - Creating a basic Ansible inventory file
	 - Installation of roles from tarballs using Ansible Galaxy
	 - Installation of collections using Ansible Galaxy
	 - Creating an Ansible Vault to store sensitive information for use in playbooks
	 - Running Ansible playbooks using Ansible Navigator with Ansible Vault variables
	 
By following this lesson guide, you now have a base KVM host, DNS server, and Ansible environment ready to support deploying a nested Single Node OpenShift, Two Node Arbiter OpenShift, or Compact Three Node OpenShift cluster - all as virtual machines that you can wipe out and redeploy if you mess something up while learning.

From this point, you can kind of "choose your own adventure" in terms of what you'd like to learn next:

- Single Node OpenShift Lab Guide
	 - Uses the fewest resources, but you will not be able to test/learn features like Live Migration (equivalent to vMotion)

- Two Node Arbiter OpenShift Lab Guide
	 - You should definitely have at least 64GB of RAM here - this is useful for learning an alternative to a two-node vSAN cluster with a witness, and uses the 30-day trial of Portworx Enterprise for storage to support Live Migration and two node arbiter configuration

- Three Node Compact Cluster OpenShift Lab Guide
	 - Again, you should definitely have at least 64GB of RAM on your `fedora-kvm` host for this one. This uses three "full" nodes that act as the control plane and worker nodes in OpenShift, and also uses Portworx Enterprise for storage to support Live Migration

Thanks for participating, and I look forward to you building your skills further in future lessons and lab guides!
