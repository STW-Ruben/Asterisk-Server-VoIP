# 📞 Asterisk PBX — Voz, Video y Mensajería de Texto

Configuración completa de Asterisk 20 con **PJSIP** lista para producción: voz, videollamadas, mensajes de texto SIP, buzón de voz, colas ACD, IVR, conferencias, TLS/SRTP y protección anti fuerza-bruta con Fail2Ban.

Diseñado para que **cualquier dispositivo pueda registrarse desde cualquier red** (no solo LAN local) usando un dominio dinámico (DuckDNS, No-IP, etc.) o IP pública fija.

## ⬇️ Clonar el repositorio

```bash
git clone https://github.com/STW-Ruben/Asterisk-Server-VoIP.git
cd Asterisk-Server-VoIP
```

## 📋 Características

- **18 extensiones** organizadas en 3 departamentos (1000-1005, 2000-2005, 3000-3005)
- **Voz**: Opus, G.722, G.711 (a-law/u-law), GSM
- **Video**: VP8, H.264, H.265
- **Mensajería**: SIP MESSAGE (texto plano entre extensiones)
- **Cifrado**: TLS para señalización + SRTP para medios
- **Buzón de voz** con notificación, **colas ACD**, **IVR**, **salas de conferencia**
- **Fail2Ban** preconfigurado contra ataques de registro SIP

## 🗂️ Estructura del repositorio

```
etc/asterisk/
├── pjsip.conf          Extensiones SIP, transportes (UDP/TCP/TLS)
├── extensions.conf     Dialplan: llamadas, IVR, colas, texto, buzón
├── voicemail.conf      Buzones de voz para las 18 extensiones
├── queues.conf         Colas ACD por departamento
├── confbridge.conf     Salas de conferencia con video
├── features.conf       Transferencias, parking, grabación
├── rtp.conf             Puertos RTP, ICE/STUN
├── musiconhold.conf    Música en espera por departamento
├── modules.conf        Desactiva chan_sip, fuerza módulos PJSIP
├── manager.conf        AMI (acceso administrativo)
├── logger.conf         Configuración de logs
└── cdr.conf             Registro de llamadas

etc/fail2ban/
├── jail.d/asterisk.conf      Jail anti fuerza-bruta
└── filter.d/                 (usa el filtro asterisk.conf de fábrica)

scripts/
└── install.sh           Instalador automático (ver abajo)
```

## 🚀 Instalación

> ⚠️ Este repo asume que **Asterisk 20 con soporte PJSIP ya está instalado** en el servidor (compilado con `--with-pjproject-bundled` o equivalente). Si todavía no tienes Asterisk instalado, instálalo primero y vuelve aquí.

### Opción A — Automática con `install.sh` (recomendada)

`install.sh` automatiza todo el proceso que se hizo manualmente al construir este proyecto: copia los `.conf`, reemplaza los placeholders con tu dominio, genera contraseñas seguras nuevas para las 18 extensiones, genera el certificado TLS con Let's Encrypt, corrige los permisos de la clave privada, configura Fail2Ban y abre los puertos en UFW.

```bash
git clone https://github.com/STW-Ruben/Asterisk-Server-VoIP.git
cd Asterisk-Server-VoIP
sudo ./scripts/install.sh
```

Te pedirá dos datos:
- **Dominio** (ej. `tudominio.duckdns.org`) — necesario para TLS
- **Email** — para el registro de Let's Encrypt

Al terminar, las contraseñas generadas quedan guardadas en `/root/asterisk-extension-passwords.txt` (con permisos `600`, solo legible por root).

### Opción B — Manual paso a paso

Si prefieres entender o controlar cada paso:

