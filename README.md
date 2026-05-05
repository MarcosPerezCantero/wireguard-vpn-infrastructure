# wireguard-raspberry-digi
Guía completa para montar una VPN WireGuard en Raspberry Pi 5 con Digi (CG-NAT), DuckDNS y acceso por escritorio remoto (RDP) a tu red local.

# 🏠 VPN WireGuard en Raspberry Pi 5 con Digi + DuckDNS + RDP

Guía completa para montar una VPN WireGuard en una Raspberry Pi 5 con operador Digi (CG-NAT), DNS dinámico con DuckDNS y acceso por escritorio remoto (RDP) a tu red local desde cualquier sitio.

## 📋 Requisitos

- Raspberry Pi 5 con Raspberry Pi OS
- Router Digi (o similar con CG-NAT)
- Docker y Docker Compose instalados en la Raspberry
- Cuenta en [DuckDNS](https://www.duckdns.org)
- Cliente WireGuard en el dispositivo desde el que te conectas (Mac, Windows, iOS, Android)

## 🔍 1. Verificar si tienes CG-NAT

Accede al panel de tu router (normalmente `192.168.1.1`) y ve a **Estado**. Fíjate en:

- **Dirección IP (WAN):** si empieza por `10.x.x.x`, `100.64-127.x.x` o `172.16-31.x.x` estás detrás de CG-NAT.
- **Puerta de enlace predeterminada:** si es una IP privada (por ejemplo `10.0.32.89`), confirma CG-NAT.

Con CG-NAT nadie desde internet puede conectarse a tu red. Necesitas una IP pública.

### Solución con Digi

Contrata **Conexión Plus** (IP dinámica pública). Tras contratar:

1. Reinicia el router.
2. Verifica que la IP WAN es pública (tipo `79.x.x.x`).
3. La puerta de enlace también debe ser pública.

> ⚠️ Si tras reiniciar sigue mostrando IP privada, llama a Digi y diles que tu WAN sigue con IP `10.x.x.x`.

## 🌐 2. DuckDNS — DNS dinámico

Como la IP pública es dinámica (puede cambiar), usamos DuckDNS para tener un dominio fijo que siempre apunte a tu IP actual.

1. Crea una cuenta en [duckdns.org](https://www.duckdns.org).
2. Crea un subdominio (por ejemplo `midominio.duckdns.org`).
3. Guarda tu **token**.

El contenedor de DuckDNS en Docker se encargará de actualizar la IP automáticamente.

## 🔧 3. Cambiar la subred de tu red local (recomendado)

La mayoría de routers usan `192.168.1.0/24` de fábrica. Esto causa conflictos cuando te conectas desde otra red que use el mismo rango (oficinas, cafeterías, hoteles). El tráfico va por la red local en vez de por la VPN.

En el router Digi, ve a **Red → Configuración de LAN** y cambia:

| Campo | Valor |
|-------|-------|
| Dirección IP | `192.168.50.1` |
| Máscara de subred | `255.255.255.0` |
| Piscina DHCP | `192.168.50.128` - `192.168.50.254` |
| Puerta de enlace | `192.168.50.1` |
| DNS primario | `192.168.50.1` |

Guarda y reinicia el router. Para acceder al router ahora será `192.168.50.1`.

### Actualizar IP estática de la Raspberry Pi 5

En Raspberry Pi 5 se usa NetworkManager. Edita el archivo de conexión:

```bash
sudo nano /etc/NetworkManager/system-connections/NOMBRE_CONEXION.nmconnection
```

Cambia la sección `[ipv4]`:

```ini
[ipv4]
method=manual
address1=192.168.50.244/24,192.168.50.1
dns=192.168.50.1
```

Reinicia la Raspberry:

```bash
sudo reboot
```

## 🐳 4. Docker Compose — WireGuard + DuckDNS

Crea un directorio para el proyecto:

```bash
mkdir ~/docker && cd ~/docker
```

Crea un archivo `.env` con tus datos:

```env
WG_HOST=midominio.duckdns.org
DD_SUBDOMAINS=midominio
DD_TOKEN=tu_token_de_duckdns
```

Crea un archivo `wg_password_hash.env` con el hash de la contraseña para la interfaz web de wg-easy:

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

> **Nota:** `network_mode: host` hace que WireGuard use directamente las interfaces de red de la Raspberry. Por eso no verás puertos mapeados en `docker ps`. Si la Raspberry está por Wi-Fi, la interfaz es `wlan0`; si está por cable, usa `eth0` (ajusta el `WG_POST_UP` en consecuencia).

## 🔀 5. Port forwarding en el router

En el router Digi, ve a **Reenvío de NAT** y crea una regla:

| Campo | Valor |
|-------|-------|
| Protocolo | UDP |
| Puerto externo | 51820 |
| IP interna | 192.168.50.244 |
| Puerto interno | 51820 |

## 📱 6. Configuración del cliente WireGuard

Instala la app de WireGuard en tu dispositivo (Mac, Windows, iOS, Android) y crea un túnel con esta configuración:

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

**Notas sobre la configuración:**

- `AllowedIPs = 10.8.0.0/24, 192.168.50.0/24` — Solo el tráfico hacia la red del túnel y tu red local pasa por la VPN. Si quieres que todo el tráfico pase por la VPN, usa `0.0.0.0/0`.
- `PersistentKeepalive = 25` — Mantiene la conexión viva, importante cuando conectas desde redes con NAT.
- `Endpoint` — Apunta a tu dominio de DuckDNS que siempre resuelve a tu IP pública actual.

## 🔐 7. MASQUERADE — Por qué es necesario

Sin MASQUERADE, los clientes VPN no pueden comunicarse con los dispositivos de tu red local. El flujo es:

1. Tu Mac envía un paquete a `192.168.50.20` (PC Windows) con IP de origen `10.8.0.2`.
2. El paquete llega a la Raspberry por el túnel WireGuard.
3. La Raspberry reenvía el paquete al PC Windows.
4. El PC Windows recibe un paquete de `10.8.0.2`, una red que no conoce, y **no sabe cómo responder**. Lo descarta.

Con MASQUERADE:

1. Tu Mac envía un paquete a `192.168.50.20` con IP de origen `10.8.0.2`.
2. El paquete llega a la Raspberry.
3. La Raspberry **cambia la IP de origen** a su propia IP (`192.168.50.244`) antes de reenviarlo.
4. El PC Windows recibe un paquete de `192.168.50.244`, responde a la Raspberry.
5. La Raspberry reenvía la respuesta de vuelta por el túnel al Mac.

### Hacer las reglas persistentes

Las reglas de iptables se pierden al reiniciar. Dos opciones:

**Opción 1 — Desde wg-easy (recomendada):** Usa `WG_POST_UP` y `WG_POST_DOWN` en el `docker-compose.yml` (ya incluido en la configuración anterior).

**Opción 2 — Con iptables-persistent:**

```bash
sudo apt install iptables-persistent
sudo iptables-legacy-save > /etc/iptables/rules.v4
```

### El comando desglosado

```
iptables-legacy -t nat -A POSTROUTING -s 10.8.0.0/24 -o wlan0 -j MASQUERADE
```

| Parte | Significado |
|-------|-------------|
| `iptables-legacy` | Usa el sistema de firewall legacy (requerido por WireGuard en Docker) |
| `-t nat` | Trabaja en la tabla NAT (modificación de IPs) |
| `-A POSTROUTING` | Aplica la regla justo antes de que el paquete salga por la interfaz |
| `-s 10.8.0.0/24` | Solo para paquetes que vengan de la red VPN |
| `-o wlan0` | Solo para paquetes que salgan por Wi-Fi (cambiar a `eth0` si usas cable) |
| `-j MASQUERADE` | Cambia la IP de origen por la IP de la interfaz de salida |

## 🖥️ 8. Escritorio remoto (RDP)

Con la VPN conectada, puedes acceder por RDP al PC Windows de tu red local.

### Requisitos en el PC Windows

1. **Activar Escritorio remoto:** Configuración → Sistema → Escritorio remoto → Activado.
2. **Permitir en el firewall:**

```cmd
netsh advfirewall firewall add rule name="Allow RDP" protocol=tcp dir=in localport=3389 action=allow
```

### Conexión desde Mac

En Microsoft Remote Desktop:

- **PC name:** `192.168.50.20` (IP del PC Windows)
- **Gateway:** **None** (importante, no usar Gateway)
- **User account:** Tu usuario de Windows

> ⚠️ Si usas un Remote Desktop Gateway por error, recibirás el error `0x300005`. Asegúrate de que Gateway está en **None**.

### Verificar conectividad

Antes de conectar por RDP, comprueba desde el Mac con la VPN activa:

```bash
# Ping a la Raspberry
ping 192.168.50.244

# Ping al PC Windows
ping 192.168.50.20

# Puerto RDP abierto
nc -zv 192.168.50.20 3389
```

## ⚠️ 9. Problemas comunes

### No llego a la Raspberry por ping

- Verifica que WireGuard está corriendo: `sudo docker ps`
- Comprueba el port forwarding en el router (UDP 51820 → 192.168.50.244)
- Verifica el handshake: `sudo docker exec wg-easy wg show`

### Llego a la Raspberry pero no al PC Windows

- Falta MASQUERADE. Ejecuta las reglas de iptables.
- Comprueba con `sudo iptables-legacy -t nat -L POSTROUTING -v` que la regla existe y tiene `pkts > 0`.
- Asegúrate de que la interfaz es correcta (`wlan0` para Wi-Fi, `eth0` para cable).
- Verifica que FORWARD está en ACCEPT: `sudo iptables -P FORWARD ACCEPT`

### iptables vs iptables-legacy

La Raspberry puede tener dos sistemas de firewall. WireGuard en Docker usa `iptables-legacy`. Docker usa `iptables` (nftables). Si las reglas no funcionan en uno, prueba en el otro. El comando `iptables -L FORWARD -v` puede mostrar `policy DROP` (puesto por Docker) que bloquea el tráfico de la VPN.

### Conflicto de subredes

Si la red donde te conectas usa el mismo rango que tu casa (por ejemplo `192.168.1.0/24`), el tráfico no pasa por la VPN. El sistema operativo envía los paquetes por la red local. Solución: cambiar la subred de casa a algo poco común como `192.168.50.0/24`.

### Las reglas se pierden al reiniciar

Usa `WG_POST_UP` en el docker-compose.yml o instala `iptables-persistent`.

## 🔒 10. Seguridad

- **WireGuard** es muy seguro. No responde a paquetes sin las claves correctas, es invisible a escaneos de puertos.
- **Nunca compartas** tus claves privadas (PrivateKey, PresharedKey) ni tu token de DuckDNS.
- Si las claves se comprometen, regenera con:

```bash
wg genkey | tee client_private.key | wg pubkey > client_public.key
wg genpsk > client_preshared.key
```

- Actualiza la clave pública en el servidor y la privada en el cliente.
- Regenera el token de DuckDNS desde su web si se ha expuesto.

## 🌍 Flujo completo de conexión

```
Mac (fuera de casa)
  → Internet
    → IP pública Digi (79.x.x.x)
      → Router Digi (port forward UDP 51820)
        → Raspberry Pi (WireGuard en Docker)
          → Túnel cifrado establecido
            → Mac envía tráfico a 192.168.50.20
              → Pasa por el túnel WireGuard
                → Raspberry lo recibe en wg0
                  → MASQUERADE: cambia origen de 10.8.0.2 a 192.168.50.244
                    → PC Windows recibe y responde
                      → Raspberry reenvía respuesta por el túnel
                        → Mac recibe
                          → RDP funciona ✅
```

## 📝 Herramientas útiles

- **Wireshark:** para depurar tráfico WireGuard, filtra por `udp.port == 51820`
- **Verificar IP pública:** `curl ifconfig.me`
- **Ver estado WireGuard:** `sudo docker exec wg-easy wg show`
- **Ver reglas NAT:** `sudo iptables-legacy -t nat -L POSTROUTING -v`
- **Escanear red local:** `nmap -sn 192.168.50.0/24` o `arp -a`
