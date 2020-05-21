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
| alpine_version | text | no         | `latest`  | Your selected alpine version |
| openvpn_version | text | no        | `latest`  | Your selected OpenVPN-Server version |
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
| server_address              | text | yes |     | The external server address (seen by clients) |
| clients                     | array of `client` | no | [] | A list of clients. For each client a client config in `ovpn` format will be created and downloaded to your local directory defined in `download_dir` |
| download_dir                | text              | no | `./clients` | Your local directory the created client `ovpn` configuration files will be downloaded |
| tls_version_min             | text              | no | `1.2` | Option to enforce a minimum TLS version |
| verbosity                   | number            | no | `3` | The verbosity level of your OpenVPN server |
| server_ip                   | ip address        | no | `10.0.0.1` | OpenVPN server ip address within the vpn net |
| network_ip                  | ip address        | no | `10.0.0.0` | OpenVPN network address |
| subnet_mask                 | subnet mask       | no | `255.255.0.0` | OpenVPN subnet mask |
| routes                      | array of text     | no | [] | Used routes configured in OpenVPN server |
| pushes                      | array of text     | no | [] | Rules pushed to the OpenVPN clients |
| network_device              | text              | no | `tun` | The used OpenVPN device |
| network_protocol            | text              | no | `udp` | The used OpenVPN protocol |

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
      server_address: 192.168.33.61
```

### Typical `playbook.yml`

```yaml
- hosts: test_machine
  become: yes

  roles:
    - role: install-openvpn
      alpine_version: 3.11.6
      openvpn_version: 2.4.8-r1
      working_directory: /srv/openvpn
      ca_common_name: mydomain.org
      ca_country_name: US
      ca_state_or_province_name: CA
      ca_locality_name: San Francisco
      ca_organization_name: Fort-Funston
      ca_email_address: me@mydomain.org
      ca_organizational_unit_name: IT Division
      server_common_name: openvpn.mydomain.org
      server_address: 192.168.33.61
      ca_key_size: 4096
      server_key_size: 4096
      dh_parameter_size: 4096
      client_cert_size: 4096
      download_dir: ~/my-ovpn-clients
      tls_version_min: 1.3
      server_ip: 10.0.0.1
      network_ip: 10.0.0.0
      subnet_mask: 255.255.0.0
      pushes:
        - dhcp-option DNS 192.168.33.1
      network_device: tun
      network_protocol: udp
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
