Ansible Role: Hashicorp Vault Server Installation and Configuration
=========

This role install and configure [Hashicorp Vautl](https://www.vaultproject.io/) in a linux server.


Requirements
------------

None

Role Variables
--------------

Available variables are listed below along with default values (see `defaults\main.yaml`)

- Vault server installation details

  Vault UNIX user/group
  ```yml
  vault_group: vault
  vault_user: vault
  ```
  
  Vault package and version to be installed
  ```yml
  vault_version: 1.12.2
  ```

  Vault installation Paths

  ```yml
  vault_bin_path: /usr/local/bin
  vault_config_path: /etc/vault
  vault_tls_path: /etc/vault/tls
  vault_plugin_path: /usr/local/lib/vault/plugins
  vault_data_path: /var/lib/vault
  vault_log_path: /var/log/vault
  ```
- Vault TLS configuration

  ```yml
  vault_enable_tls: false
  vault_key: ""
  vault_cert: ""
  custom_ca: false
  vault_ca: ""
  # vault service dns
  vault_dns: ""
  ```

  To enable configuration of TLS set `vault_enable_tls` to true and provide the private key and public certificate as content loaded into `vault_key` and `vault_cert` variables.

  if custom CA has been used to sign the TLS certificates, `custom_ca` need to be set to true, and CA certificate need to be loaded int `vault-ca` variable.

  Set `vault_dns` to FQDN of vault service used to issue the Certificate.

  They can be loaded from files using an ansible task like:

  ```yml
  - name: Load tls key and cert from files
  set_fact:
    vault_key: "{{ lookup('file','certificates/{{ inventory_hostname }}_private.key') }}"
    vault_cert: "{{ lookup('file','certificates/{{ inventory_hostname }}_public.crt') }}"
    vault_ca: "{{ lookup('file','certificates/ca.crt') }}"
  ```

- Vault inititialize

  ```yml
  vault_init: false
  vault_key_shares: 1
  vault_key_threshold: 1
  vault_keys_output: "{{ vault_config_path }}/unseal.json"
  ```

  To automatically initialize vault, set `vault_init` to true, and provide `vault_key_shares` and `vault_key_thershold` variables to specify the number of unseal keys to be generated.

  Initialization will generate json file, `vault_keys_output`, containing keys and root token

- Vault unseal and unseal service
  
  ```yml
  vault_unseal: false
  vault_unseal_service: false
  ```

  To automatically unseal vault, set `vault_unseal` to true. The unsealing process will use keys from `vault_keys_output` file

  Systemd service can be created to automatically unseal vault whenever vault service is started or restarted. To enable it, set `vault_unseal_service` to true. Oneshot `vault-unseal`. This service also uses `vault_keys_output` file.


Dependencies
------------

None

Example Playbook
----------------

The following playbook install and configure vault, enabling TLS and generating custom CA signed SSL certificates.
It also initialize and unseal Vault.

```yml
---
- name: Install and configure Vault Server
  hosts: vault-server
  become: true
  gather_facts: true
  vars:
    server_hostname: vault.ricsanfre.com
    ssl_key_size: 4096
    key_type: RSA
    country_name: ES
    email_address: admin@ricsanfre.com
    organization_name: Ricsanfre
    ansible_user: root

  pre_tasks:
    - name: Generate custom CA
      include_tasks: tasks/generate_custom_ca.yml
      args:
        apply:
          delegate_to: localhost
          become: false
    - name: Generate customCA-signed SSL certificates for minio
      include_tasks: tasks/generate_ca_signed_cert.yml
      args:
        apply:
          delegate_to: localhost
          become: false

    - name: Load tls key and cert
      set_fact:
        vault_key: "{{ lookup('file', 'certificates/' + server_hostname + '.key') }}"
        vault_cert: "{{ lookup('file', 'certificates/' + server_hostname + '.pem') }}"
        vault_ca: "{{ lookup('file', 'certificates/CA.pem') }}"

  roles:
    - role: ricsanfre.vault
      vault_enable_tls: true
      custom_ca: true
      vault_init: true
      vault_unseal: true
      vault_unseal_service: true
      tls_skip_verify: true
      display_init_response: true
```

`pre-tasks` section include tasks to generate a custom CA, and vault's private key and  certificate and load them into `vault_key`, `vault_cert` and `vault-ca` variables.

Where `generate_custom_ca.yml` contain the tasks for generating a custom CA:

```yml
---
- name: Create CA key
  openssl_privatekey:
    path: certificates/CA.key
    size: "{{ ssl_key_size | int }}"
    mode: 0644
  register: ca_key

- name: create the CA CSR
  openssl_csr:
    privatekey_path: certificates/CA.key
    common_name: Ricsanfre CA
    use_common_name_for_san: false  # since we do not specify SANs, don't use CN as a SAN
    basic_constraints:
      - 'CA:TRUE'
    basic_constraints_critical: true
    key_usage:
      - keyCertSign
    key_usage_critical: true
    path: certificates/CA.csr
  register: ca_csr

- name: sign the CA CSR
  openssl_certificate:
    path: certificates/CA.pem
    csr_path: certificates/CA.csr
    privatekey_path: certificates/CA.key
    provider: selfsigned
  register: ca_crt


```

And `generate_ca_signed_certificate.yml` contain the tasks for generating Vault's key and certifica signed by custom CA:

```yml
---
- name: Create private key
  openssl_privatekey:
    path: "certificates/{{ server_hostname }}.key"
    size: "{{ ssl_key_size | int }}"
    type: "{{ key_type }}"
    mode: 0644

- name: Create CSR
  openssl_csr:
    path: "certificates/{{ server_hostname }}.csr"
    privatekey_path: "certificates/{{ server_hostname }}.key"
    country_name: "{{ country_name }}"
    organization_name: "{{ organization_name }}"
    email_address: "{{ email_address }}"
    common_name: "{{ server_hostname }}"
    subject_alt_name: "DNS:{{ server_hostname }},IP:{{ ansible_default_ipv4.address }},IP:127.0.0.1"

- name: CA signed CSR
  openssl_certificate:
    csr_path: "certificates/{{ server_hostname }}.csr"
    path: "certificates/{{ server_hostname }}.pem"
    provider: ownca
    ownca_path: certificates/CA.pem
    ownca_privatekey_path: certificates/CA.key
```


License
-------

MIT

Author Information
------------------

Created by Ricardo Sanchez (ricsanfre)
