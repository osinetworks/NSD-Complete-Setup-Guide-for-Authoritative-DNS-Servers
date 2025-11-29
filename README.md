# NSD Complete Setup Guide

A concise and reproducible guide for deploying an authoritative DNS pair using NSD with TSIG-protected zone replication and optional hidden primary architecture.

---

## *0. Network Topology*

                     Public Internet
                           │
                  (Authoritative DNS Service)
                           │
           ┌───────────────┴───────────────┐
           │     TCP/UDP 53 (DNS Queries)  │
           └───────────────┬───────────────┘
                           │
        ┌──────────────────┴──────────────────┐
        │                                     │
┌───────┴────────────┐        ┌───────────────┴────────────┐
│  NSD Secondary     │        │  NSD Secondary            │
│  (ns1.example.com) │        │  (ns2.example.com)        │
│  IP: 1.23.45.11    │◀──────▶│  IP: 1.23.45.12           │
│  NOTIFY + XFR      │  TSIG  │  XFR (inbound)            │
└───────▲────────────┘ signed  └───────────────▲────────────┘
        │                                     │
        │     allow-notify / request-xfr       │
        └────────────────────┬─────────────────┘
                             │ (not public)
                  ┌──────────┴──────────┐
                  │ Hidden Primary      │
                  │ (master server)     │
                  │ Holds source zones  │
                  │ IP: 1.23.45.10      │
                  └─────────────────────┘


---

## *1. Systemd Service Configuration (Both Servers)*

Allow NSD to write logs when using a dedicated log file.

```bash
sudo cat > /lib/systemd/system/nsd.service << 'EOF'
[Unit]
Description=Name Server Daemon (NSD)
Documentation=man:nsd(8)
After=network.target

[Service]
Type=notify
Restart=always
ExecStart=/usr/sbin/nsd -d -P ""
ExecReload=+/bin/kill -HUP $MAINPID
CapabilityBoundingSet=CAP_CHOWN CAP_IPC_LOCK CAP_NET_BIND_SERVICE CAP_NET_ADMIN CAP_NET_RAW CAP_SETGID CAP_SETUID CAP_SYS_CHROOT
KillMode=mixed
MemoryDenyWriteExecute=true
NoNewPrivileges=true
PrivateDevices=true
PrivateTmp=true
ProtectHome=true
ProtectControlGroups=true
ProtectKernelModules=true
ProtectKernelTunables=true
ProtectSystem=strict
ReadWritePaths=/var/lib/nsd /etc/nsd /run /var/log
RuntimeDirectory=nsd
RestrictRealtime=true
SystemCallArchitectures=native
SystemCallFilter=~@clock @cpu-emulation @debug @keyring @module @mount @obsolete @resources

[Install]
WantedBy=multi-user.target
EOF
```

## *2. Install & Update Packages (Both Servers)*

```bash
sudo systemctl daemon-reload
sudo apt update && apt upgrade -y
sudo apt autoremove -y
sudo apt install nsd -y

```

## *3. Configure NSD on Primary (ns1)*

```bash
sudo cat > /etc/nsd/nsd.conf << 'EOF'
server:
    logfile: "/var/log/nsd.log"
    username: nsd
    ip-address: 1.23.45.11
    do-ip4: yes
    do-ip6: no
    verbosity: 1
    hide-version: yes
    hide-identity: yes
    identity: "ns1.example.com"
    zonesdir: "/etc/nsd/zones"

remote-control:
    control-enable: yes
    control-interface: 127.0.0.1
    control-port: 8952
    server-key-file: "/etc/nsd/nsd_server.key"
    server-cert-file: "/etc/nsd/nsd_server.pem"
    control-key-file: "/etc/nsd/nsd_control.key"
    control-cert-file: "/etc/nsd/nsd_control.pem"

key:
    name: "EXAMPLE-KEY"
    algorithm: hmac-sha256
    secret: "AlgaY9LW2yFdYqrupgNVQYL4+AyxyCQpGxkrupwWtNg="

pattern:
    name: "primary_to_secondary"
    notify: 1.23.45.12 EXAMPLE-KEY
    provide-xfr: 1.23.45.12 EXAMPLE-KEY

zone:
    name: "example.com"
    zonefile: "example.com.zone"
    include-pattern: "primary_to_secondary"

zone:
    name: "example2.com"
    zonefile: "example2.com.zone"
    include-pattern: "primary_to_secondary"

include: "/etc/nsd/nsd.conf.d/*.conf"
EOF
```

