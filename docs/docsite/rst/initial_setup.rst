==================================
Deploying YDB cluster with Ansible
==================================

This guide outlines the process of deploying a YDB cluster on a group of servers using Ansible. The recommended setup to get started is 3 servers with 3 disk drives for user data each. For reliability purposes each server should have as independent infrastructure as possible: they'd better be each in a separate datacenter or availability zone, or at least in different server racks.

For large-scale setups, it is recommended to use at least 9 servers for highly available clusters (mirror-3-dc) or 8 servers for single-datacenter clusters (block-4-2). In these cases, servers can have only one disk drive for user data each, but they'd better have an additional small drive for the operating system. You can learn about redundancy models available in YDB from the YDB cluster topology article. During operation, the cluster can be expanded without suspending user access to the databases.

Note::

    Recommended server requirements:

    - 16 CPUs (calculated based on the utilization of 8 CPUs by the storage node and 8 CPUs by the dynamic node).
    - 16 GB RAM (recommended minimum RAM).
    - Additional SSD drives for data, at least 120 GB each.
    - SSH access.
    - Network connectivity between machines in the cluster.
    - OS: Ubuntu 20+, Debian 10+
    - Internet access is needed to update repositories and download necessary packages.

Download the GitHub repository with examples for installing YDB cluster – git clone https://github.com/ydb-platform/ydb-ansible-examples.git. This repository contains a few installation templates for deploying YDB clusters in subfolders, as well as scripts for generating TLS certificates and requirement files for installing necessary Python packages. In this article, we'll use the 3-nodes-mirror-3-dc subfolder for the most simple setup. Alternatively, you can similarly use 8-nodes-block-4-2 or 9-nodes-mirror-3-dc if you have the necessary number of suitable servers.

Repository structure::

    ├── 3-nodes-mirror-3-d-v2 / 9-nodes-mirror-3-dc-v2 / 8-nodes-block-4-2-v2
    │    ├── ansible.cfg # An Ansible configuration file containing settings for connecting to servers and project structure options. It is essential for customizing Ansible's behavior and specifying default settings.
    │    ├── ansible_vault_password_file # A file containing the password for decrypting encrypted data with Ansible Vault, such as sensitive variables or configuration details. This is crucial for securely managing secrets like the root user password.
    │    ├── creds # A directory for environment variables that specify the username and password for YDB, facilitating secure access to the database.
    │    ├── files
    │    │    ├── config.yaml # A YDB configuration file, which contains settings for the database instances.
    │    └── inventory # A directory containing inventory files, which list and organize the servers Ansible will manage.
    │         ├── ydb_inventory.yaml # The main inventory file, specifying the hosts and groups for Ansible tasks.
    │         └── group_vars/ydb  # A directory containing variables for configuring hosts with YDB cluster
    │              ├── all.yaml   # Specify YDB version, databases and so on
    │              └── vault.yaml # An encrypted inventory file storing sensitive information, such as the root user's password for YDB, using Ansible Vault.
    ├── README.md # A markdown file providing a description of the repository, including how to use it, prerequisites, and any other relevant information.
    ├── requirements.txt # A file listing Python package dependencies required for the virtual environment, ensuring all necessary tools and libraries are installed.
    ├── requirements.yaml # Specifies the Ansible collections needed, pointing to the latest versions or specific versions required for the project.
    ├── TLS #A directory intended for storing TLS (Transport Layer Security) certificates and keys for secure communication.
    │    ├── ydb-ca-nodes.txt # Contains a list of Fully Qualified Domain Names (FQDNs) of the servers for which TLS certificates will be generated, ensuring secure connections to each node.
    │    └── ydb-ca-update.sh # A script for generating TLS certificates from the ydb-ca-nodes.txt list, automating the process of securing communication within the cluster.

To work with the project on a local (intermediate or installation) machine, you will need: Python 3 version 3.10+ and Ansible core version 2.15.2 or higher. Ansible can be installed and run globally (installed in the system) or in a virtual environment. If Ansible is already installed – you can move on to the step "Configuring the Ansible project"; if Ansible is not yet installed, install it using one of the following methods:

Installing Ansible globally (example for Ubuntu 22.04 LTS)::

    - Update the apt package list with sudo apt-get update.
    - Upgrade packages with sudo apt-get upgrade.
    - Install the software-properties-common package to manage your distribution's software sources – sudo apt install software-properties-common.
    - Add a new PPA to apt – sudo add-apt-repository --yes --update ppa:ansible/ansible.
    - Install Ansible – sudo apt-get install ansible-core (note that installing just ansible will lead to an unsuitable outdated version).
    - Check the Ansible core version – ansible --version (it must be between 2.11 and 2.18, 2.19 and newer are not supported yet)

Installing Ansible in a Python virtual environment::

    - Update the apt package list – sudo apt-get update.
    - Install the venv package for Python3 – sudo apt-get install python3-venv
    - Create a directory where the virtual environment will be created and where the playbooks will be downloaded. For example, mkdir venv-ansible.
    - Create a Python virtual environment – python3 -m venv venv-ansible.
    - Activate the virtual environment – source venv-ansible/bin/activate. All further actions with Ansible are performed inside the virtual environment. You can exit it with the command deactivate.
    - Install the recommended version of Ansible using the command pip3 install -r requirements.txt, while in the root directory of the downloaded repository.
    - Check the Ansible core version – ansible --version (it must be between 2.11 and 2.18, 2.19 and newer are not supported yet)
  
