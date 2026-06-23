# dns-unbound-installer

Instalador automático de **Unbound DNS** para ISP y VPS. Un solo script, menú interactivo, listo para producción.

---

## ¿Qué instala?

| Componente | Detalle |
|---|---|
| **Unbound 1.22+** | Resolver recursivo puro con DNSSEC (RFC 5011 / 8145 / 8198) |
| **DoT / DoH** | DNS-over-TLS (puerto 853) y DNS-over-HTTPS (puerto 8053) con certificado Let's Encrypt automático |
| **Query logging** | `/var/log/unbound/queries.log` vía rsyslog — retención 90 días |
| **Prometheus** | Métricas en `http://IP:9090` |
| **unbound_exporter** | Estadísticas internas de Unbound |
| **node_exporter** | Métricas del sistema (CPU, RAM, disco, red) |
| **log_exporter** | Top clientes y dominios consultados en tiempo real |
| **UFW** | Firewall: DNS restringido a las redes que configures, SSH siempre abierto |
| **Swap 2 GB** | Protección OOM en VPS con poca RAM |
| **Kernel tuning** | Buffers de red optimizados para alta carga DNS |

---

## Compatibilidad

- Debian 12 (Bookworm)
- Debian 13 (Trixie)
- Ubuntu 22.04 LTS
- Ubuntu 24.04 LTS

Arquitectura: **amd64** (arm64 funciona sin `unbound_exporter`)

---

## Instalación

```bash
wget https://raw.githubusercontent.com/mtandazo35/dns-unbound-installer/main/install-unbound-isp.sh
bash install-unbound-isp.sh
```

> Requiere ejecutar como **root**.

---

## Menú interactivo

Al ejecutar el script aparece un asistente de 2 pasos:

### Paso 1 — Redes con acceso al DNS

Ingresa las IPs o rangos que podrán usar el servidor DNS. Puedes agregar varias.

```
→ Red o IP (o 'todos' / 'listo'): 203.0.113.0/24
  ✓ Agregado: 203.0.113.0/24
→ Red o IP (o 'todos' / 'listo'): 10.0.0.0/8
  ✓ Agregado: 10.0.0.0/8
→ Red o IP (o 'todos' / 'listo'): listo
```

| Entrada | Resultado |
|---|---|
| `203.0.113.0/24` | Rango CIDR IPv4 |
| `10.0.0.5` | IP individual (se convierte a `/32`) |
| `2803:2540::/32` | Rango IPv6 |
| `todos` | Acceso público `0.0.0.0/0` |
| `listo` | Termina la entrada |

### Paso 2 — Dominio para DoH / DoT *(opcional)*

Si tienes un dominio apuntando a la IP del servidor, el instalador obtiene el certificado Let's Encrypt automáticamente y habilita DoT y DoH.

```
→ Dominio (ej: dns.tuempresa.com) [Enter = sin DoH]: dns.tuempresa.com
  ✓ DoH/DoT habilitado → https://dns.tuempresa.com:8053/dns-query
```

Deja en blanco con `Enter` para instalar solo DNS estándar (UDP/TCP puerto 53).

---

## Puertos abiertos tras la instalación

| Puerto | Protocolo | Servicio |
|---|---|---|
| 22 | TCP | SSH |
| 53 | UDP / TCP | DNS (solo redes configuradas) |
| 853 | TCP | DoT — solo si se configuró dominio |
| 8053 | TCP | DoH — solo si se configuró dominio |
| 9090 | TCP | Prometheus (solo redes configuradas) |

---

## Después de instalar

### Verificar que resuelve

```bash
dig @127.0.0.1 google.com A +short
dig @127.0.0.1 cloudflare.com A +dnssec | grep -i " ad"
```

### Ver logs de consultas en tiempo real

```bash
tail -f /var/log/unbound/queries.log
```

### Ver métricas de Prometheus

```
http://IP_SERVIDOR:9090
```

### Consultar via DoH (si se configuró)

```bash
curl -s "https://dns.tuempresa.com:8053/dns-query?name=google.com&type=A" -H "accept: application/dns-json"
```

### Consultar via DoT (si se configuró)

```bash
kdig @dns.tuempresa.com +tls google.com A
```

---

## Seguridad incluida

- **DNSSEC** completo: valida firmas, rechaza respuestas bogus, RFC 5011 auto-rollover de KSK
- **QNAME minimization**: no revela el nombre completo a los servidores raíz
- **RFC 8198 Aggressive NSEC**: reduce consultas upstream y resiste ataques de enumeración
- **Rate limiting**: 1000 QPS por cliente, backoff automático
- **Private address blocking**: bloquea respuestas con IPs RFC 1918 desde Internet
- **Hide identity/version**: no revela versión ni hostname de Unbound
- **UFW**: solo las redes explícitamente autorizadas pueden consultar el DNS

---

## Renovación automática de certificado

El instalador configura un hook de renovación en `/etc/letsencrypt/renewal-hooks/deploy/reload-unbound.sh`. Cuando `certbot renew` renueva el certificado, Unbound recarga los nuevos archivos PEM automáticamente sin interrumpir el servicio.

---

## Reinstalación / actualización

El script es idempotente. Si lo ejecutas sobre una instalación existente:
- Hace backup del `unbound.conf` anterior con timestamp
- Hace backup del `prometheus.yml` anterior con timestamp
- Sobrescribe la configuración con los nuevos valores del menú

```bash
bash install-unbound-isp.sh
```

---

## Requisitos previos

- VPS o servidor con Debian/Ubuntu (ver compatibilidad)
- Acceso root
- Puerto 80 accesible temporalmente si vas a usar DoH/DoT (solo durante la obtención del certificado)
- Registro DNS tipo `A` apuntando al servidor si vas a usar DoH/DoT
