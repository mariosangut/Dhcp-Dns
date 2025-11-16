# DHCP + DNS + DDNS Infrastructure

## Table of contents
- [DHCP + DNS + DDNS Infrastructure](#dhcp--dns--ddns-infrastructure)
  - [Table of contents](#table-of-contents)
  - [1. Project overview](#1-project-overview)
    - [Project Structure](#project-structure)
  - [2. Environment Setup](#2-environment-setup)
    - [Requirements](#requirements)
      - [**Note:**](#note)
      - [Note:](#note-1)
  - [3. Infrastructure Deployment](#3-infrastructure-deployment)
    - [Machine creation](#machine-creation)
  - [4. Simulation phase](#4-simulation-phase)
    - [Machine provisioning](#machine-provisioning)
  - [In summary, this playbook fully configures all services, enables secure communication between DHCP and DNS, and ensures the client is correctly integrated into the network.](#in-summary-this-playbook-fully-configures-all-services-enables-secure-communication-between-dhcp-and-dns-and-ensures-the-client-is-correctly-integrated-into-the-network)
  - [5. Dynamic DNS (DDNS) Configuration](#5-dynamic-dns-ddns-configuration)
  - [5.1. DNS Server – TSIG Key Handling](#51-dns-server--tsig-key-handling)
    - [5.1.1. TSIG Key Generation](#511-tsig-key-generation)
    - [5.1.2. TSIG Key Definition in BIND9](#512-tsig-key-definition-in-bind9)
    - [5.1.3. Zone Configuration (named.conf.local)](#513-zone-configuration-namedconflocal)
    - [5.1.4. Zone Files](#514-zone-files)
      - [Forward zone: `/var/lib/bind/db.mario.test`](#forward-zone-varlibbinddbmariotest)
      - [Reverse zone: `/var/lib/bind/db.192`](#reverse-zone-varlibbinddb192)
    - [5.1.5. BIND9 Verification and Restart](#515-bind9-verification-and-restart)
  - [5.2. DHCP Server – DDNS Configuration](#52-dhcp-server--ddns-configuration)
    - [5.2.1. TSIG Key Import](#521-tsig-key-import)
    - [5.2.2. DHCP Configuration (`dhcpd.conf`)](#522-dhcp-configuration-dhcpdconf)
    - [5.2.3. DHCP Listening Interface](#523-dhcp-listening-interface)
  - [5.3. How the TSIG Key Is Copied from DNS → DHCP (Ansible Workflow)](#53-how-the-tsig-key-is-copied-from-dns--dhcp-ansible-workflow)
    - [1. Ensure `/etc/dhcp` exists on the DHCP server](#1-ensure-etcdhcp-exists-on-the-dhcp-server)
    - [2. Fetch the key from the DNS server](#2-fetch-the-key-from-the-dns-server)
    - [3. Copy the key into the DHCP server](#3-copy-the-key-into-the-dhcp-server)
  - [5.4. Summary of Automated Workflow](#54-summary-of-automated-workflow)
  - [6. Verification \& Evidence](#6-verification--evidence)
    - [6.1. Verify DHCP Server Logs](#61-verify-dhcp-server-logs)
      - [**Command (on DHCP server):**](#command-on-dhcp-server)
      - [**What to look for:**](#what-to-look-for)
      - [**Evidence:**](#evidence)
    - [6.2. Verify DNS Server Logs (BIND9)](#62-verify-dns-server-logs-bind9)
      - [**Command (on DNS server):**](#command-on-dns-server)
      - [**What to look for:**](#what-to-look-for-1)
      - [**Evidence:**](#evidence-1)
    - [6.3. Validate Forward DNS Resolution (A Record)](#63-validate-forward-dns-resolution-a-record)
      - [**Command:**](#command)
      - [**Expected:**](#expected)
      - [**Evidence:**](#evidence-2)
    - [6.4. Validate Reverse DNS Resolution (PTR Record)](#64-validate-reverse-dns-resolution-ptr-record)
      - [**Command:**](#command-1)
      - [**Expected:**](#expected-1)
      - [**Evidence:**](#evidence-3)
    - [6.5. Ensure DNS Zones Were Dynamically Updated](#65-ensure-dns-zones-were-dynamically-updated)
      - [**Command (on DNS server):**](#command-on-dns-server-1)
      - [**Evidence:**](#evidence-4)
    - [6.6. Verify Client Lease Information](#66-verify-client-lease-information)
      - [**Command (on c1):**](#command-on-c1)
      - [**Evidence:**](#evidence-5)

---

## 1. Project overview

This project deploys an environment consisting of a **DNS (BIND9)** server and a **DHCP (ISC-DHCP-Server)** server that work together using the **DDNS (Dynamic DNS)** mechanism.

Thanks to DDNS, the DHCP server automatically registers the following records on the DNS server:

- **A** (name → IP)
- **PTR** (IP → name)

The environment is fully automated using **Vagrant** and **Ansible**, allowing for reproducible, test-ready implementation.

Translated with DeepL.com (free version)

---

### Project Structure
```bash
├── README.md
├── Vagrantfile
├── ansible.cfg
├── ddns_tests.sh
├── docs
│   ├── dhcp-dns.pdf
│   └── evidences.png
├── files
│   ├── dhcp
│   │   ├── dhcpd.conf
│   │   └── listen
│   ├── dns
│   │   ├── db.192
│   │   ├── db.mario.test
│   │   └── named.conf.local
│   └── resolv.conf
├── hosts.ini
└── playbooks
    ├── playbook-setup.yaml
    └── playbook-setup.yaml.bck
```

---

## 2. Environment Setup

### Requirements

- **Vagrant** ≥ 2.4  
- **VirtualBox** ≥ 7.0  
- **Ansible** ≥ 2.15  

#### **Note:**

> This project was developed on a **macOS system with an ARM processor (Apple Silicon)** using the box `bento/debian-12`.  
> If you run this project on a computer with an **AMD or Intel (x86_64)** processor, you may need to replace the box in the `Vagrantfile` with a compatible one such as:
>
> ```ruby
> config.vm.box = "debian/bullseye64"
> ```
Ensure that the chosen box supports your virtualization backend (VirtualBox, Parallels, or VMware) and architecture.

#### Note: 

>To ensure log timestamps are synchronized across all machines, the system timezone is explicitly set during provisioning:
> ```bash
> sudo timedatectl set-timezone Europe/Madrid
> ```

## 3. Infrastructure Deployment

The project provisions **three virtual machines**:

| Hostname | Role        | IP Address         |
| -------- | ----------- | ------------------ |
| `dns`    | DNS Server  | 192.168.58.10      |
| `dhcp`   | DHCP Server | 192.168.58.20      |
| `c1`     | DHCP Client | Dynamic (via DHCP) |

### Machine creation

```bash
vagrant up
```
This command performs:

1. **Creates the VMs** (`dns`, `dhcp`, and `cliente1`) with predefined network:
   - `192.168.58.0/24` 

## 4. Simulation phase
### Machine provisioning

Once the virtual machines are created with `vagrant up`, you can configure the entire DHCP + DNS + DDNS infrastructure by running:

```bash
ansible-playbook -i hosts.ini playbooks/playbook-setup.yaml
```
This command performs the full provisioning workflow, including:

- Connecting to each VM defined in `hosts.ini`
- Installing required packages (`bind9`, `isc-dhcp-server`, `chrony`, `rsyslog`, etc.)
- Creating and validating DNS zones (direct and reverse)
- Generating the TSIG key on the DNS server
- Copying the TSIG key securely to the DHCP server
- Configuring DDNS integration between DHCP and DNS
- Applying the correct permissions in `/var/lib/bind` to support dynamic updates
- Validating DHCP configuration (`dhcpd.conf`) before starting the service
- Restarting and enabling BIND9 and ISC-DHCP-Server
- Preparing the client interface (`eth1`) and requesting a DHCP lease
- Verifying that the A and PTR records are correctly created via dynamic DNS
  
  In summary, this playbook fully configures all services, enables secure communication between DHCP and DNS, and ensures the client is correctly integrated into the network.
---

## 5. Dynamic DNS (DDNS) Configuration

This section describes the complete DDNS configuration process exactly as implemented in this project, based solely on your real configuration files (`dhcpd.conf`, `named.conf.local`, `db.mario.test`, `db.192`, `listen`) and the logic of the Ansible playbook.

---

## 5.1. DNS Server – TSIG Key Handling

### 5.1.1. TSIG Key Generation

The DNS server generates the TSIG security key using:

```bash
tsig-keygen -a hmac-sha256 ddns-key
```

This produces a key stored at:

```
/etc/bind/ddns.key
```

*(The actual secret is omitted and replaced with `[redacted]`.)*

### 5.1.2. TSIG Key Definition in BIND9

The playbook inserts the generated key block into:

```
/etc/bind/named.conf.options
```

The block structure is:

```conf
key "ddns-key" {
    algorithm hmac-sha256;
    secret "[redacted]";
};
```

### 5.1.3. Zone Configuration (named.conf.local)

Your real `named.conf.local` defines both forward and reverse zones and explicitly allows updates via the TSIG key:

```conf
zone "mario.test" IN {
        type master;
        file "/var/lib/bind/db.mario.test";
        allow-update { key "ddns-key"; };
};

zone "58.168.192.in-addr.arpa" IN {
        type master;
        file "/var/lib/bind/db.192";
        allow-update { key "ddns-key"; };
};
```

### 5.1.4. Zone Files

#### Forward zone: `/var/lib/bind/db.mario.test`

```conf
$TTL	604800
@	IN	SOA	dns.mario.test. root.mario.test. (
			      2		; Serial
			 604800		; Refresh
			  86400		; Retry
			2419200		; Expire
			 604800 )	; Negative Cache TTL
@       IN      NS      dns.mario.test.
dns     IN      A       192.168.58.10
dhcp    IN      A       192.168.58.20
```

#### Reverse zone: `/var/lib/bind/db.192`

```conf
$TTL	604800
@	IN	SOA	dns.mario.test. root.mario.test. (
			      1		; Serial
			 604800		; Refresh
			  86400		; Retry
			2419200		; Expire
			 604800 )	; Negative Cache TTL
@	IN	NS	dns.mario.test.
```

### 5.1.5. BIND9 Verification and Restart

The playbook validates syntax and zones using:

- `named-checkconf`
- `named-checkzone mario.test db.mario.test`
- `named-checkzone 58.168.192.in-addr.arpa db.192`

Then restarts BIND9.

---

## 5.2. DHCP Server – DDNS Configuration

### 5.2.1. TSIG Key Import

The playbook securely copies `/etc/bind/ddns.key` from DNS to the DHCP server:

```
/etc/dhcp/ddns.key
```

with permissions `0600 root:root`.

### 5.2.2. DHCP Configuration (`dhcpd.conf`)

Your real DHCP configuration includes:

```conf
include "/etc/dhcp/ddns.key";
option domain-name "mario.test";
option domain-name-servers 192.168.58.10, 8.8.8.8;
default-lease-time 86400;
max-lease-time 691200;
option routers 192.168.58.1;

ddns-update-style interim;
ddns-updates on;
ddns-domainname "mario.test.";
ddns-rev-domainname "in-addr.arpa.";

subnet 192.168.58.0 netmask 255.255.255.0 {
    range 192.168.58.100 192.168.58.200;
    option routers 192.168.58.1;
    option domain-name-servers 192.168.58.10;
    option domain-name "mario.test";
}

zone mario.test. {
    primary 192.168.58.10;
    key "ddns-key";
}

zone 58.168.192.in-addr.arpa. {
    primary 192.168.58.10;
    key "ddns-key";
}
```

### 5.2.3. DHCP Listening Interface

Your `listen` file sets:

```conf
INTERFACESv4="eth1"
```

---

## 5.3. How the TSIG Key Is Copied from DNS → DHCP (Ansible Workflow)

The playbook implements a secure and automated transfer of the TSIG key using three steps:

###  1. Ensure `/etc/dhcp` exists on the DHCP server

```yaml
- name: Ensure /etc/dhcp exists
  ansible.builtin.file:
    path: /etc/dhcp
    state: directory
    owner: root
    group: root
    mode: "0755"
```

### 2. Fetch the key from the DNS server

```yaml
- name: Fetch TSIG key file from DNS
  ansible.builtin.fetch:
    src: /etc/bind/ddns.key
    dest: /tmp/ddns.key
    flat: true
  delegate_to: dns
  run_once: true
```

Explanation:

- `delegate_to: dns` forces the task to run **on the DNS machine**.
- The key is copied **from DNS → local Ansible controller**.
- Stored temporarily at `/tmp/ddns.key`.

### 3. Copy the key into the DHCP server

```yaml
- name: Copy TSIG key into DHCP directory
  ansible.builtin.copy:
    src: /tmp/ddns.key
    dest: /etc/dhcp/ddns.key
    owner: root
    group: root
    mode: "0600"
```

This ensures:

- DHCP and DNS share the same TSIG key  
- Permissions protect the key  
- DDNS updates are authenticated correctly  

---

## 5.4. Summary of Automated Workflow

Using the Ansible playbook:

- DNS generates the TSIG key  
- Key is inserted into BIND9  
- DNS zones are deployed  
- BIND9 is validated and restarted  
- DHCP receives the TSIG key  
- DHCP configuration is deployed  
- DHCP is validated and restarted  
- Client `c1` gets a lease  
- DNS creates A and PTR records dynamically  

---

## 6. Verification & Evidence

All tests performed during the deployment have been documented and captured as evidence.  
To keep the README clean and readable, **all screenshots, terminal outputs, and validation captures are stored in the folder:**

```
docs/evidences/
```

This folder contains:

- DHCP server logs showing lease assignment  
- DNS server logs confirming TSIG authentication and dynamic updates  
- A-record and PTR-record validation using `dig`  
- Client interface and IP assignment verification  
- Service status checks (`bind9`, `isc-dhcp-server`)  

You may refer to these files for full visual proof of:

- Correct DHCP lease issuance  
- Successful forward (A) and reverse (PTR) DNS updates  
- Proper DDNS operation between DNS and DHCP  
- Correct behavior of the client during IP request and registration  

> The Markdown intentionally does **not embed images** to maintain clarity and avoid clutter.  
> All evidences can be consulted directly inside the `docs/evidences/` directory.

---

### 6.1. Verify DHCP Server Logs

The DHCP server logs all dynamic DNS actions (DDNS updates, A records, PTR records, and lease events).

#### **Command (on DHCP server):**
```bash
sudo journalctl -u isc-dhcp-server -n 200
```

or, if using syslog:

```bash
sudo grep dhcp /var/log/syslog
```

#### **What to look for:**
- DHCPREQUEST / DHCPACK exchanges  
- Lines mentioning **“added A record”** or **“added PTR record”**  
- DDNS key authentication messages

#### **Evidence:**

```bash
DHCPDISCOVER from 08:00:27:48:57:ee via eth1
DHCPOFFER on 192.168.58.100 to 08:00:27:48:57:ee (cliente1) via eth1
DHCPREQUEST for 192.168.58.100 (192.168.58.20) from 08:00:27:48:57:ee (cliente1) via eth1
DHCPACK on 192.168.58.100 to 08:00:27:48:57:ee (cliente1) via eth1
Added new forward map from cliente1.mario.test. to 192.168.58.100
Added reverse map from 100.58.168.192.in-addr.arpa. to cliente1.mario.test.
```

---

### 6.2. Verify DNS Server Logs (BIND9)

BIND9 logs all updates applied via DDNS.

#### **Command (on DNS server):**
```bash
sudo journalctl -u bind9 -n 200
```

or, if using syslog:

```bash
sudo grep named /var/log/syslog
```

#### **What to look for:**
- Dynamic updates for A and PTR  
- Successful TSIG authentication  
- Zone writes (`.jnl` updates)

#### **Evidence:**
```bash
2025-11-16T21:15:02.005465+01:00 dns systemd[1]: Started named.service - BIND Domain Name Server.
2025-11-16T21:15:08.541190+01:00 dns named[3567]: client @0xffffae380098 192.168.58.20#38108/key ddns-key: signer "ddns-key" approved
2025-11-16T21:15:08.541437+01:00 dns named[3567]: client @0xffffae380098 192.168.58.20#38108/key ddns-key: updating zone 'mario.test/IN': adding an RR at 'cliente1.mario.test' A 192.168.58.100
2025-11-16T21:15:08.541530+01:00 dns named[3567]: client @0xffffae380098 192.168.58.20#38108/key ddns-key: updating zone 'mario.test/IN': adding an RR at 'cliente1.mario.test' TXT "003d417d8c11fb7f45ec5496c2ea7f228a"
2025-11-16T21:15:08.543702+01:00 dns named[3567]: client @0xffffae380098 192.168.58.20#38108/key ddns-key: signer "ddns-key" approved
2025-11-16T21:15:08.544333+01:00 dns named[3567]: client @0xffffae380098 192.168.58.20#38108/key ddns-key: updating zone '58.168.192.in-addr.arpa/IN': deleting rrset at '100.58.168.192.in-addr.arpa' PTR
2025-11-16T21:15:08.544646+01:00 dns named[3567]: client @0xffffae380098 192.168.58.20#38108/key ddns-key: updating zone '58.168.192.in-addr.arpa/IN': adding an RR at '100.58.168.192.in-addr.arpa' PTR cliente1.mario.test.
2025-11-16T21:15:12.009142+01:00 dns named[3567]: managed-keys-zone: Unable to fetch DNSKEY set '.': timed out
2025-11-16T21:15:12.015516+01:00 dns named[3567]: resolver priming query complete: timed out
2025-11-16T21:16:17.009675+01:00 dns named[3567]: client @0xffffae383898 192.168.58.100#40525 (mario.test): transfer of 'mario.test/IN': AXFR started (serial 3)
2025-11-16T21:16:17.009845+01:00 dns named[3567]: client @0xffffae383898 192.168.58.100#40525 (mario.test): transfer of 'mario.test/IN': AXFR ended: 1 messages, 7 records, 271 bytes, 0.001 secs (271000 bytes/sec) (serial 3)
2025-11-16T22:15:22.007516+01:00 dns named[3567]: resolver priming query complete: timed out
```

---

### 6.3. Validate Forward DNS Resolution (A Record)

From **any machine** (`dns`, `dhcp`, or `c1`):

#### **Command:**
```bash
dig @192.168.58.10 cliente1.mario.test
```

or:

```bash
nslookup cliente1.mario.test 192.168.58.10
```

#### **Expected:**
- The client name resolves to its assigned DHCP IP  
- Example:
  ```
  cliente1.mario.test.   A   192.168.58.xxx
  ```

#### **Evidence:**
```bash
vagrant@dns:~$ dig @192.168.58.10 cliente1.mario.test

; <<>> DiG 9.18.41-1~deb12u1-Debian <<>> @192.168.58.10 cliente1.mario.test
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 58366
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 2604f111a224422501000000691a40f1eb6e0e76121980c7 (good)
;; QUESTION SECTION:
;cliente1.mario.test.           IN      A

;; ANSWER SECTION:
cliente1.mario.test.    3600    IN      A       192.168.58.100

;; Query time: 0 msec
;; SERVER: 192.168.58.10#53(192.168.58.10) (UDP)
;; WHEN: Sun Nov 16 22:24:01 CET 2025
;; MSG SIZE  rcvd: 92
```

---

### 6.4. Validate Reverse DNS Resolution (PTR Record)

#### **Command:**
```bash
dig @192.168.58.10 -x 192.168.58.xxx
```

or:

```bash
nslookup 192.168.58.xxx 192.168.58.10
```

#### **Expected:**
- The assigned IP maps back to `cliente1.mario.test`

Example expected output:
```
xxx.58.168.192.in-addr.arpa.   PTR   cliente1.mario.test.
```

#### **Evidence:**
```bash
vagrant@dns:~$ dig @192.168.58.10 -x 192.168.58.100

; <<>> DiG 9.18.41-1~deb12u1-Debian <<>> @192.168.58.10 -x 192.168.58.100
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 34978
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: e58abfecd350854301000000691a412fa007115fbf340903 (good)
;; QUESTION SECTION:
;100.58.168.192.in-addr.arpa.   IN      PTR

;; ANSWER SECTION:
100.58.168.192.in-addr.arpa. 3600 IN    PTR     cliente1.mario.test.

;; Query time: 0 msec
;; SERVER: 192.168.58.10#53(192.168.58.10) (UDP)
;; WHEN: Sun Nov 16 22:25:03 CET 2025
;; MSG SIZE  rcvd: 117
```

---

### 6.5. Ensure DNS Zones Were Dynamically Updated

Check that the DNS server wrote to the journal files:

#### **Command (on DNS server):**
```bash
sudo ls -l /var/lib/bind
```

You should see:

- `db.mario.test.jnl`  
- `db.192.jnl`  

These files indicate dynamic updates are being recorded.

#### **Evidence:**
```bash
vagrant@dns:~$ sudo ls -l /var/lib/bind
total 16
-rw-r--r-- 1 bind bind 365 Nov 16 21:27 db.192
-rw-r--r-- 1 bind bind 776 Nov 16 21:15 db.192.jnl
-rw-r--r-- 1 bind bind 446 Nov 16 21:27 db.mario.test
-rw-r--r-- 1 bind bind 795 Nov 16 21:15 db.mario.test.jnl
vagrant@dns:~$ 
```

---

### 6.6. Verify Client Lease Information

To confirm the client (`c1`) received an IP from the DHCP server:

#### **Command (on c1):**
```bash
ip a
```

and:

```bash
cat /var/lib/dhcp/dhclient.leases
```

#### **Evidence:**
```bash
vagrant@cliente1:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:bc:d2:7e brd ff:ff:ff:ff:ff:ff
    altname enp0s8
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic eth0
       valid_lft 82047sec preferred_lft 82047sec
    inet6 fd17:625c:f037:2:a00:27ff:febc:d27e/64 scope global dynamic mngtmpaddr 
       valid_lft 86070sec preferred_lft 14070sec
    inet6 fe80::a00:27ff:febc:d27e/64 scope link 
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:48:57:ee brd ff:ff:ff:ff:ff:ff
    altname enp0s9
    inet 192.168.58.100/24 brd 192.168.58.255 scope global dynamic eth1
       valid_lft 82142sec preferred_lft 82142sec
    inet6 fe80::a00:27ff:fe48:57ee/64 scope link 
       valid_lft forever preferred_lft forever
```
```bash
vagrant@cliente1:~$ cat /var/lib/dhcp/dhclient.leases
lease {
  interface "eth1";
  fixed-address 192.168.58.100;
  option subnet-mask 255.255.255.0;
  option routers 192.168.58.1;
  option dhcp-lease-time 86400;
  option dhcp-message-type 5;
  option domain-name-servers 192.168.58.10;
  option dhcp-server-identifier 192.168.58.20;
  option domain-name "mario.test";
  renew 1 2025/11/17 06:48:18;
  rebind 1 2025/11/17 17:15:08;
  expire 1 2025/11/17 20:15:08;
}
```
---




