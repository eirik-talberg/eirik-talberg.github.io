---
layout: post
title: "Homelab Pt. 2: Kubernetes sandbox cluster creation and configuration with Ansible and ArgoCD"
categories:
  - homelab
---

* Referere til forrige artikkel
* Cluster-typer og arkitektur
* IaC-prinsipper
* Installere k3s med ansible
* Konfigurere ArgoCD med app of apps pattern for deklarativ konfigurasjon
* Cluster-infrastruktur og fellesfunksjonalitet

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

## Cloud-Init

Instead of describing it with my own words, here's an excerpt from
the [official site](https://cloudinit.readthedocs.io/en/latest/):
> Cloud-init is the industry standard multi-distribution method for cross-platform cloud instance initialisation. It is
> supported across all major public cloud providers, provisioning systems for private cloud infrastructure, and
> bare-metal
> installations.
>
> During boot, cloud-init identifies the cloud it is running on and initialises the system accordingly. Cloud instances
> will automatically be provisioned during first boot with networking, storage, SSH keys, packages and various other
> system aspects already configured.

While we won't be 