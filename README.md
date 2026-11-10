# Linux Server Provisioning

> Built out a mixed-OS server environment from scratch on enterprise virtualization — Windows Server
> 2022, Rocky Linux, and Ubuntu — each with assigned static networking and guest tooling.

*CNIT 340 — Unix/Linux System Administration, Lab 1, Group 063.*

## Summary

Provisioned three VMs on Purdue's Nutanix cluster per an assigned group spec: correct CPU/memory,
custom disk partitioning, static IP/gateway/DNS for each host, Secure Boot + vTPM, and guest tools
(NGT). The foundation layer every other sysadmin and security task sits on top of.

## Environment

- Nutanix AHV cluster (`virtual-lab.cit.lcl`), group `CNIT340G063`
- Windows Server 2022, Rocky Linux 10.2, Ubuntu
- Assigned networking (per group): separate subnets for Windows / Rocky / Ubuntu, shared DNS
  (`44.2.1.44/45`) and time servers (`tick`/`tock.cit.lcl`)

## What I did &amp; found

| Host | Work | Note |
| --- | --- | --- |
| Windows Server 2022 | VM create, VirtIO driver load, NGT, static IP | Desktop Experience install, Secure Boot + vTPM |
| Rocky Linux 10.2 | Minimal install, **custom partitioning** (`/`, `/home`, `/var`, swap), static IP, hostname | Separate `/var` and `/home` partitions limit blast radius of a full disk |
| Ubuntu | VM create, static networking, patching | Rounded out the mixed-OS environment |

Partitioning `/var` and `/home` separately is a small hardening habit: runaway logs or user data
can't fill the root filesystem and take the box down.

## Screenshots

See [`screenshots/`](screenshots/) for the VM configs, partitioning, network settings, and successful
boots.

## Skills demonstrated

Enterprise virtualization (Nutanix) · multi-OS provisioning · Linux installation & partitioning ·
static network configuration · Windows Server setup · guest tooling