```bash
git clone https://github.com/STW-Ruben/Asterisk-Server-VoIP.git
cd Asterisk-Server-VoIP

# 1. Reemplaza los placeholders con tus datos reales
#    CHANGE_ME_PUBLIC_IP_OR_DOMAIN -> tu IP pública o dominio
#    CHANGE_ME_DOMAIN              -> tu dominio (para TLS)
#    CHANGE_ME_XXXX                -> contraseña de cada extensión
sed -i 's/CHANGE_ME_PUBLIC_IP_OR_DOMAIN/tudominio.duckdns.org/g' etc/asterisk/pjsip.conf
sed -i 's/CHANGE_ME_DOMAIN/tudominio.duckdns.org/g' etc/asterisk/pjsip.conf

# 2. Copia la configuración al servidor
sudo cp etc/asterisk/*.conf /etc/asterisk/
sudo cp etc/fail2ban/jail.d/asterisk.conf /etc/fail2ban/jail.d/
sudo chown asterisk:asterisk /etc/asterisk/*.conf

# 3. Certificado TLS (Let's Encrypt, requiere dominio apuntando al servidor)
sudo apt install -y certbot
sudo systemctl stop asterisk
sudo certbot certonly --standalone -d tudominio.duckdns.org
# Dale acceso de lectura a Asterisk sobre la clave privada
sudo chmod 755 /etc/letsencrypt/archive /etc/letsencrypt/live
sudo chown root:asterisk /etc/letsencrypt/archive/tudominio.duckdns.org/privkey*.pem
sudo chmod 640 /etc/letsencrypt/archive/tudominio.duckdns.org/privkey*.pem

# 4. Iniciar
sudo systemctl start asterisk
sudo systemctl restart fail2ban
sleep 5
asterisk -rx 'pjsip show endpoints'   # debe mostrar 18 objetos
asterisk -rx 'pjsip show transports'  # debe mostrar udp, tcp y tls
```

## ⚠️ Notas críticas aprendidas en producción

1. **No uses templates con nombres distintos** (`endpoint-tpl`, `1002-auth`, `1002-aor`) — el motor *sorcery* de PJSIP puede rechazar objetos silenciosamente sin loguear el error. Usa el mismo nombre de sección para los tres bloques (`[1002]` repetido), tal como está en este repo.
2. **Evita acentos/tildes** en los archivos `.conf` — un encoding corrupto en un comentario puede romper el parser de sorcery para todo lo que viene después en el archivo, sin mensaje de error claro.
3. **TLS no es recargable** (`pjsip reload` no aplica cambios de certificado) — requiere `systemctl restart asterisk` completo.
4. **Let's Encrypt restringe permisos** de `privkey.pem` a solo `root` — Asterisk corre como usuario `asterisk` y no podrá leer el certificado hasta que ajustes permisos (ver paso 3 arriba).
5. **UFW puede neutralizar Fail2Ban** silenciosamente — si usas UFW, asegúrate que la cadena `f2b-asterisk` esté **antes** que `ufw-before-input` en `INPUT` (usa `chain=INPUT` en la acción de Fail2Ban, como en este repo) y que incluya tanto TCP como UDP.
6. **Fail2Ban necesita `backend=polling`** en este setup — Asterisk no escribe estos logs en el journal de systemd por defecto, así que el backend `systemd` no detecta nada.

## 🌐 Por qué necesitas una IP pública

Este PBX está diseñado para que **cualquier extensión pueda registrarse desde cualquier red** (casa, datos móviles, oficina remota, otro país) — no solo desde la LAN donde vive el servidor. Eso solo es posible si el servidor tiene una **IP pública accesible desde Internet**.

Si Asterisk corre detrás de un router doméstico con IP privada (NAT), los clientes externos nunca podrán completar el `REGISTER` ni establecer el canal RTP de voz/video, sin importar qué tan bien esté configurado `pjsip.conf`. Las opciones son:

1. **VPS con IP pública dedicada** (recomendado, es como se probó este repo) — DigitalOcean, Linode, Vultr, AWS Lightsail, etc.
2. **Servidor propio + IP pública fija de tu ISP** + port forwarding en el router
3. **Servidor propio + IP dinámica** + dominio DDNS (DuckDNS, No-IP) + port forwarding

### Configuración usada en este repo (DigitalOcean)

Este proyecto fue probado en un **Droplet de DigitalOcean** con Ubuntu 22.04/24.04:

- Cualquier Droplet trae IP pública asignada por defecto, sin NAT ni configuración adicional — ideal para esto.
- El firewall de DigitalOcean (Cloud Firewall) es independiente del firewall del sistema operativo (UFW). Si usas el firewall de DigitalOcean además de UFW, asegúrate de abrir los mismos puertos en **ambos lados**:
  - `22/tcp` (SSH)
  - `5060/udp` y `5060/tcp` (SIP)
  - `5061/tcp` (SIP sobre TLS)
  - `10000-20000/udp` (RTP — voz y video)
