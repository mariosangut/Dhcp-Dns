# README — Servidor DHCP + DNS con Actualización Dinámica (DDNS)

## Tabla de Contenidos
- [1. Descripción del Proyecto](#1-descripción-del-proyecto)  
  - [Estructura del Proyecto](#estructura-del-proyecto)
- [2. Preparación del Entorno](#2-preparación-del-entorno)  
  - [Requisitos](#requisitos)
- [3. Despliegue de la Infraestructura](#3-despliegue-de-la-infraestructura)  
  - [Provisionamiento de Máquinas](#provisionamiento-de-máquinas)
- [4. Configuración de DNS y DHCP](#4-configuración-de-dns-y-dhcp)  
  - [4.1 Servidor DNS (BIND9)](#41-servidor-dns-bind9)
  - [4.2 Servidor DHCP (ISC-DHCP-Server)](#42-servidor-dhcp-isc-dhcp-server)
- [5. Pruebas de DDNS](#5-pruebas-de-ddns)
- [6. Resolución de Problemas](#6-resolución-de-problemas)
- [7. Conclusiones](#7-conclusiones)

---

## 1. Descripción del Proyecto

Este proyecto despliega un entorno formado por un servidor **DNS (BIND9)** y un servidor **DHCP (ISC-DHCP-Server)** que trabajan conjuntamente mediante el mecanismo **DDNS (Dynamic DNS)**.

Gracias a DDNS, el servidor DHCP registra automáticamente en el servidor DNS los registros:

- **A** (nombre → IP)  
- **PTR** (IP → nombre)  

El entorno está completamente automatizado mediante **Vagrant** y **Ansible**, permitiendo una implementación reproducible y lista para pruebas.

---

### Estructura del Proyecto

```
├── Vagrantfile
├── ansible.cfg
├── docs
│   └── dhcp-dns.pdf
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
    ├── playbook-provision.yaml
    └── playbook-setup.yaml
```

---

## 2. Preparación del Entorno

### Requisitos

- **Vagrant ≥ 2.4**
- **VirtualBox ≥ 7.0**
- **Ansible ≥ 2.15**
- Box base recomendada: `bento/debian-12`

En caso de error con procesadores Intel/AMD, usar:

```ruby
config.vm.box = "debian/bullseye64"
```

---

## 3. Despliegue de la Infraestructura

Para crear y configurar toda la infraestructura:

```bash
vagrant up --provision
```

### Provisionamiento de Máquinas

El entorno crea dos máquinas:

| Host | Rol | IP |
|------|------|----|
| `dns` | Servidor BIND9 | 192.168.X.10 |
| `dhcp` | Servidor DHCP | 192.168.X.20 |

Durante el provisionamiento:

- Se instalan BIND9 y DHCP  
- Se copian los archivos desde `/files`  
- Se generan/distribuyen claves TSIG  
- Se habilita la actualización dinámica  
- Se recargan servicios y se validan configuraciones  

---

## 4. Configuración de DNS y DHCP

---

## 4.1 Servidor DNS (BIND9)

### 1. Generar clave TSIG

```bash
sudo tsig-keygen -a hmac-sha256 ddns-key > /etc/bind/ddns.key
```

### 2. Añadir la clave a `named.conf.options`

```bash
key "ddns-key" {
    algorithm hmac-sha256;
    secret "CLAVE_GENERADA";
};
```

### 3. Habilitar actualizaciones en las zonas

En `named.conf.local`:

```bash
zone "mario.test" IN {
    type master;
    file "/etc/bind/db.mario.test";
    allow-update { key "ddns-key"; };
};

zone "192.in-addr.arpa" IN {
    type master;
    file "/etc/bind/db.192";
    allow-update { key "ddns-key"; };
};
```

### 4. Validación y Reinicio

```bash
sudo named-checkconf
sudo named-checkzone mario.test /etc/bind/db.mario.test
sudo systemctl restart bind9
```

---

## 4.2 Servidor DHCP (ISC-DHCP-Server)

### 1. Copiar la clave TSIG a `/etc/dhcp/ddns.key`

Debe ser idéntica a la generada en el DNS.

### 2. Incluirla en `dhcpd.conf`

```bash
include "/etc/dhcp/ddns.key";
```

### 3. Activar DDNS

```bash
ddns-update-style interim;
ddns-domainname "mario.test.";
ddns-rev-domainname "192.in-addr.arpa.";
```

### 4. Configurar las zonas

```bash
zone mario.test. {
    primary 192.168.X.10;
    key "ddns-key";
}

zone 192.in-addr.arpa. {
    primary 192.168.X.10;
    key "ddns-key";
}
```

### 5. Reiniciar el servidor DHCP

```bash
sudo systemctl restart isc-dhcp-server
```

---

## 5. Pruebas de DDNS

### 1. Solicitar IP desde un cliente

```bash
sudo dhclient -v
```

### 2. Consultar el registro DNS

Registro A:

```bash
dig cliente1.mario.test
```

Zona inversa:

```bash
dig -x 192.168.X.Y
```

### 3. Ver los logs

Logs del DNS:

```bash
journalctl -u bind9 -f
```

Logs del DHCP:

```bash
grep DHCP /var/log/syslog
```

---

## 6. Resolución de Problemas

| Error | Causa | Solución |
|-------|--------|-----------|
| `update failed: REFUSED` | Clave TSIG incorrecta | Reemplazar ddns.key por el del DNS |
| `no valid key found` | Falta el include | Añadir `include "/etc/dhcp/ddns.key"` |
| Zonas no se actualizan | Serial incorrecto en DB | Editar manualmente y reiniciar |
| El cliente no usa DNS | resolv.conf incorrecto | Sustituir por el de `/files/resolv.conf` |

---

## 7. Conclusiones

Este proyecto permite comprender y desplegar un entorno donde:

- DHCP y DNS están integrados  
- Las actualizaciones de zona son seguras mediante TSIG  
- Se automatiza todo mediante Ansible  
- El entorno es totalmente reproducible  

Futuras mejoras posibles:

- Añadir pruebas automáticas  
- Verificación post-deploy  
- Automatizar clientes para pruebas continuas  
- Añadir logs detallados por zonas  
