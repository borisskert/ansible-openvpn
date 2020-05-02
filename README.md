# ansible-openvpn

Configure and install on OpenVPN server as docker container running as systemd service.

## System requirements

* Docker
* Systemd

## Role requirements

* python-docker package

## What this role does

* Create Certificate Authority
* Create server certificate
* Generate Diffie-Hellman parameter
* Create TLS-Auth Key
* Create Client certificates (and download them)
* Build Docker image (based on alpine)
* Create and enable Systemd service

## Role parameters

| Variable       | Type | Mandatory? | Default | Description                  |
|----------------|------|------------|---------|------------------------------|
| alpine_version | text | no         | latest  | Your selected alpine version |
| openvpn_version | text | no        | latest  | Your selected OpenVPN-Server version |
| openvpn_port    | number | no        | 1194  | Used network port |
| interface       | ip address | no    | 0.0.0.0 | Bound network address |
| ca_key_size     | number     | no    | 2048    | Keysize of the Certificate Authority |
| server_key_size | number     | no    | 2048    | Keysize of the server certificate |
| dh_parameter_size | number   | no    | 1024    | Size of the Diffie-Hellman parameter |
| ca_country_name   | text     | yes   |         | Certificate authority country code (i.e. `US`) |
| ca_locality_name  | text     | yes   |         | Certificate authority locality (i.e. `San Francisco`) |
| ca_state_or_province_name | text | yes |       | Certificate authority state or province (i.e. `CA`) |
| ca_email_address          | text | yes |       | Certificate authority email address (i.e. `me@myhost.mydomain.org`) |
| ca_common_name            | text | yes |       | Certificate authority common name (i.e. `mydomain.org`) |
| ca_organization_name      | text | yes |       | Certificate authority organization name (i.e. `Fort-Funston`) |
| ca_organizational_unit_name | text | yes |     | Certificate authority organizational unit name (i.e. `IT Division`) |
| server_common_name          | text | yes |     | Common name of the server certificate |
| clients                     | array of `client` | no | [] | A list of clients. For each client a client config in `ovpn` format will be created and downloaded to your local directory defined in `download_dir` |
| download_dir                | text              | no | `./clients` | Your local directory the created client `ovpn` configuration files will be downloaded |
| tls_version_min             | text              | no | `1.2` | Option to enforce a minimum TLS version |

### `client` definition

| Variable | Type | Mandatory? | Default | Description                  |
|----------|------|------------|---------|------------------------------|
| name     | text | yes        |         | Your client's common name. Used for the corresponding client certificate and `ovpn` filename |
| passphrase | text | no       |         | The client's private key passphrase                                                          |

## Usage

### Add to `requirements.yml`

```yaml
- name: install-openvpn
  src: https://github.com/borisskert/ansible-openvpn.git
  scm: git
```

### Minimal `playbook.yml`

```yaml
- hosts: test_machine
  become: yes

  roles:
    - role: install-openvpn
      working_directory: /srv/openvpn
      ca_common_name: mydomain.org
      ca_country_name: US
      ca_state_or_province_name: CA
      ca_locality_name: San Francisco
      ca_organization_name: Fort-Funston
      ca_email_address: me@mydomain.org
      ca_organizational_unit_name: IT Division
      server_common_name: openvpn.mydomain.org
      server_address: 192.168.33.21
```

### Typical `playbook.yml`

```yaml
- hosts: test_machine
  become: yes

  roles:
    - role: install-openvpn
      alpine_version: 3.11.6
      openvpn_version: 2.4.8-r1
      openvpn_port: 1194
      interface: 0.0.0.0
      working_directory: /srv/openvpn
      ca_common_name: mydomain.org
      ca_country_name: US
      ca_state_or_province_name: CA
      ca_locality_name: San Francisco
      ca_organization_name: Fort-Funston
      ca_email_address: me@mydomain.org
      ca_organizational_unit_name: IT Division
      server_common_name: openvpn.mydomain.org
      server_address: 192.168.33.21
      ca_key_size: 4096
      server_key_size: 4096
      dh_parameter_size: 4096
      client_cert_size: 4096
      download_dir: ~/my-ovpn-clients
      tls_version_min: 1.3
      clients:
        - name: vpnclient01
          passphrase: abc
        - name: vpnclient02
```

## Running Tests

You need:

* Vagrant
* VirtualBox
* Ansible

Run tests:

```shell script
cd tests
./tests.sh
```
