---
layout: post
title: "Homelab Pt. 1: Creating Ubuntu 22.04 VM templates with Cloud-Init support with Ansible"
description: Using automation to simplify boring things
categories:
  - homelab
---

> **EDITOR'S NOTE**
>
> This article has been rewritten and edited significantly. The first version was, quite frankly, subpar and lacked both
> structure and a good flow. I also completely glossed over what these tools are well suited for and why I like them.
> I've tried to remedy it. Some important things to note, is that I've completely removed the section concerning
> creating the API user in Proxmox for Terraform to use, as it fell out of scope and will be adressed in the next
> article.
>
> On to the show.

---

## Introduction

Greetings!

In my continuing search for an over-engineered homelab, I have now arrived at the point where I can
configure the creation of virtual machines for our workloads. If you find this stuff interesting, you should maybe check
out my previous article
on [VM disk benchmarking]({% post_url 2023-09-14-kvm-disk-benchmarks %}). As always, every bit of code and
configuration (except secret things) is available in
the [homelab repo on GitHub](https://github.com/eirik-talberg/homelab).

### Goals and motivations

So why are we going through all this hassle? Couldn't you just create each VM through the Web UI and install Ubuntu like
normal people? Well, for one, I'm lazy. If we consider each environment to need at least four (4) VMs (1x master, 3x
agents), I'd have to do the same thing, manually, four times **per cluster**, and ain't nobody got time for that. Also,
I subscribe to the notion that infrastructure should
be [treated as cattle, not pets](https://cloudscaling.com/blog/cloud-computing/the-history-of-pets-vs-cattle/).

In that spirit, we want everything to be described using Configuration/Infrastructure as Code, so we can easily
provision, deploy and kill to our heart's content. Ideally, I want absolutely everything to be configured this way,
but certain parts of bare-metal infrastructure isn't really up to it. For instance, pfSense, my router software, seems
not to be well supported by neither Ansible nor Terraform. The same seems to be the case with unRAID, which I use for
mass storage.

### What's in the toolbox

To achieve our goals, we're going to be leaning heavily on [Ansible](https://www.ansible.com/), which is a Python
application, now owned by Red Hat, but avaialble generally as a Python package.

#### Ansible with Poetry

Personally, I love Ansible. I use it all the time for scripts and setups that I need to be repeatable, and I find it to
be way easier to maintain and update than a loose collection of Bash scripts or similar solutions. If you find yourself
often repeating certain things, Ansible can be really low-entry way to simplify it. I've even used it as a stand-in for
a proper build/pipeline solution where old client environments have limited my options.

[Poetry](https://python-poetry.org/) is a way to handle dependency management in Python. I'm old and remember the old
ways of dealing with dependencies with [PIP](https://pypi.org/project/pip/) and manual requirements files, which were
flat TXT files with lists of libraries and respective versions. They had no descriptions of transitive dependencies, and
updating the files themselves were difficult and tended to break things. It gets messy quickly, and I've found Poetry to
be a nice way to handle both Python interpreter and library/dependency management, with a good CLI and sensible
configuration files.

### Architecture

This quick chart (made with [mermaid, btw!](https://mermaid.js.org/intro/)) shows the relevant machines that will be
involved.

```mermaid!
flowchart LR

    A[macos-14.0@macbook-air] -->|SSH| B(ubuntu-22.04@ws-01)
    A -->|SSH| C(pve-8.0@pve-01)
```

My laptop acts as the controller node where Ansible runs, and it talks to a Ubuntu 22.04 workstation VM (I'll elaborate
on this later) and the Proxmox Hypervisor node over SSH.

## Ansible Playbook

The operations are collected in a Playbook (a collection of Ansible tasks and roles).

{% raw %}

```yaml
- name: Download and configure Ubuntu cloud image
  hosts:
    - ws-01
  run_once: true
  roles:
    - ubuntu-cloud-image
- name: Copy image to PVE
  hosts:
    - localhost
  vars:
    image_from: "{{ hostvars['ws-01']['ansible_user'] }}@{{ hostvars['ws-01']['ansible_host'] }}:{{ cloud_image_location }}"
    image_to: "{{ hostvars['pve-01']['ansible_user'] }}@{{ hostvars['pve-01']['ansible_host'] }}:{{ cloud_image_location }}"
  tasks:
    - name: Copy image
      ansible.builtin.command:
        cmd: scp {{ image_from }} {{ image_to }}
- name: Create PVE VM template
  hosts:
    - pve-01
  run_once: true
  roles:
    - pve-vm-template
```

{% endraw %}

The major parts are described below, the middle stage is a simple workaround to move the finished image between the
Ubuntu workstation and Proxmox.

### Download and configure Ubuntu cloud image

If you want to, you can build your own VM from scratch in Proxmox, and create a template from it. I don't think that's
really necessary for most use cases, but some esoteric setup could require it. Since we can use any standard
installation of Ubuntu Server as our base, we will download Ubuntu's official cloud images.

[ansible/roles/ubuntu-cloud-image/tasks/main.yaml](https://github.com/eirik-talberg/homelab/blob/b3d8db228375a4f909d3b38a66708d628f172b7f/ansible/roles/ubuntu-cloud-image/tasks/main.yaml)
{% raw %}

```yaml
- name: "Check if download is necessary"
  ansible.builtin.stat:
    path: "{{ cloud_image_location }}"
  register: image_status
- name: "Download current cloud image"
  ansible.builtin.get_url:
    url: "{{ download_url }}"
    dest: "{{ cloud_image_location }}"
  when: not image_status.stat.exists
- name: "Install qemu-guest-agent on cloud image"
  ansible.builtin.shell:
    cmd: sudo virt-customize -a {{ cloud_image_location }} --install qemu-guest-agent
  become: yes
- name: "Enable qemu-guest-agent service on cloud image"
  ansible.builtin.shell:
    cmd: sudo virt-customize -a {{ cloud_image_location }} --firstboot-command 'sudo service qemu-guest-agent start'
  become: yes
```

You'll notice some variable references here, they are declared in variables files local to the inventory and the role:

```yaml
download_url: https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img
cloud_image_location: /tmp/ubuntu.img
```

{% endraw %}

There's an important caveat here: Using virt-customize must be done on a machine from the same distro family as the
target, which is why I have a separate Ubuntu VM from before (running on my unRAID box) that will handle downloading and
configuring the image. The only modifications we need to do is installing and enabling `qemu-guest-agent`.

#### What is QEMU Guest Agent?

[QEMU Guest Agent](https://pve.proxmox.com/wiki/Qemu-guest-agent) is described pretty well on the official Proxmox Wiki.
It helps Proxmox monitor the state and condition of our VMs. If we don't add this to the image, we will have to manually
install it on each guest afterward. Since Terraform is declarative, this would create a diff between the real VM state
and the declared one in Terraform, causing the `qemu-guest-agent` to be disabled. Enabling it allows PVE to monitor and
control the guest properly.

### Create a VM template in Proxmox to clone with Terraform

Based on the image we created in the previous step, we will create a VM in Proxmox, configure it, and create a template
from it. A VM template in Proxmox can be cloned fully to create new VMs many times, and is a very handy way to create a
common setup. This could be done with many different distros, I just prefer Ubuntu.

* Basert på imaget vi har laget, lager vi først en VM, som vi så lager template fra.
* Nevne hva storage_pool, osv betyr
* Bytte ut kode

Next, we need to create a VM template in Proxmox, that can be cloned later on. The VM ID needs to be some value that is
not likely to be used, since they are unique.

[ansible/roles/pve-vm-template/tasks/main.yaml](https://github.com/eirik-talberg/homelab/blob/b3d8db228375a4f909d3b38a66708d628f172b7f/ansible/roles/pve-vm-template/tasks/main.yaml):

{% raw %}

```yaml
- name: "Delete old VM or template"
  ansible.builtin.shell:
    cmd: qm destroy {{ vm_id }}
  ignore_errors: true
- name: "Create VM for template"
  ansible.builtin.shell:
    cmd: qm create {{ vm_id }} --name {{ template_name }} --memory {{ memory }} --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-pci
- name: "Import cloud image to VM"
  ansible.builtin.shell:
    cmd: qm set {{ vm_id }} --scsi0 {{ storage_pool }}:0,import-from={{ cloud_image_location }}
- name: "Set boot order for faster boot"
  ansible.builtin.shell:
    cmd: qm set {{ vm_id }} --boot order=scsi0
- name: "Set VGA output"
  ansible.builtin.shell:
    cmd: qm set {{ vm_id }} --serial0 socket --vga serial0
- name: "Create template from VM"
  ansible.builtin.shell:
    cmd: qm template {{ vm_id }}
```

Again, some variables that are in use:

```
vm_id: 9999
memory: 2048
storage_pool: local-btrfs
template_name: ubuntu-2204-cloud-init
```

{% endraw %}

The VM ID can be any number, just pick one that is not likely to create collisions with your VM structure (all VMs in
Proxmox have a unique ID). The `storage_pool` variable is unique to my setup, and since I have Proxmox installed on a
mirrored BTRFS array, the image storage is configured to be `local-btrfs`. Memory is set to something sensible, but is
likely to be configured on a per VM basis anyway. `template_name` is the name I give to the template for further reuse (
we will reference this name when cloning the VMs).

`qm` is the API inside Proxmox to manage VMs. The VM itself could probably have been created
with Terraform (although each run would then recreate the VM, since the final step of creating a template from the VM
is destructive and non-repeatable).

This process can be made more configurable, and can be repeated in a loop per image/OS type if you make the image a
variable. I might do that later.

## Conclusion

There we have it. This step is only really needed once per VM type/version, and you can reuse this template many times.
Overall, there are definite improvements to be made to the Ansible playbooks, but it works as it is for my use case.

Check out the links below if you want to know more about the underlying technologies and APIs in play here.

### Resources and references

* [GitHub repo](https://github.com/eirik-talberg/homelab)
* [Getting started with Ansible](https://docs.ansible.com/ansible/latest/getting_started/index.html)
* [Ansible playbooks](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_intro.html)
* [Proxmox User Management](https://pve.proxmox.com/wiki/User_Management)
* [Proxmox KVM Virtual Machines](https://pve.proxmox.com/wiki/Qemu/KVM_Virtual_Machines)
* [QEMU Guest agent](https://pve.proxmox.com/wiki/Qemu-guest-agent)
* [Ubuntu Cloud images](https://cloud-images.ubuntu.com/)