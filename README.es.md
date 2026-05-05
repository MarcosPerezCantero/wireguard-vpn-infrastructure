# Infraestructura VPN Self-Hosted con WireGuard, Docker y DNS Dinamico

Configuracion completa de una VPN self-hosted usando WireGuard en Raspberry Pi 5, incluyendo bypass de CG-NAT, DNS dinamico con DuckDNS, enrutamiento con iptables MASQUERADE y acceso seguro por RDP a la red local.

> Available in English: [README.md](README.md)

## Descripcion general

Este proyecto documenta el proceso completo de construir una infraestructura de acceso remoto desde cero. El objetivo es conectarse por Escritorio Remoto (RDP) desde cualquier red externa a un PC Windows en la red de casa, usando una VPN WireGuard en una Raspberry Pi 5 como punto de entrada.

La configuracion cubre problemas reales como Carrier-Grade NAT (CG-NAT), direcciones IP dinamicas, conflictos de subredes, enrutamiento con iptables entre interfaces y troubleshooting de firewall.

## Arquitectura

```
Cliente (fuera de casa)
  -> Internet
    -> IP publica dinamica (ISP)
      -> Router (port forward UDP 51820)
        -> Raspberry Pi 5 (WireGuard en Docker)
          -> Tunel cifrado establecido
            -> Cliente envia trafico a 192.168.50.20
              -> Pasa por el tunel WireGuard
                -> Raspberry lo recibe en wg0
                  -> MASQUERADE: cambia origen de 10.8.0.2 a 192.168.50.244
                    -> PC Windows recibe y responde
                      -> Raspberry reenvia respuesta por el tunel
                        -> Cliente recibe
                          -> Conexion RDP establecida
```

## Requisitos

