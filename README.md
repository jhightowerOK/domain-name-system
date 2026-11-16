## Authoritative DNS Server on Debian 13 (Trixie)

Use BIND9 (bind9), the standard open-source DNS server software, to configure an authoritative internal zone example, `lab.local`.

---

#### Update Packages and Install BIND9

Requires root or sudo privileges
Refresh the package index and install BIND9 along with its command-line diagnostic utilities (dig, nslookup, and configuration checkers).

```
sudo apt update && sudo apt upgrade -y
sudo apt install -y bind9 bind9-utils bind9-doc dnsutils
```

#### Configure BIND9 Options and Forwarders

Back up default config files before editing
Configure BIND9 to listen on your server's IPv4 interface, restrict recursion to your local lab subnet (preventing an open resolver), and forward external internet queries upstream.

Edit `/etc/bind/named.conf.options`:

```
acl "trusted" {
    127.0.0.0/8;
    192.168.1.0/24;  // Adjust to match your lab subnet
};

options {
    directory "/var/cache/bind";

    recursion yes;
    allow-recursion { trusted; };
    listen-on { 127.0.0.1; 192.168.1.10; }; // Replace .10 with server IP
    listen-on-v6 { none; };

    forwarders {
        1.1.1.1;
        9.9.9.9;
    };

    dnssec-validation auto;
};
```

#### Declare Forward and Reverse Zones

Register your custom domain (lab.local) and its corresponding reverse lookup zone in /etc/bind/named.conf.local.

```
zone "lab.local" {
    type master;
    file "/etc/bind/zones/db.lab.local";
};

zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.192.168.1";
};
```

#### Create the Forward Zone File

Always end fully qualified domain names with a trailing dot
Create a dedicated directory for your zone files and populate the forward lookup records.

```
sudo mkdir -p /etc/bind/zones
Create /etc/bind/zones/db.lab.local:

Plaintext
$TTL    604800
@       IN      SOA     ns1.lab.local. admin.lab.local. (
                          2026111601 ; Serial (YYYYMMDDNN)
                              604800 ; Refresh
                               86400 ; Retry
                             2419200 ; Expire
                              604800 ) ; Negative Cache TTL
;
@       IN      NS      ns1.lab.local.
ns1     IN      A       192.168.1.10
web     IN      A       192.168.1.20
www     IN      CNAME   web.lab.local.
```

#### Create the Reverse Zone File

Populate /etc/bind/zones/db.192.168.1 so IP addresses resolve back to hostnames via PTR records.

```
$TTL    604800
@       IN      SOA     ns1.lab.local. admin.lab.local. (
                          2026111601 ; Serial
                              604800 ; Refresh
                               86400 ; Retry
                             2419200 ; Expire
                              604800 ) ; Negative Cache TTL
;
@       IN      NS      ns1.lab.local.
10      IN      PTR     ns1.lab.local.
20      IN      PTR     web.lab.local.
```

#### Validate Syntax and Restart BIND9

Do not restart the service until both syntax checks pass silently
Run BIND's built-in validation tools to catch missing semicolons or trailing dots, then reload the daemon.

```
sudo named-checkconf
sudo named-checkzone lab.local /etc/bind/zones/db.lab.local
sudo named-checkzone 1.168.192.in-addr.arpa /etc/bind/zones/db.192.168.1

sudo systemctl restart named
sudo systemctl enable named
```

#### Verify Resolution with dig

Query the local DNS server directly (@127.0.0.1) to confirm forward, reverse, and forwarded external lookups work properly.

```Bash
# Test forward A record
dig @127.0.0.1 web.lab.local +short

# Test reverse PTR record
dig @127.0.0.1 -x 192.168.1.20 +short

# Test external forwarding
dig @127.0.0.1 debian.org +short
```
