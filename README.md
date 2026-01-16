This project is meant to help deploy learning environments for OpenShift and OpenShift virtualization - specifically targeted towards vSphere users who may not have deep Linux, Kubernetes, or KubeVirt experience.

To understand the background on this project, please read my [introductory LinkedIn post](https://www.linkedin.com/pulse/nested-openshift-sandbox-labs-introduction-tim-darnell-ea29c/).

Contributions/corrections are always welcome!

## How To Get Started  
These labs are meant to be run as nested environments. So, to get started to build out a base KVM hypervisor VM - you'll need to start with lab lesson one:  

[Lab Lesson One - Base Environment Build](/lab-lessons/Lesson-1_base_env_build/lab-procedures.md)  

![Lesson One Environment Diagram](/lab-lessons/Lesson-1_base_env_build/images/Fedora-nested-ocp-fedora-kvm.png)  

Once you have your base environment built, it gets pretty simple to deploy a handful of different OpenShift environments to play with using Ansible:  

### Single Node OpenShift with OpenShift Virtualization
[Lab Lesson Two - Single Node OpenShift](/lab-lessons/Lesson-2_sno_deploy/lab-procedures.md)  
- Single Node OpenShift (SNO) Deployment
- LVM Storage Operator
- NMState OVS Networking for VMs
- HyperConverged Operator (OpenShift Virtualization)  

![Lesson Two Environment Diagram](/lab-lessons/Lesson-2_sno_deploy/images/Fedora-nested-ocp-sno.png)  

### Two Node Arbiter OpenShift with OpenShift Virtualization  
[Lab Lesson Three - Two Node Arbiter](/lab-lessons/Lesson-2_sno_deploy/lab-procedures.md)  
- Two Node Arbiter OpenShift (TNA) Deployment
- Portworx Enterprise TNA Configuration
- NMState OVS Networking for VMs
- HyperConverged Operator (OpenShift Virtualization)  

![Lesson Three Environment Diagram](/lab-lessons/Lesson-3_tna_deploy/images/Fedora-nested-ocp-tna.png)  

### Three Node Compact OpenShift with OpenShift Virtualization  
*Coming Soon!*
