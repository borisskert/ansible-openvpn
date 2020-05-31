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
| openvpn_alpine_version | text | no         | `latest`  | Your selected alpine version |
| openvpn_version | text | no        | `latest`  | Your selected OpenVPN-Server version |
| openvpn_ca_key_size     | number     | no    | 2048    | Keysize of the Certificate Authority |
| openvpn_server_key_size | number     | no    | 2048    | Keysize of the server certificate |
| openvpn_dh_parameter_size | number   | no    | 1024    | Size of the Diffie-Hellman parameter |
| openvpn_ca_country_name   | text     | yes   |         | Certificate authority country code (i.e. `US`) |
| openvpn_ca_locality_name  | text     | yes   |         | Certificate authority locality (i.e. `San Francisco`) |
| openvpn_ca_state_or_province_name | text | yes |       | Certificate authority state or province (i.e. `CA`) |
| openvpn_ca_email_address          | text | yes |       | Certificate authority email address (i.e. `me@myhost.mydomain.org`) |
| openvpn_ca_common_name            | text | yes |       | Certificate authority common name (i.e. `mydomain.org`) |
| openvpn_ca_organization_name      | text | yes |       | Certificate authority organization name (i.e. `Fort-Funston`) |
| openvpn_ca_organizational_unit_name | text | yes |     | Certificate authority organizational unit name (i.e. `IT Division`) |
| openvpn_server_common_name          | text | yes |     | Common name of the server certificate |
| openvpn_server_address              | text | yes |     | The external server address (seen by clients) |
| openvpn_clients                     | array of `client` | no | [] | A list of clients. For each client a client config in `ovpn` format will be created and downloaded to your local directory defined in `openvpn_download_dir` |
| openvpn_download_dir                | text              | no | `./clients` | Your local directory the created client `ovpn` configuration files will be downloaded |
| openvpn_tls_version_min             | text              | no | `1.2` | Option to enforce a minimum TLS version |
| openvpn_verbosity                   | number            | no | `3` | The verbosity level of your OpenVPN server |
| openvpn_server_ip                   | ip address        | no | `10.0.0.1` | OpenVPN server ip address within the vpn net |
| openvpn_network_ip                  | ip address        | no | `10.0.0.0` | OpenVPN network address |
| openvpn_subnet_mask                 | subnet mask       | no | `255.255.0.0` | OpenVPN subnet mask |
| openvpn_routes                      | array of text     | no | [] | Used routes configured in OpenVPN server |
| openvpn_pushes                      | array of text     | no | [] | Rules pushed to the OpenVPN clients |
| openvpn_network_device              | text              | no | `tun` | The used OpenVPN device |
| openvpn_network_protocol            | text              | no | `udp` | The used OpenVPN protocol |

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
      openvpn_working_directory: /srv/openvpn
      openvpn_ca_common_name: mydomain.org
      openvpn_ca_country_name: US
      openvpn_ca_state_or_province_name: CA
      openvpn_ca_locality_name: San Francisco
      openvpn_ca_organization_name: Fort-Funston
      openvpn_ca_email_address: me@mydomain.org
      openvpn_ca_organizational_unit_name: IT Division
      openvpn_server_common_name: openvpn.mydomain.org
      openvpn_server_address: 192.168.33.61
```

### Typical `playbook.yml`

```yaml
- hosts: test_machine
  become: yes

  roles:
    - role: install-openvpn
      openvpn_alpine_version: 3.11.6
      openvpn_version: 2.4.8-r1
      openvpn_working_directory: /srv/openvpn
      openvpn_ca_common_name: mydomain.org
      openvpn_ca_country_name: US
      openvpn_ca_state_or_province_name: CA
      openvpn_ca_locality_name: San Francisco
      openvpn_ca_organization_name: Fort-Funston
      openvpn_ca_email_address: me@mydomain.org
      openvpn_ca_organizational_unit_name: IT Division
      openvpn_server_common_name: openvpn.mydomain.org
      openvpn_server_address: 192.168.33.61
      openvpn_ca_key_size: 4096
      openvpn_server_key_size: 4096
      openvpn_dh_parameter_size: 4096
      openvpn_client_cert_size: 4096
      openvpn_download_dir: ~/my-ovpn-clients
      openvpn_tls_version_min: 1.3
      openvpn_server_ip: 10.0.0.1
      openvpn_network_ip: 10.0.0.0
      openvpn_subnet_mask: 255.255.0.0
      openvpn_pushes:
        - dhcp-option DNS 192.168.33.1
      openvpn_network_device: tun
      openvpn_network_protocol: udp
      openvpn_clients:
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

## iptables configuration

By default iptables is blocking your openvpn traffic so you need to configure them.
I didn't use ansible to do that.

[Arash Milani](https://arashmilani.com) wrote a [small article](https://arashmilani.com/post?id=53) that helped me a lot.
 Thank you for that!

```shell script
#!/bin/bash

# Backup your iptables settings (restore them with `iptables-restore < ~/iptables.backup`)
iptables-save > ~/iptables.backup

# Setup your ethernet here. See a list of your interfaces here with this command: `ip link show` 
INTERFACE=eth0

# Setup your used OpenVPN protocol
PROTOCOL=udp

# Change the ip address mask according to your info of tun0 result while running `ifconfig` command
MASK=10.0.0.0/24

# Allow the tcp connection on the OpenVPN port
iptables -A INPUT -i "$INTERFACE" -m state --state NEW -p "$PROTOCOL" --dport 1194 -j ACCEPT

# Allow TUN interface connections to OpenVPN server
iptables -A INPUT -i tun+ -j ACCEPT

# Allow TUN interface connections to be forwarded through other interfaces
iptables -A FORWARD -i tun+ -j ACCEPT
iptables -A FORWARD -i tun+ -o "$INTERFACE" -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i "$INTERFACE" -o tun+ -m state --state RELATED,ESTABLISHED -j ACCEPT

# NAT the VPN client traffic to the Internet.
iptables -t nat -A POSTROUTING -s "$MASK" -o "$INTERFACE" -j MASQUERADE

# If your default iptables OUTPUT value is not ACCEPT, you will also need a line like:
iptables -A OUTPUT -o tun+ -j ACCEPT
```

## Testing

Requirements:

* [Vagrant](https://www.vagrantup.com/)
* [VirtualBox](https://www.virtualbox.org/)
* [Ansible](https://docs.ansible.com/)
* [Molecule](https://molecule.readthedocs.io/en/latest/index.html)
* [yamllint](https://yamllint.readthedocs.io/en/stable/#)
* [ansible-lint](https://docs.ansible.com/ansible-lint/)
* [Docker](https://docs.docker.com/)

### Run within docker

```shell script
molecule test
```

### Run within Vagrant

```shell script
 molecule test --scenario-name vagrant --parallel
```

I recommend to use [pyenv](https://github.com/pyenv/pyenv) for local testing.
Within the Github Actions pipeline I use [my own molecule Docker image](https://github.com/borisskert/docker-molecule).

## License

MIT

## Author Information

* [borisskert](https://github.com/borisskert)
