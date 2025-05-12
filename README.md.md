
# BIND9 DNS Server Setup (Ubuntu VM using Vagrant)

This guide outlines the steps to install and configure a BIND9 DNS server on an Ubuntu virtual machine provisioned with Vagrant. The domain used in this setup is `templab.lan`, and the server IP is `192.168.10.13`.

---

## 1. Install BIND9 and Utilities

Install the required packages:

```bash
sudo apt update
sudo apt install -y bind9 bind9utils bind9-doc dnsutils
```

---

## 2. Navigate to BIND Configuration Directory

```bash
cd /etc/bind
```

---

## 3. Backup Configuration Files

Before making changes, back up existing files:

```bash
sudo cp named.conf.options named.conf.options.bak
sudo cp named.conf.local named.conf.local.bak
```

---

## 4. Configure Global DNS Options

Edit the options file:

```bash
sudo vim named.conf.options
```

Example configuration:

```conf
options {
    directory "/var/cache/bind";

    listen-on port 53 { 127.0.0.1; };
    allow-query { any; };
    recursion no;

    forwarders {
        8.8.8.8;
        1.1.1.1;
    };

    dnssec-validation auto;
    auth-nxdomain no;
};
```

Validate the configuration:

```bash
sudo named-checkconf
```

---

## 5. Define DNS Zones

Edit the zone configuration file:

```bash
sudo vim named.conf.local
```

Add:

```conf
zone "templab.lan" {
    type master;
    file "/etc/bind/db.templab.lan";
};

zone "17.16.172.in-addr.arpa" {
    type master;
    file "/etc/bind/db.172.16.17";
};
```

---

## 6. Create Zone Files

### Forward Zone File

Create from the default template:

```bash
sudo cp db.local db.templab.lan
sudo vim db.templab.lan
```

Example `db.templab.lan` content:

```dns
$TTL    604800
@       IN      SOA     ns1.templab.lan. admin.templab.lan. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

@       IN      NS      ns1.templab.lan.
ns1     IN      A       192.168.10.13
dhcp1   IN      A       192.168.10.14
```

### Reverse Zone File

Create from the loopback reverse zone:

```bash
sudo cp db.127 db.172.16.17
sudo vim db.172.16.17
```

Example `db.172.16.17` content:

```dns
$TTL    604800
@       IN      SOA     ns1.templab.lan. admin.templab.lan. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

@       IN      NS      ns1.templab.lan.
13      IN      PTR     ns1.templab.lan.
14      IN      PTR     dhcp1.templab.lan.
```

---

## 7. Validate Zone Files

Check for syntax errors:

```bash
sudo named-checkzone templab.lan db.templab.lan
sudo named-checkzone 17.16.172.in-addr.arpa db.172.16.17
```

---

## 8. Configure Netplan to Use DNS Server

```bash
cd /etc/netplan
sudo vim 50-vagrant.yaml
```

Example:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s8:
      addresses:
        - 192.168.251.25/24
      nameservers:
        addresses: [127.0.0.1]
```

Apply the config:

```bash
sudo netplan apply
```

---

## 9. Restart and Check BIND Service

Restart the DNS server:

```bash
sudo systemctl restart bind9
sudo systemctl status bind9
```

You should see the service active and listening on ``.

---

## Notes

- `templab.lan` is your internal DNS domain.
- `ns1.templab.lan.` is the name server.
- `admin.templab.lan.` is the admin contact (replace `@` with `.`).
- Reverse zone `17.16.172.in-addr.arpa` matches IP range `172.16.17.0/24`.

---