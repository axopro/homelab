# Fase 1 – Reglas del firewall (FW01)

Configuración de OPNsense desde la interfaz web: asistente inicial, alias, DHCP y reglas que deciden qué red puede hablar con cuál.

## Acceso a la interfaz web

- Máquina de administración: Ubuntu Desktop conectada a la red interna `GESTION` (IP por DHCP, `10.10.10.x`).
- URL: `https://10.10.10.1`, usuario `root`.

## Asistente inicial (System → Configuration → Wizard)

| Pantalla | Campo | Valor |
|---|---|---|
| General | Hostname / Domain | `FW01` / `homelab.lan` |
| General | DNS | `1.1.1.1` y `8.8.8.8`, sin Override DNS |
| General | Resolver (Unbound) | Activado: el firewall hace de DNS para las redes internas |
| Time Server | Timezone | `Europe/Madrid` |
| WAN | Tipo | DHCP (NAT de VirtualBox) |
| WAN | Block RFC1918 Private Networks | Desmarcado (la WAN está en la red privada `10.0.2.0/24`) |
| WAN | Block bogon networks | Marcado |
| LAN | IP | `10.10.10.1/24` (sin cambios) |

## Alias

Firewall → Aliases.

| Nombre | Tipo | Contenido | Uso |
|---|---|---|---|
| `PRIVADAS` | Network(s) | `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` | Todas las redes privadas: las del lab, la WAN de VirtualBox y la red de casa |

Tras guardar el alias hay que pulsar **Apply** en la lista de aliases. Comprobación desde la consola de FW01 (opción 8, Shell):

```bash
pfctl -t PRIVADAS -T show
```

## DHCP (Services → Dnsmasq DNS & DHCP)

- **DHCP ranges:** LAN `10.10.10.100–200`, OPT2 `10.10.30.100–200`, OPT3 `10.10.99.100–200`.
- **General → Interface:** LAN, OPT2 y OPT3. Si solo está la LAN, el resto de redes no recibe IP.

## Política entre redes

| Red | Interfaz | Puede ir a… | No puede ir a… |
|---|---|---|---|
| Gestión | LAN | Todo | – |
| Servidores | OPT1 | DNS del firewall e internet | Otras redes internas y red de casa |
| Usuarios | OPT2 | DNS del firewall, Servidores e internet | Gestión, Lab y red de casa |
| Lab aislado | OPT3 | Nada | Todo |

Criterios:
- Las reglas se aplican en la interfaz por la que **entra** el tráfico.
- Se evalúan de arriba abajo y gana la primera que coincide.
- Lo que no está permitido está bloqueado (denegación por defecto).
- El firewall es *stateful*: las respuestas a una conexión permitida vuelven sin necesitar regla.

## Reglas (Firewall → Rules)

En todas: Direction `in`, TCP/IP Version `IPv4`.

**LAN (Gestión):** reglas por defecto de OPNsense (anti-lockout y permitir todo).

**OPT1 (Servidores)**

| # | Acción | Protocolo | Origen | Destino | Puerto | Descripción |
|---|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | OPT1 network | This Firewall | DNS (53) | DNS al firewall |
| 2 | Pass | any | OPT1 network | `! PRIVADAS` | any | Internet, no redes internas |

**OPT2 (Usuarios)**

| # | Acción | Protocolo | Origen | Destino | Puerto | Descripción |
|---|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | OPT2 network | This Firewall | DNS (53) | DNS al firewall |
| 2 | Pass | any | OPT2 network | OPT1 network | any | Acceso a servidores |
| 3 | Pass | any | OPT2 network | `! PRIVADAS` | any | Internet, no redes internas |

La regla 2 va antes que la 3 porque Servidores también es una red privada. Se permite `any` porque Active Directory usa muchos puertos; se afinará en la fase de servidores.

**OPT3 (Lab aislado)**

| # | Acción | Protocolo | Origen | Destino | Log | Descripción |
|---|---|---|---|---|---|---|
| 1 | Block | any | OPT3 network | any | Sí | Lab aislado: bloquear todo |

`! PRIVADAS` = destino invertido (casilla **Invert** del campo Destination): cualquier destino que no sea una red privada, es decir, internet.

## Comprobaciones

Con la máquina de pruebas movida a cada red interna en VirtualBox.

| Desde | Prueba | Resultado |
|---|---|---|
| Usuarios (`10.10.30.x`, DHCP) | `ping 8.8.8.8` | Responde |
| Usuarios | `nslookup google.com` | Resuelve |
| Usuarios | `ping 10.10.20.1` | Responde |
| Usuarios | `ping 10.10.10.1` | Bloqueado |
| Usuarios | `ping 192.168.1.1` (router de casa) | Bloqueado |
| Lab (`10.10.99.50`, IP fija) | `ping 8.8.8.8`, `10.10.10.1`, `10.10.30.1` | Bloqueado |

Si el portátil está en una red que bloquea el ping (por ejemplo, la del instituto), la salida a internet se comprueba con `curl -I https://www.google.com` en lugar de `ping 8.8.8.8`.

Los bloqueos del Lab aparecen en **Firewall → Log Files → Live View** en rojo, por la interfaz OPT3 y con la regla *Lab aislado: bloquear todo*.

IP fija en el Lab (Ubuntu Desktop):

```bash
sudo nmcli networking off
sudo ip link set enp0s3 up
sudo ip addr add 10.10.99.50/24 dev enp0s3
sudo ip route add default via 10.10.99.1
```

Volver a DHCP tras cambiar de red:

```bash
sudo ip addr flush dev enp0s3
sudo nmcli networking on
```

Comprobaciones desde la consola de FW01:

```bash
tail -f /var/log/filter/latest.log | grep 10.10.30   # log del firewall en directo
pfctl -vvss | grep -A2 10.10.10.1                     # conexiones activas y regla que las permite
pfctl -vvsr | grep -A3 "^@121 "                       # ver una regla por su número
```

Instantánea de FW01 con esta configuración: `FW01 - reglas ok`.

## Siguiente paso

Fase 2: servidores. Ubuntu Server (SRV-LNX01) y Windows Server con Active Directory (DC01) en la red de Servidores.
