# Ansible for NAT, and Hashicorp Vault EC2 instances

## Problem statement

Your objective is to utilize Terraform (AWS provider only) and Ansible to establish the following infrastructure components. While you're
restricted from using external modules in Terraform, you can leverage external modules for Ansible as needed:

1. Virtual Private Cloud (VPC) – Create a VPC that meets to the following requirements:
   - 3 public subnets spread across different Availability Zones
   - 3 private subnets spread across different Availability Zones
2. Network Address Translation (NAT) Instance – Utilize Ansible to provision a NAT instance.
3. Self-Managed HashiCorp Vault Standalone Instance:

   Use Ansible to establish a standalone HashiCorp Vault instance with the specified attributes:

   - Integrate the DynamoDB backend for secure storage.
   - Implement SSL for secure communication.

## About this repository

This repository contains Ansible playbooks to establish the following tasks:

1. Configure NAT server using `iptables`.
2. Configure standalone HashiCorp Vault instance which integrate the DynamoDB backend for secure storage and implement SSL for secure communication.

You need to completely deployed the AWS infrastructure using the [Terraform repository](https://github.com/gitHub882000/ansible4-hashivault-infra) before moving to this repository.

## Directory structure

The overall directory structure of the project is as follow:

```
inventories/
├─ develop/
│  ├─ ...
├─ production/
│  ├─ ...
├─ staging/
│  ├─ group_vars/
│  │  ├─ server.yaml
│  ├─ host_vars/
│  │  ├─ hashivaultserver.yaml
│  │  ├─ natserver.yaml
├─ hosts
templates/
├─ hashivault.hcl.j2
hashivaultserver.yaml
natserver.yaml
```

1. In `inventories/` directory, `develop/`, `staging/`, and `production/` represents the workload environment. Although only `staging/` has the real Ansible variables, theoretically, each environment directory should have the same structure as follows:

   1. `group_vars/` contains variables of host groups. For example, `server.yaml` contains variables of `server` group.
   2. `host_vars/` contains variables of each host. For example, `hashivaultserver.yaml` contains variables of `hashivaultserver` host.

   The declaration of hosts is presented in `inventories/hosts`.

2. `templates` contains the Jinja2 templates that are leveraged in the playbooks. For example, `hashivault.hcl.j2` is a Jinja2 template for Hashicorp Vault configuration.
3. `hashivaultserver.yaml` and `natserver.yaml` is the playbook to configure Hashicorp Vault server and NAT server respectively.

## Instructions

### Prerequisites

These are the prerequisites to run the codes:

- You have deployed the AWS infrastructure using the [Terraform repository](https://github.com/gitHub882000/ansible4-hashivault-infra) and SSH-ed to the bastion host.

- You then change the current bastion host's working directory into the playbook repository:

```
cd ansible4-hashivault-playbook
```

- You have already taken note of all the private IPs of the 3 EC2 instances: `nat-server` instance, `test-nat-server` instance, and `hashivault` instance.

### How to run

**Note:** All of the following steps should be done on the bastion host, not on the host where your Terraform code was executed.

#### Configure NAT server

1. Confirm that the test-NAT EC2 instance cannot connect to the outside world by SSH-ing to it and ping:

```
ping 8.8.8.8
```

2. Add the private IP address of the NAT server that you took note when you deployed Terraform infrastructure to the file `inventories/staging/host_vars/natserver.yaml`:

```
ansible_host: <natserver-private-ip>
```

3. Run the following command to configure NAT server:

```
ansible-playbook -i inventories/staging/hosts natserver.yaml
```

4. Once the playbook is done running. SSH to the test-NAT EC2 instance and try ping-ing to the Internet. Now the EC2 instance in the private subnet can talk to the outside world via the NAT server:

```
ping 8.8.8.8
```

#### Configure Hashicorp Vault server

1. Add the private IP address of the Hashivault server that you took note when you deployed Terraform infrastructure to the file `inventories/staging/host_vars/hashivaultserver.yaml`:

```
ansible_host: <hashivaultserver-private-ip>
vault_listen_address: "0.0.0.0:8200"
vault_remote_address: "vault.lab.aandd.io:8200"
...
```

3. Run the following command to configure Hashivault server:

```
ansible-playbook -i inventories/staging/hosts hashivaultserver.yaml
```

4. Once the playbook is done running. You can get the root token of the Vault server from the `stdout_lines` of the `Display Vault Unseal Keys and Root Token` task. If you accidentally shutdown the terminal, you can SSH to the Hashivault server and retrieve the root token from the file:

```
/home/ubuntu/.confidential/vault_init.txt
```

5. Now you can access to the Vault server via either the Internet browser or `vault` command line.

## Some considerations

1. The playbook `natserver.yaml` is **idempotent** but `hashivaultserver.yaml` is not. This means `hashivaultserver.yaml` can only be executed once to bootstrap the Vault server onto the EC2 instance. Running the playbook multiple times will not yield the same result.

2. The TLS full chain certificate, TLS private key, unseal keys, and root token of the Vault server, are currently stored in plain text in the directory `/home/ubuntu/.confidential/`. In the future, a more secured vault mechanism should be leveraged.
