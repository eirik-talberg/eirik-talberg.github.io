---
layout: post
title: "Homelab Pt. 1: Creating Ubuntu 22.04 VM templates with Cloud-Init support with Ansible"
categories:
  - homelab
---

Greetings!

In my continuing search for an over-engineered homelab, I have now arrived at the point where I can
configure the creation of virtual machines for our workloads. To achieve this, I will be using Terraform and Ansible
together, and everything will be scripted for reproducibility.

To put it shortly, here are the steps we are going to perform:

1. Prepare the hypervisor for automated VM control
2. Download and configure a cloud image of Ubuntu 22.04
3. Create a VM with Cloud-Init and template it for later use

If you find this stuff interesting, you should maybe check out my previous article
on [VM disk benchmarking]({% post_url 2023-09-14-kvm-disk-benchmarks %}). As always, every bit of code and
configuration (except secret things) is available in
the [homelab repo on GitHub](https://github.com/eirik-talberg/homelab).

# Hypervisor preparations

To start off, we need to do some preparations in our hypervisor. We are
using [Proxmox VE](https://www.proxmox.com/en/proxmox-virtual-environment/overview), but these steps
should be reproducible in other hypervisor platforms as well, check out the relevant documentation.

For Terraform to be able to interface with PVE, we need to set up an API user with the correct permissions. Usually, you
would do this using the UI, but we want to do Infrastructure as Code, so we're gonna automate it
with [Ansible](https://www.ansible.com/).

## Create Terraform API user

This is the Ansible playbook for creating a new API user for Terraform in Proxmox:

[ansible/create-terraform-user.yaml](https://github.com/eirik-talberg/homelab/blob/aa7399575abf45c6ee4784d45e74c5754a3ef093/ansible/create-terraform-user.yaml):
{% raw %}

```yaml
- name: "Create API user"
  hosts:
    - pve-hosts
  vars:
    role_name: terraform
    user_name: terraform
    password: "{{ lookup('ansible.builtin.password', '/tmp/terraform_password') }}"
    privileges:
      - Datastore.AllocateSpace
      - Datastore.Audit
      - Pool.Allocate
      - Sys.Audit
      - Sys.Console
      - Sys.Modify
      - VM.Allocate
      - VM.Audit
      - VM.Clone
      - VM.Config.CDROM
      - VM.Config.Cloudinit
      - VM.Config.CPU
      - VM.Config.Disk
      - VM.Config.HWType
      - VM.Config.Memory
      - VM.Config.Network
      - VM.Config.Options
      - VM.Migrate
      - VM.Monitor
      - VM.PowerMgmt
      - SDN.Use
  tasks:
    - name: "Check if role already exists"
      ansible.builtin.shell:
        cmd: pveum role list --noborder 1 --noheader 1 | grep {{ role_name }}
      register: role_exists
    - name: "Add new role for provisioning users"
      ansible.builtin.shell:
        cmd: pveum role add {{ role_name }}
      when: not role_exists
    - name: "Set privileges for role"
      ansible.builtin.shell:
        cmd: pveum role modify {{ role_name }} -privs "{{ privileges | join(' ')  }}"
    - name: "Check if user already exists"
      ansible.builtin.shell:
        cmd: pveum user list --noborder 1 --noheader 1 | grep {{ user_name }}
      register: user_exists
    - name: "Add user"
      ansible.builtin.shell:
        cmd: pveum user add {{ user_name }}@pve --password {{ password }}
      when: not user_exists
    - name: "Add role to user"
      ansible.builtin.shell:
        cmd: pveum aclmod / -user {{ user_name }}@pve -role {{ role_name }}
```

{% endraw %}

Here's what's actually going down here:

1. We're checking for an existing role, because if we try to add a duplicate, `pveum` errors out
    * If the role doesn't exist, we make it
2. Set the list of privileges for the role. The list was based on the official docs and some trial and error
3. Check if the user with that name already exists, if not we create it with a throwaway password
4. Add the user to the given role

Some considerations:

* As it stands, it is not very configurable. It should probably be moved to a role in Ansible, and made the
  username/group configurable. However, that's not really needed at the moment, since we only need this user to
  interface with PVE.
* The password is a generated one, that we throw away. This user is only supposed to use access tokens anyway.
* Using the `cmd` directive is not really considered best practise for Ansible, but it'll do for now. I'm going to dig
  around for a possible proper Ansible module in Galaxy later (hah).
* Fetching the actual access token is done through the Web UI in Proxmox - I should find a way to automate it.

# Download and configure Ubuntu 22.04 cloud-ready image

Since we don't want to manually perform the installation on each separate VM, we are going to download some cloud-ready
images that Ubuntu themselves helpfully supply. Keeping true to our method, there's a playbook for that too!

[ansible/prepare-cloud-init-images.yaml](https://github.com/eirik-talberg/homelab/blob/aa7399575abf45c6ee4784d45e74c5754a3ef093/ansible/prepare-cloud-init-images.yaml):

```yaml
- name: Create CloudInit image for KVM hosts
  hosts:
    - localhost
  tasks:
    - name: "Check if download is necessary"
      ansible.builtin.stat:
        path: /tmp/jammy-server-cloudimg-amd64.img
      register: image_status
    - name: "Download current cloud image"
      ansible.builtin.get_url:
        url: https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img
        dest: /tmp/
      when: not image_status.stat.exists
    - name: "Install qemu-guest-agent on cloud image"
      ansible.builtin.shell:
        cmd: sudo virt-customize -a /tmp/jammy-server-cloudimg-amd64.img --install qemu-guest-agent

```

Most of this is self-explanatory, but the important part is the final step there. To ensure proper integrations and
communication between PVE and the guest, we need to ensure that `qemu-guest-agent` is available on the guest. If we
don't add this to the image, we will have to manually install it on each guest afterward. Since Terraform is
declarative, this would create a diff between the real VM state and the declared one in Terraform, causing
the `qemu-guest-agent` to be disabled. Enabling it allows PVE to monitor and control the guest properly.

# Create a VM template in Proxmox to clone with Terraform

Next, we need to create a VM template in Proxmox, that can be cloned later on. The VM ID needs to be some value that is
not likely to be used, since they are unique.

[ansible/prepare-cloud-init-images.yaml](https://github.com/eirik-talberg/homelab/blob/aa7399575abf45c6ee4784d45e74c5754a3ef093/ansible/prepare-cloud-init-images.yaml):

{% raw %}

```yaml
- name: "Copy image to PVE hosts"
  hosts:
    - pve-hosts
  vars:
    vm_id: 9999
    memory: 2048
    storage_pool: local-btrfs
    template_name: ubuntu-2204-cloud-init
  tasks:
    - name: "Copy cloud image"
      ansible.builtin.copy:
        src: /tmp/jammy-server-cloudimg-amd64.img
        dest: /tmp/jammy-server-cloudimg-amd64.img
    - name: "Delete old VM or template"
      ansible.builtin.shell:
        cmd: qm destroy {{ vm_id }}
      ignore_errors: true
    - name: "Create VM for template"
      ansible.builtin.shell:
        cmd: qm create {{ vm_id }} --name {{ template_name }} --memory {{ memory }} --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-pci
    - name: "Import cloud image to VM"
      ansible.builtin.shell:
        cmd: qm set {{ vm_id }} --scsi0 {{ storage_pool }}:0,import-from=/tmp/jammy-server-cloudimg-amd64.img
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

{% endraw %}

To create a template for further use, we must first create a VM in the traditional way. A template in Proxmox is based
on an existing VM. We create the VM using the `qm` API within Proxmox. The VM itself could probably have been created
with Terraform (although each run would then recreate the VM, since the final step of creating a template from the VM
is destructive and non-repeatable).

This process can be made more configurable, and can be repeated in a loop per image/OS type if you make the image a
variable. I might do that later.

# Conclusion

There we have it. This step is only really needed once per VM type/version, and you can reuse this template many times.
Overall, there are definite improvements to be made to the Ansible playbooks, but it works as it is for my use case.

Check out the links below if you want to know more about the underlying technologies and APIs in play here.

# Resources and references

* [GitHub repo](https://github.com/eirik-talberg/homelab)
* [Getting started with Ansible](https://docs.ansible.com/ansible/latest/getting_started/index.html)
* [Ansible playbooks](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_intro.html)
* [Terraform Proxmox Provider](https://registry.terraform.io/providers/Telmate/proxmox/latest/docs)
* [Proxmox User Management](https://pve.proxmox.com/wiki/User_Management)
* [Proxmox KVM Virtual Machines](https://pve.proxmox.com/wiki/Qemu/KVM_Virtual_Machines)
* [Ubuntu Cloud images](https://cloud-images.ubuntu.com/)
