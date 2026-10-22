# Fase 1 – Red y firewall (FW01)

Firewall OPNsense que conecta y separa las cuatro redes del lab y les da salida a internet.

## La máquina

| Parámetro | Valor |
|---|---|
| Sistema | OPNsense 26.7 (amd64), instalación UFS |
| Hipervisor | VirtualBox |
| Tipo de VM | BSD – FreeBSD (64-bit) |
| RAM / CPU / Disco | 1 GB / 1 vCPU / 20 GB (dinámico) |
| Tarjetas de red | 5 × Intel PRO/1000 MT Desktop (82540EM) |

## Interfaces y direccionamiento

| Adaptador VirtualBox | Conectado a | Interfaz OPNsense | Red | IP del firewall | DHCP |
|---|---|---|---|---|---|
| 1 | NAT | WAN (em0) | Internet | 10.0.2.15 (DHCP de VirtualBox) | – |
| 2 | Red interna `GESTION` | LAN (em1) | Gestión | 10.10.10.1/24 | 10.10.10.100 – 10.10.10.200 |
| 3 | Red interna `SERVIDORES` | OPT1 (em2) | Servidores | 10.10.20.1/24 | No (IPs fijas) |
| 4 | Red interna `USUARIOS` | OPT2 (em3) | Usuarios | 10.10.30.1/24 | 10.10.30.100 – 10.10.30.200 |
| 5 | Red interna `LAB` | OPT3 (em4) | Lab aislado | 10.10.99.1/24 | 10.10.99.100 – 10.10.99.200 |

- Cada red usa el `.1` como puerta de enlace (el firewall).
- Las IPs `.2` a `.99` quedan libres para máquinas con IP fija.
- IPv6 desactivado en todas las interfaces.

## Configuración de la VM en VirtualBox

Adaptadores 1–4 desde la interfaz gráfica. El 5 y el modelo de tarjeta, con VBoxManage (VM apagada):

```bash
cd "/c/Program Files/Oracle/VirtualBox"
./VBoxManage.exe modifyvm "FW01" --nic5 intnet --intnet5 "LAB"
./VBoxManage.exe modifyvm "FW01" --nictype1 82540EM --nictype2 82540EM --nictype3 82540EM --nictype4 82540EM --nictype5 82540EM
```

## Configuración de OPNsense (consola)

1. Instalación con el usuario `installer`: teclado español, UFS, disco `ada0`.
2. Contraseña de root cambiada (opción 3).
3. Asignación de interfaces (opción 1): WAN `em0`, LAN `em1`, OPT1 `em2`, OPT2 `em3`, OPT3 `em4`.
4. IP y DHCP de cada interfaz según la tabla (opción 2).
5. Comprobación: ping a `8.8.8.8` (opción 7) con 0 % de pérdida.
6. Instantánea de la VM con esta configuración.

## Siguiente paso

Entrar a la interfaz web (`https://10.10.10.1`) desde una máquina de la red de Gestión y configurar las reglas entre redes.