## *4. Configure NSD on Secondary (ns2)*

```bash
cat > /etc/nsd/nsd.conf << 'EOF'
server:
    logfile: "/var/log/nsd.log"
    username: nsd
    ip-address: 1.23.45.12
    do-ip4: yes
    do-ip6: no
    verbosity: 1
    hide-version: yes
    hide-identity: yes
    identity: "ns2.example.com"
    zonesdir: "/etc/nsd/zones"

remote-control:
    control-enable: yes
    control-interface: 127.0.0.1
    control-port: 8952
    server-key-file: "/etc/nsd/nsd_server.key"
    server-cert-file: "/etc/nsd/nsd_server.pem"
    control-key-file: "/etc/nsd/nsd_control.key"
    control-cert-file: "/etc/nsd/nsd_control.pem"

key:
    name: "EXAMPLE-KEY"
    algorithm: hmac-sha256
    secret: "AlgaY9LW2yFdYqrupgNVQYL4+AyxyCQpGxkrupwWtNg="

pattern:
    name: "secondary_from_primary"
    allow-notify: 1.23.45.11 EXAMPLE-KEY
    request-xfr: 1.23.45.11 EXAMPLE-KEY

zone:
    name: "example.com"
    zonefile: "example.com.zone"
    include-pattern: "secondary_from_primary"

zone:
    name: "example2.com"
    zonefile: "example2.com.zone"
    include-pattern: "secondary_from_primary"

include: "/etc/nsd/nsd.conf.d/*.conf"
EOF
```

## *5. Create Zone Directory & Set Permissions (Both Servers)*

```bash
sudo mkdir -p /etc/nsd/zones
sudo chown -R nsd:nsd /etc/nsd
```

## *6. Generate Control Keys (Both Servers)*

```bash
sudo nsd-control-setup
```

## *7. Generate New TSIG Secret (Optional but recommended). Use same key for both servers.*
If you want to create a fresh TSIG key:
```bash
dd if=/dev/random bs=32 count=1 2>/dev/null | base64
```

## *8. Sample Forward Zone*

```bash
sudo cat > /etc/nsd/zones/example.com.zone << 'EOF'
$ORIGIN example.com.
$TTL 3600

@ IN SOA ns1.example.com. hostmaster.example.com. (
    2025112901 ; serial
    3600       ; refresh
    900        ; retry
    604800     ; expire
    86400 )    ; minimum

    IN NS ns1.example.com.
    IN NS ns2.example.com.

; Name Servers
ns1 IN A 1.23.45.11
ns2 IN A 1.23.45.12

; Services
@    IN A 1.23.45.100
www  IN A 1.23.45.100
ftp  IN A 1.23.45.101
EOF
```
```bash
sudo chown -R nsd:nsd /etc/nsd/zones
```

## *9. Sample Reverse Zone*
Use reverse DNS zones only if delegated to you.

```bash
sudo cat > /etc/nsd/zones/45.23.1.in-addr.arpa.zone << 'EOF'
$ORIGIN 45.23.1.in-addr.arpa.
$TTL 3600

@ IN SOA ns1.example.com. hostmaster.example.com. (
    2025112901 ; serial
    3600       ; refresh
    900        ; retry
    604800     ; expire
    86400 )    ; minimum

    IN NS ns1.example.com.
    IN NS ns2.example.com.

11  IN PTR ns1.example.com.
12  IN PTR ns2.example.com.
100 IN PTR www.example.com.
101 IN PTR ftp.example.com.
EOF
```

## *10. Validate & Start Service*

```bash
sudo nsd-checkconf /etc/nsd/nsd.conf
sudo systemctl restart nsd
sudo tail -n 100 -f /var/log/nsd.log
```

## *11. Hidden Primary Concept (Optional Architecture)*
A hidden primary:
1 stores the editable zone files
2 transfers updates to secondaries via TSIG-signed NOTIFY + XFR
3 is not listed in public NS delegation
4 This reduces reconnaissance exposure but does not replace standard security hardening.

