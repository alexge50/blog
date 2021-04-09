Title: The journey of my "cloud" infrastructure; What I learned about terraform
Date: 2021-04-09 23:58
Category: servers-stuff
Tags: devops, virtualization
Summary: In this article I will present my self hosting journey and go in depth about my latest configuration of the server, which now uses terraform and kvm.

# Proceeding with caution is advised
This article purely reflects my opinions and should not be used to draw conclusions about the benefits and disadvantages
of the technologies that will be mentioned. By no means the information and methodologies presented in this article are
at a professional level, as such the article might contain mild to vast amounts of hacks and bodging. Do proceed with care. 

# My journey in self-hosted world
## Prelude
My journey in the self-hosted world has started in 2016, after having a summer job, I got enough money to buy a second hand 
server - which at that time I had been planning for a while. Not knowing better, the first hypervisor software I had used 
was Proxmox. However, I had only used Proxmox as means to create containers, and not virtual machines. For a long while,
Proxmox has been good enough for my needs. As I was getting into the habit of keeping my computer with the latest and greatest
kernel version, I had realised that the Debian - the distribution on which Proxmox is based - was not cutting it.

In between not finding a great way to perform backups for the container images on Proxmox and my burning desire to run 
Linux kernel version 5, I decided to switch to Fedora in combination with [LXD (container management software)](https://linuxcontainers.org/lxd/introduction/).

## update.sh
During this whole time, I would manage all the containers and software manually, which was increasingly becoming more
and more tedious as I wanted to perform updates more frequently. 

As I didn't discover Ansible and other automation software yet, I did what every programmer would do - turn to scripts
to solve the issue. `update.sh` was the name of the script I used to run in order to update my system. 
The script itself was just executing the package manager update command in each container, by using LXD's `lxc exec` command:

```sh
lxc exec container -- dnf update
# ... 
```

## The dawn of DevOps era

In 2020, I had my final exams for high school and I had to study for them, during the period before the exams my server
was sitting turned off waiting for me to take care of it. At the end of the summer, I had decided to take a break 
from the summer break to configure my servers a bit more seriously, as I was about to leave for university. 
I finally stopped procrastinating and sleeping on Ansible and a more DevOps flow for my server.

During this week, I wrote ansible yaml files to configure my "infrastructure" - the networking interfaces on the server
and the containers, with properties (the networks the container is attached to, number of cores, ram size, disk storage quote)
registered in the inventory file. On top of that, I wrote roles for a range of other things, including ssh servers,
`firewalld` configurations, reverse proxy (I decided writing my own instead of using an already written one).

Everything was fine with my ansible until...

## BTRFS disaster and the new age of Terraform
After a power outage at the location of the server, the BTRFS pool I was using for the container storage was left corrupt,
with no way for me to recover it. This unfortunate corruption prompted me to make a change I wanted to do way before.

As I realized the Ansible role I created for creating containers wasn't cutting it and that LXD had some funky things, 
I decided to instead become even more professional and use what the big boys use - Terraform. However, instead of
using OpenStack or any other cloud provider, I wanted to use a KVM provider. While containers are great technology,
working with LXD didn't feel like it was an established piece of software (the fact that it's only available on 
snapcraft in many distributions doesn't help it either). Add the awkward configuration required to add remotes (I 
was planning on adding the ansible playbooks in a CI book all along) was not cutting it for me. As I was researching 
the options I would have to connect to the LXD containers from the CI, I had realized that I am better off with libvirt's
over ssh remoting. Not to mention the very great `virt-manager` tool, that shows my vms as a very nice list.

In between doing my coursework, I had spent some time playing with libvirt and terraform on my testing server, and when 
I became confident in my terraform code, I decided to roll the terraform configurations out to my "production" server.
Everything went almost smoothly, except some cloudinit oddities, and some other configuration bugs I had in my
ansible playbooks.

And... this is the present day, now my server is working fine, all containers were replaced with vms, the RAM consumption
sky-rocketed - but I have enough RAM on the trusty second hand old server I bought - and everything in terms of VM 
creation is managed by Terraform. I will take the reminder of this post show you, dear reader, what I learned about
Terraform, in the small amount of time I have been using it.

# What I learned about Terraform

***This section is not supposed to be a tutorial in and on itself, but rather an extension of tutorials on getting
started with terraform and the libvirt provider.***