Navigate to the root directory of the downloaded repository and execute the command ansible-galaxy install -r requirements.yaml – this will download the Ansible collections ydb_platform.ydb and community.general, which contain roles and plugins for installing YDB.

Configure the Ansible project
=============================

Edit the variable files
Regardless of the chosen cluster topology (3-nodes-mirror-3-dc, 9-nodes-mirror-3-dc, or 8-nodes-block-4-2), the main parameters for installing and configuring YDB are contained in the inventory file all.yaml, which is located in the inventory/group_vars/ydb directory.

Connection settings::

    ansible_user: "ubuntu" # specify the user for Ansible to connect via SSH.
    ansible_ssh_common_args: "-o ProxyJump=<ansible_user>@<static-node-1-IP>" # option for connecting Ansible to a server by IP, from which YDB will be installed (including ProxyJump server). It is used when installing YDB from a local machine not included in the private DNS zone.
    ansible_ssh_private_key_file: "~/.ssh/id_rsa" # change the default private SSH-key path to the actual one, usually such files are located in ~/.ssh folder

Sources of YDB executables::

    ydb_version: 24.4.4.12              # automatically download one of the YDB official releases by version number. For example, 24.4.4.12. Also this value is used for the folder name of release on a server (/opt/ydb/release/{ydb_version})
    ydb_archive: files/ydbd-25.1.4.7-linux-amd64.tar.gz # a local filesystem path for a YDB distribution archive downloaded or otherwise prepared in advance.
    ydbd_binary: files/ydbd             # local filesystem path for YDB server (required for installation without Internet access)
    ydb_cli_binary: files/ydb           # local filesystem path for YDB CLI (required for installation without Internet access)
    ydbops_binary: files/ydbops         # local filesystem path for YDB Ops tool (required for installation without Internet access)
    ydb_dstool_binary: files/ydb-dstool # local filesystem path for YDB DSTool (required for installation without Internet access)

Possible combinations::

    # Download YDB version and install default YDB Ops and YDB DSTool
    ydb_version: 24.4.4.12
    
    # Use local archive and binaries for installation
    ydb_archive: files/ydbd-25.1.4.7-linux-amd64.tar.gz
    ydbops_binary: files/ydbops
    ydb_dstool_binary: files/ydb-dstool


Optional changes in the var files
---------------------------------------
Feel free to change these settings if needed, but it is not necessary in straightforward cases::

    ydb_cores_static: 2  # set the number of CPU cores allocated to static nodes.
    ydb_cores_dynamic: 2 # set the number of CPU cores allocated to dynamic nodes.
    ydb_tls_dir: files/certs/ # specify a local path to a folder with TLS certificates prepared in advance. It must contain the ca.crt file and subdirectories with names matching node hostnames, containing certificates for a given node. If omitted, self-signed TLS certificates will be generated automatically for the whole YDB cluster.

    ydb_database_groups: 1 

The optimal value of the ydb_database_groups setting in the vars section depends on available disk drives. Assuming only one database in the cluster, use the following logic::

    For production-grade deployments, use disks with a capacity of over 800 GB and high IOPS, then choose the value for this setting based on the cluster topology:
    For block-4-2, set ydb_database_groups to 95% of your total disk drive count, rounded down.
    For mirror-3-dc, set ydb_database_groups to 84% of your total disk drive count, rounded down.
    For testing YDB on small disks, set ydb_database_groups to 1 regardless of cluster topology.

Changing the root user password
-------------------------------
Next, you can change the standard YDB root user password contained in the encrypted inventory file vault.yaml.
Execute the command, update password and save changes::
    
    ansible-vault edit inventory/group_vars/ydb/vault.yaml

Content of this file::

    ydb_password: password # This password will be used for the default user (root)

For your information, Ansible is using ansible_vault_password_file.txt for encrypting and decrypting vault files. Recreate vault.yaml if you change this password (ansible-vault create inventory/group_vars/ydb/vault.yaml)

Prepare the YDB configuration file
----------------------------------
The YDB configuration file contains the settings for YDB nodes and is located in the subdirectory /files/config.yaml. A detailed description of the configuration file settings for YDB can be found in the article YDB cluster configuration.

The default YDB configuration file already includes almost all the necessary settings for deploying the cluster. You need to replace the standard FQDNs of hosts with the current FQDNs in the hosts and blob_storage_config sections:

- host section::
  
    ...
    hosts:
    - host: static-node-1.ydb-cluster.com
    host_config_id: 1
    walle_location:
        body: 1
        data_center: 'zone-a'
        rack: '1'
    ...

- blob_storage_config section::

    ...
    - fail_domains:
        - vdisk_locations:
        - node_id: static-node-1.ydb-cluster.com
            pdisk_category: SSD
            path: /dev/disk/by-partlabel/ydb_disk_1
    ...

The rest of the sections and settings in the configuration file can remain unchanged.


Deploying the YDB cluster
-------------------------

The repository contains some sets of templates for deploying a YDB cluster. All these options can be scaled to any required number of servers, considering a number of technical requirements.

To prepare your template, you can follow the instructions below:

- Create a copy of the directory with the ready example (3-nodes-mirror-3-dc-v2, 9-nodes-mirror-3-dc-v2, or 8-nodes-block-4-2-v2).
- Specify the FQDNs of the servers in the file TLS/ydb-ca-nodes.txt and execute the script ydb-ca-update.sh to generate sets of TLS certificates
- Change the group_vars/ydb/all.yaml according to the instructions.
- Make changes to the YDB configuration file according to the instructions.
- In the directory of the cloned template, execute the command ansible-playbook ydb_platform.ydb.initial_setup