- Apunta tu dominio DDNS (ej. DuckDNS) a la IP pública del Droplet, ya que esta no cambia salvo que destruyas y recrees la máquina.

```bash
# Verifica tu IP pública asignada
curl -4 ifconfig.me
```

## 🔐 Certificado TLS — por qué y cómo

El bloque `[transport-tls]` en `pjsip.conf` cifra la señalización SIP (usuario/contraseña, headers) para que no viaje en texto plano por Internet. Sin esto, cualquiera que intercepte el tráfico entre tu cliente y el servidor podría ver las credenciales de la extensión.

### Por qué se usó Let's Encrypt (gratis) en vez de un certificado autofirmado

Un certificado autofirmado funciona técnicamente, pero **cada cliente SIP** (Linphone, Zoiper, etc.) mostrará una advertencia de seguridad y, en muchos casos, **rechazará la conexión TLS por defecto** a menos que instales manualmente el certificado de tu CA en cada dispositivo. Con Let's Encrypt, el certificado es reconocido automáticamente por todos los clientes sin configuración extra — pero **requiere que tengas un dominio** (no funciona solo con IP).

### Requisitos para generar el certificado

1. Un dominio o subdominio que **apunte a la IP pública de tu servidor** (en este proyecto: DuckDNS gratuito → `chivoisland.duckdns.org`)
2. El puerto **80/tcp** temporalmente libre (certbot en modo `standalone` lo usa para validar el dominio)
3. Asterisk **detenido** durante la generación (libera el puerto 80, que normalmente no usa, pero por si hay otro servicio ahí)

```bash
apt install -y certbot
systemctl stop asterisk
certbot certonly --standalone -d tudominio.duckdns.org
systemctl start asterisk
```

### El problema de permisos que casi siempre aparece

Let's Encrypt guarda la clave privada (`privkey.pem`) con permisos `600` y dueño `root`. Asterisk corre como usuario `asterisk`, así que **no podrá leer la clave** hasta que ajustes los permisos manualmente — y si no lo haces, el transporte TLS falla en silencio (no aparece en `pjsip show transports`, y el log no muestra un error obvio).

```bash
chmod 755 /etc/letsencrypt/archive /etc/letsencrypt/live
chown root:asterisk /etc/letsencrypt/archive/tudominio.duckdns.org/privkey*.pem
chmod 640 /etc/letsencrypt/archive/tudominio.duckdns.org/privkey*.pem
systemctl restart asterisk    # TLS no es recargable, requiere restart completo
```

### Renovación automática

Los certificados de Let's Encrypt expiran cada 90 días. Certbot instala un timer/cron automático, pero como Asterisk necesita permisos especiales sobre la clave cada vez que se renueva, agrega un hook:

```bash
cat > /etc/letsencrypt/renewal-hooks/deploy/asterisk-permissions.sh << 'EOF'
#!/bin/bash
DOMAIN="tudominio.duckdns.org"
chmod 755 /etc/letsencrypt/archive /etc/letsencrypt/live
chown root:asterisk /etc/letsencrypt/archive/$DOMAIN/privkey*.pem
chmod 640 /etc/letsencrypt/archive/$DOMAIN/privkey*.pem
systemctl restart asterisk
EOF
chmod +x /etc/letsencrypt/renewal-hooks/deploy/asterisk-permissions.sh
```

Este script se ejecuta automáticamente cada vez que certbot renueva el certificado, sin que tengas que acordarte de hacerlo a mano.

## 🔐 Seguridad

- Cambia **todas** las contraseñas `CHANGE_ME_XXXX` antes de exponer el servidor a Internet
- Fail2Ban bloquea automáticamente IPs tras 3 intentos fallidos en 5 minutos (ban de 24h)
- Activa TLS para evitar contraseñas en texto plano en la red
- Si usas DigitalOcean u otro proveedor cloud, configura también su Cloud Firewall, no solo UFW

## 📱 Conectar un cliente (Linphone, Zoiper, etc.)

| Campo | Valor |
|---|---|
| Servidor | tu dominio o IP pública |
| Puerto | `5060` (UDP/TCP) o `5061` (TLS) |
| Usuario | número de extensión (ej. `1002`) |
| Contraseña | la que configuraste en `pjsip.conf` |

## 📜 Licencia

Úsalo libremente, adaptado a tu propia infraestructura.