- Raspberry Pi 5 con Raspberry Pi OS
- Router con CG-NAT (probado con Digi Espana)
- Docker y Docker Compose instalados en la Raspberry Pi
- Cuenta en DuckDNS (https://www.duckdns.org)
- Cliente WireGuard en el dispositivo desde el que te conectas (macOS, Windows, iOS, Android)

## 1. Deteccion y bypass de CG-NAT

### Detectar CG-NAT

Accede al panel de administracion del router y revisa el estado WAN. Estas detras de CG-NAT si:

- La IP WAN es privada (empieza por `10.x.x.x`, `100.64-127.x.x` o `172.16-31.x.x`).
- La puerta de enlace predeterminada es una IP privada.

Con CG-NAT, ninguna conexion entrante puede llegar a tu red desde internet.

### Solucion

Contacta con tu ISP y solicita una IPv4 publica dinamica. Con Digi Espana se llama "Conexion Plus". Tras la activacion:

1. Reinicia el router.
2. Verifica que la IP WAN es publica (por ejemplo `79.x.x.x`).
3. La puerta de enlace tambien debe ser una IP publica.

Si la WAN sigue mostrando una IP privada tras reiniciar, contacta con el ISP de nuevo — puede que necesiten activarlo manualmente.

## 2. DNS dinamico con DuckDNS

Como la IP publica es dinamica y puede cambiar en cualquier momento, DuckDNS proporciona un dominio fijo que siempre resuelve a la IP publica actual.

1. Crea una cuenta en [duckdns.org](https://www.duckdns.org).
2. Crea un subdominio (por ejemplo `midominio.duckdns.org`).
3. Guarda tu token.

El contenedor de DuckDNS en Docker se encarga de actualizar la IP automaticamente.

## 3. Cambiar la subred de casa (recomendado)

La mayoria de routers vienen con `192.168.1.0/24` por defecto. Esto causa conflictos de enrutamiento cuando te conectas desde redes que usan el mismo rango (oficinas, hoteles, cafeterias). El trafico destinado a la VPN va por la red local en vez de por el tunel.

Cambia la subred de casa a algo menos comun en la configuracion LAN del router:

| Campo | Valor |
|-------|-------|
| IP del router | `192.168.50.1` |
| Mascara de subred | `255.255.255.0` |
| Rango DHCP | `192.168.50.128` - `192.168.50.254` |
| Puerta de enlace | `192.168.50.1` |
| DNS primario | `192.168.50.1` |

### Actualizar la IP estatica de la Raspberry Pi 5

Raspberry Pi 5 usa NetworkManager. Edita el archivo de conexion:

```bash
sudo nano /etc/NetworkManager/system-connections/NOMBRE_CONEXION.nmconnection
```

Actualiza la seccion `[ipv4]`:

```ini
[ipv4]
method=manual
address1=192.168.50.244/24,192.168.50.1
dns=192.168.50.1
```

Reinicia:

```bash
sudo reboot
```

## 4. Docker Compose — WireGuard y DuckDNS

Crea un directorio para el proyecto:

```bash
mkdir ~/docker && cd ~/docker
```

Crea un archivo `.env` con tus credenciales:

```env
WG_HOST=midominio.duckdns.org
DD_SUBDOMAINS=midominio
DD_TOKEN=tu_token_de_duckdns
```

Crea un archivo `wg_password_hash.env` con el hash de la contrasena para la interfaz web de wg-easy:

```env
PASSWORD_HASH=tu_hash_aqui
```

Crea el `docker-compose.yml`:

```yaml
services:
  wg-easy:
    image: ghcr.io/wg-easy/wg-easy:latest
    container_name: wg-easy
    network_mode: host
    environment:
      LANG: "es"
      WG_HOST: "${WG_HOST}"
      WG_DEFAULT_DNS: "1.1.1.1"
      WG_ALLOWED_IPS: "10.8.0.0/24,192.168.50.0/24"
      WG_POST_UP: "iptables-legacy -t nat -A POSTROUTING -s 10.8.0.0/24 -o wlan0 -j MASQUERADE; iptables-legacy -P FORWARD ACCEPT; iptables -P FORWARD ACCEPT"
      WG_POST_DOWN: "iptables-legacy -t nat -D POSTROUTING -s 10.8.0.0/24 -o wlan0 -j MASQUERADE"
    env_file:
      - wg_password_hash.env
    volumes:
      - ./wireguard:/etc/wireguard
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    restart: unless-stopped

  duckdns:
    image: lscr.io/linuxserver/duckdns:latest
    container_name: duckdns
    environment:
      PUID: 1000
      PGID: 1000
      TZ: "Europe/Madrid"
      SUBDOMAINS: "${DD_SUBDOMAINS}"
      TOKEN: "${DD_TOKEN}"
    restart: unless-stopped
```

Levanta los contenedores:

```bash
sudo docker compose up -d
```

Notas:
- `network_mode: host` significa que WireGuard usa directamente las interfaces de red de la Raspberry Pi. Los puertos no apareceran en `docker ps` — esto es normal.
- Si la Raspberry Pi esta conectada por Wi-Fi, la interfaz es `wlan0`. Si esta por cable, cambia `wlan0` por `eth0` en los comandos `WG_POST_UP` y `WG_POST_DOWN`.

## 5. Port forwarding en el router

En el panel de administracion del router, crea una regla de reenvio de puertos:

| Campo | Valor |
|-------|-------|
| Protocolo | UDP |
| Puerto externo | 51820 |
| IP interna | 192.168.50.244 |
| Puerto interno | 51820 |

## 6. Configuracion del cliente WireGuard

Instala la app de WireGuard en tu dispositivo y crea un tunel:

```ini
[Interface]
PrivateKey = TU_CLAVE_PRIVADA
Address = 10.8.0.2/24
DNS = 1.1.1.1

[Peer]
PublicKey = CLAVE_PUBLICA_DEL_SERVIDOR
PresharedKey = TU_PRESHARED_KEY
AllowedIPs = 10.8.0.0/24, 192.168.50.0/24
Endpoint = midominio.duckdns.org:51820
PersistentKeepalive = 25
```

Notas sobre la configuracion:

- `AllowedIPs = 10.8.0.0/24, 192.168.50.0/24` enruta solo el trafico de la VPN y la red local a traves del tunel. Usa `0.0.0.0/0` para enrutar todo el trafico por la VPN.
- `PersistentKeepalive = 25` mantiene la conexion viva cuando estas detras de NAT.
- `Endpoint` apunta al dominio de DuckDNS que siempre resuelve a la IP publica actual.

## 7. MASQUERADE — Por que es necesario

Sin MASQUERADE, los clientes VPN no pueden comunicarse con los dispositivos de la red local. Esto es lo que ocurre:

1. El cliente envia un paquete a `192.168.50.20` (PC Windows) con IP de origen `10.8.0.2`.
2. El paquete llega a la Raspberry Pi a traves del tunel WireGuard.
3. La Raspberry Pi reenvia el paquete al PC Windows.
4. El PC Windows recibe un paquete de `10.8.0.2`, una red que no conoce, y lo descarta.

Con MASQUERADE:

1. El cliente envia un paquete a `192.168.50.20` con IP de origen `10.8.0.2`.
2. El paquete llega a la Raspberry Pi.
3. La Raspberry Pi cambia la IP de origen a su propia IP (`192.168.50.244`) antes de reenviarlo.
4. El PC Windows recibe un paquete de `192.168.50.244`, responde a la Raspberry Pi.
5. La Raspberry Pi reenvia la respuesta de vuelta por el tunel al cliente.

### El comando explicado

```
iptables-legacy -t nat -A POSTROUTING -s 10.8.0.0/24 -o wlan0 -j MASQUERADE
```

| Parte | Significado |
|-------|-------------|
| `iptables-legacy` | Usa el sistema de firewall legacy (requerido por WireGuard en Docker) |
| `-t nat` | Opera en la tabla NAT (modificacion de IPs) |
| `-A POSTROUTING` | Aplica la regla justo antes de que el paquete salga por la interfaz |
| `-s 10.8.0.0/24` | Solo aplica a paquetes de la red VPN |
| `-o wlan0` | Solo aplica a paquetes que salen por Wi-Fi (cambiar a `eth0` para cable) |
| `-j MASQUERADE` | Sustituye la IP de origen por la IP de la interfaz de salida |

### Hacer las reglas persistentes

Las reglas de iptables se pierden al reiniciar. Dos opciones:

Opcion 1 — Usando wg-easy (recomendada): Usa `WG_POST_UP` y `WG_POST_DOWN` en `docker-compose.yml` (ya incluido en la configuracion anterior).

Opcion 2 — Usando iptables-persistent:

```bash
sudo apt install iptables-persistent
sudo iptables-legacy-save > /etc/iptables/rules.v4
```

## 8. Escritorio remoto (RDP)

Con la VPN conectada, puedes acceder al PC Windows por RDP.

### Configuracion del PC Windows

1. Activar Escritorio remoto: Configuracion -> Sistema -> Escritorio remoto -> Activado.
2. Permitir en el firewall:

```cmd
netsh advfirewall firewall add rule name="Allow RDP" protocol=tcp dir=in localport=3389 action=allow
```

### Conexion desde macOS

En Microsoft Remote Desktop:

- PC name: `192.168.50.20` (IP local del PC Windows)
- Gateway: **None** (importante — no usar Remote Desktop Gateway)
- User account: Tus credenciales de Windows

Usar un Remote Desktop Gateway por error produce el error `0x300005`. Asegurate de que Gateway esta en **None**.

### Verificar conectividad

Antes de conectar por RDP, verifica desde el cliente con la VPN activa:

```bash
# Ping a la Raspberry Pi
ping 192.168.50.244

# Ping al PC Windows
ping 192.168.50.20

# Comprobar puerto RDP
nc -zv 192.168.50.20 3389
```

## 9. Problemas comunes

### No llego a la Raspberry Pi por ping

- Verifica que WireGuard esta corriendo: `sudo docker ps`
- Comprueba el port forwarding en el router (UDP 51820 -> 192.168.50.244)
- Verifica el handshake: `sudo docker exec wg-easy wg show`
- Comprueba el IP forwarding: `cat /proc/sys/net/ipv4/ip_forward` (debe ser `1`)

### Llego a la Raspberry Pi pero no al PC Windows

- Falta la regla de MASQUERADE. Comprueba con: `sudo iptables-legacy -t nat -L POSTROUTING -v`
- Verifica que la regla tiene `pkts > 0` tras enviar trafico. Si `pkts` se queda en 0, la regla no esta coincidiendo.
- Asegurate de que la interfaz de salida es correcta (`wlan0` para Wi-Fi, `eth0` para cable).
- Comprueba la politica FORWARD: `sudo iptables -L FORWARD -v`. Si muestra `policy DROP`, ejecuta `sudo iptables -P FORWARD ACCEPT`.

### iptables vs iptables-legacy

La Raspberry Pi puede tener dos sistemas de firewall funcionando. WireGuard en Docker usa `iptables-legacy`. Docker usa `iptables` (nftables). Si las reglas no funcionan en uno, prueba en el otro. Docker pone la politica FORWARD en DROP por defecto en nftables, lo que bloquea el reenvio de trafico de la VPN.

### Conflicto de subredes

Si la red desde la que te conectas usa el mismo rango que tu red de casa (por ejemplo ambas usan `192.168.1.0/24`), el trafico no pasara por la VPN. El sistema operativo envia los paquetes por la red local en vez de por el tunel. Solucion: cambiar la subred de casa a algo poco comun como `192.168.50.0/24`.

Puedes verificar que el tunel funciona haciendo ping al gateway de la VPN: `ping 10.8.0.1`. Si responde pero las direcciones `192.168.50.x` no, es un conflicto de subredes.

### Las reglas se pierden al reiniciar

Usa `WG_POST_UP` en `docker-compose.yml` o instala `iptables-persistent`.

## 10. Seguridad

- WireGuard no responde a paquetes sin claves validas. Es invisible a escaneos de puertos.
- Nunca compartas las claves privadas (PrivateKey, PresharedKey) ni tu token de DuckDNS.
- Si las claves se comprometen, regeneralas:

```bash
wg genkey | tee client_private.key | wg pubkey > client_public.key
wg genpsk > client_preshared.key
```

- Actualiza la clave publica del cliente en la configuracion del servidor y la clave privada en la configuracion del cliente.
- Regenera el token de DuckDNS desde la web si ha quedado expuesto.

## Comandos utiles

| Comando | Proposito |
|---------|-----------|
| `sudo docker exec wg-easy wg show` | Ver estado de WireGuard y handshakes |
| `sudo iptables-legacy -t nat -L POSTROUTING -v` | Ver reglas NAT y contadores de paquetes |
| `sudo iptables -L FORWARD -v` | Comprobar politica de la cadena FORWARD |
| `curl ifconfig.me` | Ver IP publica actual |
| `nmap -sn 192.168.50.0/24` | Escanear dispositivos en la red local |
| `nc -zv 192.168.50.20 3389` | Comprobar si el puerto RDP es accesible |
| `udp.port == 51820` | Filtro de Wireshark para trafico WireGuard |