[Terraform]() is an ["Infrastructure as Code"]() software tool. In layman's terms, Infrastructure as Code means
creating/managing infrastructure via means of a configuration language. These configuration languages, as far as I am aware,
are usually descriptive languages, in this case being Terraform itself. In my case, what I needed from is a way
for me to "declare" vms with a limited amount of configurability - CPU cores count, RAM, the network bridge the vm
is attached to, the ip address or dhcp and the distribution of linux to be used. Since I am using KVM, I have decided
to use the [libvirt provider](https://github.com/dmacvicar/terraform-provider-libvirt) (providers are akin to modules,
though they 'provide' resource types that can be defined).

A vm can be simply defined as a resource the following way:

```tf
resource "libvirt_domain" "vm-test" {
    name   = "vm-test"
    memory = "1024"
    vcpu   = 1

    cloudinit = libvirt_cloudinit_disk.commoninit.id

    network_interface {
        network_name = "default"
    }

    console {
        type = "pty"
        target_port = "0"
        target_type = "serial"
    }

    console {
        type = "pty"
        target_type = "virtio"
        target_port = "1"
    }

    disk {
        volume_id = libvirt_volume.fedora-qcow2.id
    }

    graphics {
        type = "spice"
        listen_type = "address"
        autoport = true
    }
}
```

The block of code covers defining a resource of type `libvirt_domain` - domain being a vm, with the name of the **resource**
being "vm-test" and the name of the vm itself being `vm-test`. I was a bit confused at first about the difference
between the resource itself and the domain, however, as the language allows "loops", one resource can keep track of
multiple domains. 

While the other options to set the vm up are rather clear, one noteworthy thing is the ability to reference other resources,
in this case the `cloudinit` image and the disk image are referenced from other terraform resources. The libvirt provider
allows the user to create several types of resources: pools (where the disk images are stored), disks, cloudinit disks,
networks and domains.

## Loops
Terraform's looping mechanism will be the feature that I will use to achieve templating of the vm configuration. 
The way that looping works is kind of peculiar, however, not very different compared to Ansible.

In order to loop over an object, the terraform "abstract" resource itself provides a special option, called `for_each`,
which can be used as follows:

```tf
resource "libvirt_domain" "domain-vms" {
    for_each = {for vm in var.vms: vm.name => vm}

    name = each.value.name
    memory = each.value.memory
    vcpu = each.value.cpu

    ...
}
```

However there is a catch, considering how Terraform calculates the state of the resource itself and the changes it has 
to make to the infrastructure, the only objects that can be looped over are basically sets and dictionaries. 
In the previous example, the variable `var.vms` itself is a list, however, the construct 
`{for vm in var.vms: vm.name => vm}` loops over the list and creates a dictionary with pairs of a `vm.name` as a key and
the `vm` object itself as value.

