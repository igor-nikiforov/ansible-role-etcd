# Ansible role for etcd
This role configures etcd and etcdctl on your target host. It supports all etcd configuration options and strives to be as flexible as possible.

## Requirements

This role developed and tested with following Ansible versions:

| Name                                                   | Version         |
|--------------------------------------------------------|-----------------|
| [ansible](https://pypi.org/project/ansible-base/)      | ```>= 2.9.13``` |
| [ansible-base](https://pypi.org/project/ansible-base/) | ```>= 2.10.1``` |

Other Ansible versions was not tested but will probably work.

## Installation

Use ```ansible-galaxy install igor_nikiforov.etcd``` to install the latest stable release of role.

You could also install it from requirements ```ansible-galaxy install -r requirements.yml```:

```yaml
# requirements.yml
---
roles:
  - name: igor_nikiforov.etcd
    version: v1.0.0
```

## Platforms

| Name   | Version             |
|--------|---------------------|
| Debian | ```buster```        |
| Ubuntu | ```focal, groovy``` |
| CentOS | ```7.4+, 8```       |
| RedHat | ```7.4+, 8```       |

Other OS distributions was not tested but will probably work. In case if not please raise a PR!

## Variables

| Name                                                                                           | Description                                        | Default   |
|------------------------------------------------------------------------------------------------|----------------------------------------------------|-----------|
| <a name="etcd_version"></a> [etcd_version](#variable\_etcd_version)                            | Version of etcd to be installed                    | `3.4.13`  |
| <a name="etcd_user"></a> [etcd_user](#variable\_etcd_user)                                     | etcd user                                          | `etcd`    |
| <a name="etcd_group"></a> [etcd_group](#variable\_etcd_group)                                  | etcd group                                         | `etcd`    |
| <a name="etcd_config"></a> [etcd_config](#variable\_etcd_config)                               | List of key-values etcd configuration parameters.  | `{}`      |
| <a name="etcd_service_enabled"></a> [etcd_service_enabled](#variable\_etcd_service_enabled)    | Whether the service should start on boot.          | `True`    |
| <a name="etcd_service_state"></a> [etcd_service_state](#variable\_etcd_service_state)          | Service state for etcd.                            | `started` |
| <a name="etcdctl_output_format"></a> [etcdctl_output_format](#variable\_etcdctl_output_format) | Output format to be used in etcdctl.               | `table`   |

## Usage

Role supports all etcd configuration parameters which could be passed via ```etcd_config``` variable. You could find example of YAML config format in [etcd official repository](https://github.com/etcd-io/etcd/blob/main/etcd.conf.yml.sample) and all availible flags with discription in [etcd official documentation](https://etcd.io/docs/v3.4/op-guide/configuration/).

etcd supports two main methods to build a cluster:

1. [Static.](https://etcd.io/docs/v3.4/op-guide/clustering/#static)

   After playbook execution you should manually add each member from one of host using ```etcdctl member add``` command. It supposing that you will do this manually or automate in separate Ansible task.

2. [DNS discovery.](https://etcd.io/docs/v3.4/op-guide/clustering/#dns-discovery)

   Main prerequisite here is to have ready SRV and A records in your DNS local zone. Please carefully check [requirements for DNS records](https://etcd.io/docs/v3.4/op-guide/clustering/#create-dns-srv-records) which should be created in advance. If everything created properly following DNS discovery related properties needs to be added to ```etcd_config```:
   ```yaml
   etcd_config:
     discovery-srv: "company.local"
     discovery-srv-name: "dev" # optional
   ```
   After playbook execution etcd cluster will be automatically created. It is strongly recommended to use this method in production.

Important:
- Don't forget to change ```etcd_config.initial-cluster-state``` from ```new``` to ```existing``` in playbook after first execution.
- Use ```serial: 1``` in your playbook after you build a cluster to safely update it in case of configuration change. More info [here](https://docs.ansible.com/ansible/latest/user_guide/playbooks_strategies.html).

## Examples
### Static

```yaml
# playbook.yml
---
- hosts: all
  become: True
  gather_facts: False

  pre_tasks:
    - wait_for_connection: {timeout: 300}
    - setup:

  tasks:
    - name: Install etcd
      import_role:
        name: etcd
      vars:
        etcd_version: "3.4.13"
        etcd_config:
          name: "{{ ansible_facts.hostname }}"
          data-dir: "/var/lib/etcd/data"
          wal-dir: "/var/lib/etcd/wal"
          initial-advertise-peer-urls: "https://{{ ansible_facts.fqdn }}:2380"
          initial-cluster-token: "token"
          initial-cluster-state: "new"
          advertise-client-urls: "https://{{ ansible_facts.fqdn }}:2379"
          listen-client-urls: "https://{{ ansible_default_ipv4.address }}:2379,https://127.0.0.1:2379"
          listen-peer-urls: "https://{{ ansible_default_ipv4.address }}:2380"
          client-transport-security:
            trusted-ca-file: "{{ etcd_conf_dir }}/certs/ca.crt"
            cert-file: "{{ etcd_conf_dir }}/certs/server.crt"
            key-file: "{{ etcd_conf_dir }}/certs/server.key"
          peer-transport-security:
            trusted-ca-file: "{{ etcd_conf_dir }}/certs/ca.crt"
            cert-file: "{{ etcd_conf_dir }}/certs/server.crt"
            key-file: "{{ etcd_conf_dir }}/certs/server.key"
          log-level: "debug"
          logger: "zap"
```

### DNS discovery

```yaml
# playbook.yml
---
- hosts: all
  become: True
  gather_facts: False

  pre_tasks:
    - wait_for_connection: {timeout: 300}
    - setup:

  tasks:
    - name: Install etcd
      import_role:
        name: etcd
      vars:
        etcd_version: "3.4.13"
        etcd_config:
          name: "{{ ansible_facts.hostname }}"
          data-dir: "/var/lib/etcd/data"
          wal-dir: "/var/lib/etcd/wal"
          discovery-srv: "company.local"
          initial-advertise-peer-urls: "https://{{ ansible_facts.fqdn }}:2380"
          initial-cluster-token: "token"
          initial-cluster-state: "new"
          advertise-client-urls: "https://{{ ansible_facts.fqdn }}:2379"
          listen-client-urls: "https://{{ ansible_default_ipv4.address }}:2379,https://127.0.0.1:2379"
          listen-peer-urls: "https://{{ ansible_default_ipv4.address }}:2380"
          client-transport-security:
            trusted-ca-file: "{{ etcd_conf_dir }}/certs/ca.crt"
            cert-file: "{{ etcd_conf_dir }}/certs/server.crt"
            key-file: "{{ etcd_conf_dir }}/certs/server.key"
          peer-transport-security:
            trusted-ca-file: "{{ etcd_conf_dir }}/certs/ca.crt"
            cert-file: "{{ etcd_conf_dir }}/certs/server.crt"
            key-file: "{{ etcd_conf_dir }}/certs/server.key"
          log-level: "debug"
          logger: "zap"
```

## License

MIT

## Author Information

[Igor Nikiforov](https://github.com/igor-nikiforov)