# OCP Quicklab installer
This repository contains all what is necessary to install OCP 3.11 from scratch in a quicklab environment.

<!-- TOC -->

1. [Section 2.Configure the installation](#section-2configure-the-installation)
    1. [Section 2.1 Configure the VMs](#section-21-configure-the-vms)
        1. [Directory structure](#directory-structure)
        2. [Configure /group_vars/all.yml](#configure-group_varsallyml)
        3. [Configure /inventory-configuration](#configure-inventory-configuration)
        4. [Configure /group_vars/vault.yml](#configure-group_varsvaultyml)
    2. [Section 2.2 Configure the OCP playbook](#section-22-configure-the-ocp-playbook)
        1. [RH registry authentication](#rh-registry-authentication)
        2. [Duckdns](#duckdns)
        3. [Htpasswd users](#htpasswd-users)
    3. [Section 2.3 Run the configurer](#section-23-run-the-configurer)
2. [Section 3. Install OCP](#section-3-install-ocp)
3. [Section 4. Reset localhost](#section-4-reset-localhost)

<!-- /TOC -->



## Section 2.Configure the installation

### Section 2.1 Configure the VMs

This section uses Ansible to perform the following actions:

- In localhost:
  - Configure `/home/$USER/.ssh/config` to access VMs with the SSH correct user.
  - Configure `/etc/hosts` to resolve correctly the IPs of the VMs.
- In OCP VMs:
  - Copy the ssh keys.
  - Allow openshift users to escalate privileges without password.
  - Register the VMs.
  - Configure OCP repositories
  - Install and update packages.

#### Directory structure

The following diagram shows the structure of folder "2-vms-configurer".

```bash
├── 2-vms-configurer
│   ├── ansible.cfg
│   ├── ansible.log
│   ├── group_vars
│   │   ├── all.yml
│   │   └── vault.yml
│   ├── inventory
│   ├── main.yml
│   └── named
│       ├── 122.168.192.db
│       ├── named.conf
│       └── ocp.vm.db
```

#### Configure /group_vars/all.yml

This file contains:
- All the hosts with IPs.
- Repositories to enable in the VMs.
- Packages to install in all the VMs and extra packages for the bastion.

Modify this yaml to add or remove any configuration


#### Configure /inventory-configuration

This file allows you to classify the hosts that you created in section 1. You can classify machines as you wish. The only requirement is that some packages will be installed only in VMs under the group `[bastion]`, so define at least one machine in this group.

For example, this is a copy of the file used currently:

```ini
[bastion]
bastion.miniocp.vm

[ocp]
master.miniocp.vm
infra.miniocp.vm
node.miniocp.vm
```

#### Configure /group_vars/vault.yml

Create a new file `/group_vars/vault.yml` with the following content:

```yaml
vault_rh_subscription_username: <RH_USERNAME>
vault_rh_subscription_password: <RH_PASSWORD>
vault_rh_subscription_poolid:   <POOL_ID>
```
This file has not been added to the repository for security purposes and `.gitignore` is configured to ignore this file.



  

### Section 2.2 Configure the OCP playbook

#### RH registry authentication

The new registry, `registry.redhat.io`, requires authentication for access to images and hosted content on OpenShift Container Platform. Access https://access.redhat.com/containers/ and create a new service account. Then, add this configuration to the Ansible Vault file `/group_vars/vault.yml`. This variables will then be copied to the `/group_vars/all.yml` in execution time.
```yaml
vault_rh_registry_username: <username>
vault_rh_registry_token: <token_value>
```

For more information, please check the following instructions: https://docs.openshift.com/container-platform/3.11/install_config/configuring_red_hat_registry.html#install-config-configuring-red-hat-registry



#### Htpasswd users

To simplify the installation and configuration process of the security in the cluster, the `hosts` file contains some predefined user and passwords. To generate passwords use the `htpasswd` command:

```bash
htpasswd -nb <user> <password>
```

After that, copy the users and passwords in `/group_vars/all.yml` with the following structure:

```yaml
openshift_master_htpasswd_users:
  - user: admin
    pass: $apr1$His0EwFR$UkDefLNZsO7SVCO.5932t1 # admin
  - user: developer
    pass: $apr1$S9dJkChQ$SrjX./WknEViSPERfa38t0 # developer
```

#### Duckdns

Duck DNS is a free service which will point a DNS (sub domains of duckdns.org) to an IP of your choice. This add-on includes support for Let’s Encrypt and will automatically create and renew your certificates. You will need to sign up for a Duck DNS account before using this add-on.

Go to https://www.duckdns.org and create a new account. Choose the sub-domain that you prefer and add it to your account. Now you can use it as your `openshift_master_default_subdomain` in the `/group_vars/all.yml` file. For example:

```yaml
openshift_master_default_subdomain: miniocp.duckdns.org
```



### Section 2.3 Run the configurer

Once all the files are set, you need to run two playbooks to configure all the machines.

The **first playbook** configures localhost to connect correctly to the VMs.

```bash
ansible-playbook configurer-localhost.yml
```

This command will prompt you for two passwords required for different things:
   
1) **"SUDO password"**. This password is prompted because `ansible.cfg` contains the option `become_ask_pass = True`. It is also possible to add the option `--ask-become-pass`in the `ansible-playbook` command to obtain the same result. It allows you to escalate privileges in your host machine to configure `/etc/hosts`. This is the password to run `sudo` commands.

<!-- 2) **"Vault password"**. This password is prompted because `ansible.cfg` contains the option `ask_vault_pass = True`. It is also possible to add the option `--ask-vault-pass` in the `ansible-playbook` command to obtain the same result. It allows you to open the vault file that contains information about your RH account. -->




The **second playbook** prepares the nodes for the OCP installation:

```bash
ansible-playbook configurer-vms.yml --ask-vault-pass
```

This command will prompt you for three passwords required for different things:

   
1) **"SUDO password[defaults to SSH password]"**. This password is prompted because `ansible.cfg` contains the option `become_ask_pass = True`. It is also possible to add the option `--ask-become-pass`in the `ansible-playbook` command to obtain the same result. It allows you to escalate privileges in your host machine to configure `/etc/sudoers`. This is the password to run `sudo` commands.

2) **"Vault password"**. This password is prompted because `ansible.cfg` contains the option `ask_vault_pass = True`. It is also possible to add the option `--ask-vault-pass` in the `ansible-playbook` command to obtain the same result. It allows you to open the vault file that contains information about your RH account.

3) **"SSH password"**. This password is prompted because the option `--ask-pass` is present. It allows you to connect to all the hosts to copy the ssh-key. By default, according to the kcli template, the password is `openshift`. This parameter is only necessary the first time that Ansible connects to each host.





## Section 3. Install OCP

To install OCP 3.11 just ssh the bastion machine and execute the installation playbook:

```bash
ssh lb-0.alopezme.lab.pnq2.cee.redhat.com
cd ocp-cluster-installer
ansible-playbook install.yml
ansible-playbook post-install.yml
```

Now that everything is installed, you can adjust the memory settings to give more RAM for the applications:

```bash
kcli update --memory 2048 bastion.miniocp.vm    # Previously 4096, now 2GB
kcli update --memory 4096 master.miniocp.vm     # Previously 6144, now 4GB
kcli update --memory 9216 infra.miniocp.vm      # Previously 6144, now 9GB
kcli update --memory 7168 node.miniocp.vm       # Previously 4096, now 7GB
# Total = 22 GB
```

There is one pod that cannot start do to its memory request `logging-es-data-master` of project `openshift-logging`:

```bash
oc patch dc/logging-es-data-master-juk77cph -p '{"spec": {"template": {"spec": {"containers": [{"name": "elasticsearch","resources": {"requests": {"memory": "1Gi"}}}]}}}}' -n openshift-logging
oc rollout latest logging-es-data-master-juk77cph
```

Finally, now you are able to log in as `oc login -u system:admin` from the master node to give clusterAdmin permissions to your account:

```bash
oc adm policy add-cluster-role-to-user cluster-admin admin
oc adm policy add-cluster-role-to-user cluster-admin alvaro
```

## Section 4. Reset localhost


```bash
ansible-playbook configurer-uninstaller.yml --ask-pass
```