There are other ways to achieve iteration over a list, such as the [toset](https://www.terraform.io/docs/language/functions/toset.html)
function.

## Input Variables
Input Variables are properties to a Terraform file that are defined in a different file that can be either a `.tfvars` file
or a `json` file. These variables are special because each used variable needs to be declared by type. The declaration
of the `var.vms` variable is as follows:
```tf
variable "vms" {
    type=list(
        object({
        name = string
        distro = string
        cpu = number
        memory = string
        storage = number
        networks = list(
            object({
                name=string,
                ip-address=string,
                gateway=string
            })
        ),
        })
    )
}
```

A `tfvars` file looks as follows:
```tf
vms = [
    {
        "name": "test",
        "distro": "base-fedora33",
        "cpu": 1,
        "memory": "1024",
        "storage": 5,
        "networks": [{
            "name": "test",
            "ip-address": "",
            "gateway": ""
        }],
    },
```

In my case, an empty ip-address means that the ip is dynamically allocated.

The interesting aspect of Input Variables is that input checking can be done in the variable declaration block, however
checking which I have not done.

## cloud-init
[cloud-init](https://cloudinit.readthedocs.io/en/latest/) is a mechanism provided by Linux distributions in cloud images
to be able to configure the image in a first boot, without requiring user intervention. The way that cloud images are
set up in Terraform is by first declaring a `data` object that is going to be a generated template from the cloud-init 
configuration files:

```tf
data "template_file" "user_data" {
    for_each = {for vm in var.vms: vm.name => vm}

    template = templatefile("${path.module}/cloudinit/cloud_init.cfg", {name = each.value.name, ssh_keys=var.authorized-ssh-keys})
}

data "template_file" "network_config" {
    for_each = local.vms_networks

    template = templatefile("${path.module}/cloudinit/network.cfg", {networks = each.value})
}
```

After which the cloud-init image resource itself is declared:

```tf
resource "libvirt_cloudinit_disk" "commoninit" {
    for_each = {for vm in var.vms: vm.name => vm}

    name = join("-", [each.value.name, "commoninit.iso"])
    user_data = data.template_file.user_data[each.value.name].rendered
    network_config = data.template_file.network_config[each.value.name].rendered
    pool = "vm-pool"
}
```

The cloud-init I am using are looking as follows:

`cloud-init.cfg`

```yaml
#cloud-config

preserve_hostname: true
hostname: ${name}
runcmd:
 - [ ls, -l, / ]
 - [ sh, -xc, "echo $(date) ': hello world!'" ]
 - [ hostnamectl, set-hostname, "${name}" ]
ssh_pwauth: 0
disable_root: 1
users:
  - name: alex
    sudo: ALL=(ALL) NOPASSWD:ALL
    groups: users, admin
    home: /home/alex
    shell: /bin/bash
    lock_passwd: true
    ssh-authorized-keys:
      %{for key in ssh_keys ~}
- ${key}
      %{ endfor ~}
      
final_message: "The system is finally up, after $UPTIME seconds"
```

This is the main config file, and it is mainly concerned about setting up the hostname and the admin user with the ssh keys.
However, in my quest, I have discovered a few gotchas with cloud-init:
- the first line of the `cloud_init.cfg` file has to be `#cloud-config`
- On the Fedora image, in particular, the module for setting up the hostname did not appear to work, as such,
I had to resort to running a `hostnamectl` command to change the hostname of the vm.
  
The `network.cfg` file is a yaml file and looks as follows:
```yaml
version: 2
ethernets:
  %{ for index, network in networks ~}
eth${index}:
    match:
      macaddress: ${network.mac-address}
%{if network.ip-address == "" ~}
    dhcp4: true
%{else ~}
    addresses:
      - ${network.ip-address}/24
    gateway4: ${network.gateway}
    nameservers: 
        addresses: [${network.gateway}]
%{endif ~}
  %{ endfor ~}
```
This file is concerned about setting up the static IP address or a DHCP configuration for each network interface attached
to the vm - more details are covered in the networking section.

## Networking
My network is managed by a [pfSense](https://www.pfsense.org/) router, where I have created VLANs to separate the vms, 
by their job. On the server, a bridge for each VLAN is created then each vm is attached to the aforementioned bridge. As
each VM can be attached to multiple VLANs, it is mandatory to create the configuration system to be able to manage multiple
networking interface for each vm, each of the interface being configured with either a static ip or a dynamic ip. 

Because I was not able to get the networking configuration of cloud-init to work nicely with trying to match the network
cards by order, I have decided to use a [mac address generator provider](https://github.com/ivoronin/terraform-provider-macaddress)
and assign each network interface with a well known mac address I can use in the cloud-init configuration.

One of the main hurdles I came across with the networking configuration is adding a dynamic number of interfaces to the domain,
which can be done using the `dynamic` block as follows:

```tf
resource "libvirt_cloudinit_disk" "commoninit" {
    for_each = {for vm in var.vms: vm.name => vm}
    
    ...
    
    dynamic "network_interface" {
        for_each = local.vms_networks[each.value.name]
            # local.vms_network is a variable that contains the network interface options (ip, the bridge it is attached to) 
            # and the generated MAC address.
        content {
            bridge = join("-", [network_interface.value["name"], "br"])
            mac = network_interface.value["mac-address"]
            hostname = each.value.name
        }
  }
  
  ...
}
```

The bridge networks are configured to the host through NetworkManager, and they are defined to libvirt manually, as I have
not managed to move the definitions in the terraform file yet. 

# The Sequel
The saga is not done yet, it rarely is. One of the biggest improvements I want to bring to my small "cloud" is to make
it actually be managed entirely by Continuous Integration. Until now, I have done everything to move into that direction,
however the main challenge will be creating a good backuping plan, to be able to backup everything before anything is
changed by the Continuous Integration.

As this is one of the biggest next steps, there are lots of small steps that I want to implement - one of them being
a DNS server maintained by Ansible. The reason why I need DHCP on some of the hosts is that it is the only way
these hosts can be registered in pfSense's DNS server. However, by swapping out all dynamic IPs for static ones
and creating a DNS server pfSense can forward my local domains too, I will not need DHCP anymore, which overall would make
management easier.