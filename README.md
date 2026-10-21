# 🛡️ Pi-hole en Docker - Bloqueador de anuncios y DNS autohospedado

[![GitHub release](https://img.shields.io/github/release/pihole/pihole.svg?style=flat-square)](https://github.com/pi-hole/pi-hole/releases)
[![Docker Pulls](https://img.shields.io/docker/pulls/pihole/pihole.svg?style=flat-square)](https://hub.docker.com/r/pihole/pihole)
[![License](https://img.shields.io/github/license/pi-hole/pi-hole.svg?style=flat-square)](https://github.com/pi-hole/pi-hole/blob/master/LICENSE)

## 📋 Descripción general

Pi-hole es un servidor DNS que bloquea anuncios, rastreadores y dominios no deseados para toda tu red local. Actúa como servidor DNS para toda tu red: cuando un dispositivo pide resolver un dominio de publicidad o rastreo, Pi-hole responde con una dirección vacía y la conexión nunca llega a salir de casa. Funciona en cualquier dispositivo que use tu red, sin instalar nada en cada móvil, televisor o portátil.

Esta configuración Docker utiliza la imagen oficial `pihole/pihole:2026.10.0` que incluye Pi-hole FTL v6.8 (motor DNS), Pi-hole Web v6.6.1 (interfaz) y Pi-hole Core v6.5 (scripts de gestión), con 17 avisos de seguridad corregidos y base Alpine 3.24.2.

## ✨ Características principales

- 🚫 **Bloqueo de anuncios y rastreadores** a nivel de red para todos los dispositivos
- 🌐 **Servidor DNS autohospedado** con interfaz web renovada y API documentada
- 🔒 **17 avisos de seguridad corregidos** en la release 2026.10.0 (varios de severidad alta)
- ⚙️ **Configuración mediante variables de entorno** `FTLCONF_` y carpetas persistentes
- 📊 **Panel de consultas en tiempo real** para monitorizar dominios bloqueados
- 🔧 **Soporte para DHCP y NTP** opcional dentro del contenedor
- 🐳 **Despliegue sencillo con Docker Compose** usando imagen oficial

## 📋 Requisitos del sistema

- Docker y Docker Compose (en sistemas antiguos, el comando puede ser `docker-compose`)
- Un host Linux con IP fija en tu red local (recomendado para que los dispositivos siempre encuentren el DNS)
- Puertos libres:
  - **53 TCP/UDP** (DNS)
  - **80 TCP** (interfaz web)
  - **443 TCP** (HTTPS, con certificado autofirmado generado por FTL)
- Opcional:
  - **67 UDP** si usas Pi-hole como servidor DHCP
  - **123 UDP** si lo usas como servidor NTP
- **Aviso con Ubuntu**: `systemd-resolved` suele ocupar el puerto 53 en el host. Si el contenedor no arranca por conflicto de puerto, revisa qué proceso lo usa con `sudo ss -lntup | grep :53` antes de tocar nada.

## 🐳 Instalación

### Paso 1: docker-compose.yml

```yaml
# Basado en el template oficial de Pi-hole v6
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:2026.10.0
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "80:80/tcp"
      - "443:443/tcp"
      # Descomenta si usas Pi-hole como servidor DHCP
      # - "67:67/udp"
      # Descomenta si usas Pi-hole como servidor NTP
      # - "123:123/udp"
    environment:
      TZ: 'Europe/Madrid'
      # Cambia esta contraseña por una propia y segura
      FTLCONF_webserver_api_password: 'cambia-esta-contrasena'
      # Con la red bridge por defecto, el DNS debe escuchar en ALL
      FTLCONF_dns_listeningMode: 'ALL'
    volumes:
      # Carpeta persistente con la configuración y las bases de datos
      - './etc-pihole:/etc/pihole'
      # Descomenta solo si tienes ficheros dnsmasq.d personalizados (ver nota)
      # - './etc-dnsmasq.d:/etc/dnsmasq.d'
    cap_add:
      # Necesario solo para DHCP
      - NET_ADMIN
      # Necesario solo si Pi-hole sincroniza la hora del host como cliente NTP
      - SYS_TIME
      # Opcional: da algo más de tiempo de CPU a Pi-hole
      - SYS_NICE
    restart: unless-stopped
```

> **Importante**: La contraseña de ejemplo es solo un marcador. Si no defines `FTLCONF_webserver_api_password`, Pi-hole genera una aleatoria al arrancar. Pon una propia antes de levantar el contenedor.

### Paso 2: Iniciar Pi-hole

```bash
docker compose up -d
# Revisa los logs hasta que el arranque termine
docker compose logs -f pihole
```

### Paso 3: Probar el DNS

```bash
# Desde el propio host, consulta a Pi-hole
dig @127.0.0.1 example.com

# Un dominio de publicidad debería responder con 0.0.0.0 o con la IP de bloqueo
dig @127.0.0.1 doubleclick.net
```

### Acceder a la interfaz web

- **Interfaz web local**: `http://localhost/admin`
- **Desde otro dispositivo**: `http://IP_DEL_HOST/admin`
- **Carpeta de configuración**: `./etc-pihole`

## ⚙️ Configuración

1. **Zona horaria**: Modifica `TZ` en `environment` según tu ubicación (ej. `Europe/Madrid`, `America/Mexico_City`)
2. **Contraseña web**: Cambia `FTLCONF_webserver_api_password` por una contraseña segura y única
3. **Modo de escucha DNS**: `FTLCONF_dns_listeningMode: 'ALL'` permite escuchar en todas las interfaces (necesario con red bridge por defecto)
4. **Persistencia de datos**: El volumen `./etc-pihole:/etc/pihole` guarda configuración, bases de datos y listas de bloqueo
5. **Capacidades**: `NET_ADMIN` (DHCP), `SYS_TIME` (NTP), `SYS_NICE` (prioridad CPU) - ajusta según necesidades
6. **Puertos opcionales**: Descomenta puertos 67/UDP (DHCP) y 123/UDP (NTP) si los vas a usar
7. **dnsmasq.d personalizado**: Si tienes ficheros personalizados, descomenta el volumen `./etc-dnsmasq.d` y añade `FTLCONF_misc_etc_dnsmasq_d: 'true'` en environment

## 🚀 Primeros pasos

1. Entra en la interfaz web (`http://IP_DEL_HOST/admin`) con la contraseña que definiste en `FTLCONF_webserver_api_password`
2. En tu router, configura el DNS de la red para que apunte a la IP fija de tu host
3. Revisa las listas de bloqueo activas y añade las que necesites desde la sección **Adlists**
4. Consulta el **panel de consultas** (Query Log) para comprobar que los dominios bloqueados aparecen como tal
5. **Consejo**: Si la red usa la configuración bridge por defecto de Docker, Pi-hole recibe consultas desde la IP del contenedor y no ve las IPs reales de los clientes. Usar `network_mode: host` resuelve esto, pero en ese caso cambian los puertos y debes ajustar la configuración a tu entorno

## 💡 Casos de uso

- 🏠 **Bloqueo de publicidad en toda la casa**: televisores, móviles y consolas sin instalar extensiones
- 🕵️ **Control de rastreadores**: menos telemetría saliendo de tus dispositivos
- 🧪 **DNS propio para el laboratorio**: resolver nombres de tus servicios autohospedados desde la misma instancia
- 📡 **Servidor DHCP ligero**: Pi-hole también puede repartir direcciones, si necesitas controlar la red desde un único punto

## 🔒 Acceso remoto seguro

> **Recomendación de seguridad**: No expongas la interfaz web de Pi-hole a Internet directamente. Si necesitas acceder desde fuera de casa:
> - Usa una **VPN** (WireGuard, Tailscale, OpenVPN) para conectarte a tu red local
> - O pon la interfaz detrás de un **proxy inverso con autenticación** (Traefik, Nginx Proxy Manager, Authelia)
> - La release 2026.10.0 corrige vulnerabilidades como DNS rebinding y XSS almacenado que podrían explotarse si la interfaz está expuesta

## 🛠️ Gestión y mantenimiento

### Ver estado
```bash
docker compose ps
```

### Ver logs
```bash
docker compose logs -f pihole
```

### Actualizar a la release 2026.10.0
```bash
# Cambia la etiqueta en docker-compose.yml o usa latest
docker compose pull
docker compose up -d
```
> **Etiquetas**: Puedes fijar la versión (`pihole/pihole:2026.10.0`) para actualizar de forma controlada, o usar `latest` para recibir siempre la última. Las notas de cada release detallan los componentes incluidos.

### Detener Pi-hole
```bash
docker compose down
```

### Comprobar la versión en ejecución
```bash
# Dentro del contenedor, muestra las versiones de Core, Web y FTL
docker exec -it pihole pihole -v
```

### Nota sobre migración desde v5
Si vienes de Pi-hole v5 y ya usabas la carpeta `etc-dnsmasq.d`, mantenla montada durante el primer arranque de v6 para que la migración sea completa. Después puedes quitarla.

## 📝 Licencia

Pi-hole está licenciado bajo **EUPL-1.2** (European Union Public License). Consulta el [repositorio oficial](https://github.com/pi-hole/pi-hole/blob/master/LICENSE) para los detalles de licencia de cada componente (FTL, Web, Core).

---

> 📖 **Artículo original**: [Cómo instalar Pi-hole en Docker - Bloqueador de anuncios y DNS autohospedado (release 2026.10.0)](https://genbyte.blogspot.com/2026/10/pi-hole-en-docker-bloqueador-de.html)  
> 🎥 **Vídeo tutorial**: [Canal GENBYTE en YouTube](https://www.youtube.com/@genbyte)  
> ☕ **Apoya el proyecto**: [Ko-fi](https://ko-fi.com/genbyte) | [Newsletter](https://genbyte.blogspot.com/newsletter)