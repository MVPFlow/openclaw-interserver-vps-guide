## 🔒 Guía de Seguridad y Mejores Prácticas para VPS con OpenClaw

> **Basado en auditoría real del sistema** (Debian 12, OpenClaw 2026.4.29)  
> *"Hardening paso a paso para un asistente de IA autónomo y seguro"*

---

### 📋 Índice

- [1. Introducción y filosofía de seguridad](#1-introducción-y-filosofía-de-seguridad)
- [2. Evaluación de riesgos: Modelo de amenazas](#2-evaluación-de-riesgos-modelo-de-amenazas)
- [3. Línea base del sistema (Post-instalación)](#3-línea-base-del-sistema-post-instalación)
- [4. Hardening de SSH (Acceso seguro)](#4-hardening-de-ssh-acceso-seguro)
- [5. Configuración del Firewall (UFW)](#5-configuración-del-firewall-ufw)
- [6. Protección contra fuerza bruta (Fail2ban)](#6-protección-contra-fuerza-bruta-fail2ban)
- [7. Desactivación de servicios innecesarios](#7-desactivación-de-servicios-innecesarios)
- [8. Hardening específico de OpenClaw](#8-hardening-específico-de-openclaw)
- [9. Red privada con Tailscale (recomendado)](#9-red-privada-con-tailscale-recomendado)
- [10. Monitoreo, auditoría y actualizaciones automáticas](#10-monitoreo-auditoría-y-actualizaciones-automáticas)
- [11. Checklist de seguridad](#11-checklist-de-seguridad)

---

## 1. Introducción y filosofía de seguridad

Un asistente de IA como OpenClaw es una herramienta poderosa, pero su naturaleza autónoma y acceso a herramientas del sistema lo convierten en un objetivo atractivo para atacantes si no se asegura adecuadamente. Esta guía te llevará desde una configuración básica hasta un nivel avanzado de *hardening* (endurecimiento), basada en principios fundamentales:

- **Principio de mínimo privilegio (PoLP)**: Otorgar solo los permisos estrictamente necesarios.
- **Defensa en profundidad**: Múltiples capas de seguridad.
- **Cero confianza (*Zero Trust*)**: Verificar explícitamente antes de confiar.
- **Automatización y monitoreo**: Detectar y responder a incidentes rápidamente.

### ¿Por qué es importante?

Los agentes de IA interpretan instrucciones y actúan en consecuencia. Un atacante podría, mediante **inyección de comandos (*prompt injection*)** en un correo o mensaje que el agente procesa, lograr que ejecute comandos maliciosos, exfiltre claves API o acceda a datos sensibles. Una configuración insegura por defecto puede exponer tu servidor a riesgos graves.

---

## 2. Evaluación de riesgos: Modelo de amenazas

**Antes de blindar tu VPS, identifica a qué te enfrentas**:

| Amenaza | Descripción | Impacto potencial |
|---|---|---|
| **Acceso no autorizado por SSH** | Ataques de fuerza bruta o credenciales débiles | Control total del servidor |
| **Explotación de servicios expuestos** | Puertos abiertos innecesarios (p. ej., OpenClaw en `0.0.0.0`) | Acceso no autorizado a la API |
| **Inyección de comandos en el agente** | Instrucciones maliciosas en contenido que el agente procesa | Ejecución remota de comandos, robo de datos |
| **Escalada de privilegios** | Explotación de vulnerabilidades del sistema o del contenedor | Compromiso del host |
| **Fuga de información** | Logs excesivos, errores con claves API, tráfico en texto plano | Exposición de secretos |

**Suposiciones operativas**:
- Cualquier email, página web, PDF o log puede contener instrucciones ocultas.
- El agente intentará ser útil a menos que se le restrinja explícitamente.
- Si el agente puede llamar herramientas, el acceso a esas herramientas es el verdadero límite de seguridad.
- Si el agente tiene acceso de red sin restricciones, la exfiltración es trivial.

---

## 3. Línea base del sistema (Post-instalación)

> **Nota**: Ubuntu 26.04 LTS ya incluye mejoras como `sudo-rs` (reescritura en Rust), AppArmor con 108 perfiles, y actualizaciones de seguridad automáticas habilitadas por defecto.

### Paso 1: Actualizar el sistema

```bash
sudo apt update && sudo apt full-upgrade -y
```

Si el kernel se actualizó, reinicia:

```bash
[ -f /var/run/reboot-required ] && sudo reboot
```

### Paso 2: Instalar paquetes esenciales

```bash
sudo apt install -y ufw fail2ban unattended-upgrades apt-listchanges curl wget git htop net-tools
```

### Paso 3: Configurar actualizaciones automáticas de seguridad

```bash
sudo dpkg-reconfigure -plow unattended-upgrades
```

Verifica que funciona:

```bash
sudo unattended-upgrades --dry-run --debug 2>&1 | head -20
```

### Paso 4: Verificar la hora y zona horaria

```bash
timedatectl
```

---

## 4. Hardening de SSH (Acceso seguro)

El acceso SSH es la puerta de entrada a tu servidor. Un endurecimiento adecuado es crítico.

### 4.1 Crear un usuario no root (si solo existe `root`)

```bash
sudo adduser devops
sudo usermod -aG sudo devops
```

### 4.2 Configurar autenticación por clave SSH

En tu ordenador local, genera un par de claves (si no lo tienes):

```bash
ssh-keygen -t ed25519 -C "tu-email@ejemplo.com"
```

Copia la clave pública al servidor:

```bash
ssh-copy-id devops@IP_DE_TU_VPS
```

**Prueba** que puedes iniciar sesión con la clave antes de continuar:

```bash
ssh devops@IP_DE_TU_VPS
```

### 4.3 Endurecer la configuración de SSH

Crea un archivo de configuración específico (drop-in) para mantener limpio el archivo principal:

```bash
sudo nano /etc/ssh/sshd_config.d/hardened.conf
```

Añade las siguientes líneas:

```bash
# Deshabilitar login root
PermitRootLogin no

# Deshabilitar autenticación por contraseña (solo claves)
PasswordAuthentication no
PubkeyAuthentication yes

# Limitar intentos de autenticación
MaxAuthTries 3
MaxSessions 3

# Deshabilitar forwarding innecesario
X11Forwarding no
AllowTcpForwarding no

# Timeout de inactividad (5 minutos)
ClientAliveInterval 300
ClientAliveCountMax 2

# Restringir a usuarios específicos (cambia "devops" por tu usuario)
AllowUsers devops
```

**Antes de reiniciar, verifica que la configuración no tenga errores de sintaxis**:

```bash
sudo sshd -t
```

Si no muestra errores, reinicia el servicio:

```bash
sudo systemctl restart sshd
```

> 🚨 **IMPORTANTE**: Mantén abierta tu sesión SSH actual mientras pruebas la nueva configuración en una terminal separada. Si algo sale mal, aún puedes recuperar el acceso.

---

## 5. Configuración del Firewall (UFW)

El firewall es tu primera línea de defensa contra accesos no autorizados. La regla de oro: **siempre permite SSH antes de activar el firewall**.

### 5.1 Configurar reglas básicas

```bash
# Establecer políticas por defecto
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Permitir SSH (puerto 22, o el puerto personalizado si lo cambiaste)
sudo ufw allow 22/tcp
# Si cambiaste el puerto SSH, usa: sudo ufw allow 2222/tcp

# Permitir HTTP/HTTPS (si tienes servicios web)
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Limitar intentos de conexión SSH (rate limiting)
sudo ufw limit 22/tcp
```

### 5.2 Activar el firewall

```bash
sudo ufw enable
```

### 5.3 Verificar el estado

```bash
sudo ufw status verbose
```

**Salida esperada**:

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     LIMIT IN    Anywhere
443/tcp                    ALLOW IN    Anywhere
22/tcp (v6)                LIMIT IN    Anywhere (v6)
443/tcp (v6)               ALLOW IN    Anywhere (v6)
```

### ⚠️ Advertencia importante sobre Docker y UFW

Docker manipula directamente `iptables` y, por defecto, **ignora las reglas de UFW** si el puerto está expuesto con `0.0.0.0`. Por eso, en nuestra configuración de OpenClaw usamos `127.0.0.1:18789:18789` en lugar de `0.0.0.0:18789`, asegurando que el puerto no quede expuesto.

---

## 6. Protección contra fuerza bruta (Fail2ban)

Fail2ban analiza logs y bloquea automáticamente IPs con múltiples intentos fallidos, mitigando ataques de fuerza bruta.

### 6.1 Instalar y configurar

```bash
# Crear una copia local de la configuración
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

### 6.2 Configurar protección para SSH

Edita el archivo de configuración:

```bash
sudo nano /etc/fail2ban/jail.local
```

Busca la sección `[sshd]` y ajústala así:

```ini
[sshd]
enabled = true
port    = ssh
filter  = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
findtime = 600
```

> **Explicación**: `maxretry=3` permite 3 intentos fallidos en `findtime=600` segundos (10 minutos), luego bloquea la IP por `bantime=3600` segundos (1 hora).

### 6.3 Iniciar y habilitar Fail2ban

```bash
sudo systemctl restart fail2ban
sudo systemctl enable fail2ban
```

### 6.4 Verificar el estado

```bash
sudo fail2ban-client status sshd
```

**Salida esperada**:

```
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     0
|  `- File list:        /var/log/auth.log
`- Actions
   |- Currently banned: 0
   |- Total banned:     0
   `- Banned IP list:
```

---

## 7. Desactivación de servicios innecesarios

Cada servicio activo amplía la superficie de ataque.

### 7.1 Listar servicios en ejecución

```bash
systemctl list-units --type=service --state=running
```

### 7.2 Desactivar servicios no esenciales (ejemplos comunes)

```bash
sudo systemctl stop cups
sudo systemctl disable cups

sudo systemctl stop avahi-daemon
sudo systemctl disable avahi-daemon

sudo systemctl stop bluetooth
sudo systemctl disable bluetooth
```

### 7.3 Verificar puertos abiertos

```bash
sudo netstat -tulpn | grep LISTEN
```

Solo deben aparecer los puertos que realmente necesitas (SSH, HTTP, HTTPS, y los de tus aplicaciones como OpenClaw en `127.0.0.1`).

---

## 8. Hardening específico de OpenClaw

Basado en la auditoría realizada y en buenas prácticas documentadas.

### 8.1 Configuración actual (evaluada) y mejoras necesarias

| Componente | Estado actual | Riesgo | Acción correctiva |
|---|---|---|---|
| **Gateway bind** | `loopback` en config, pero contenedor arranca con `--bind lan` | Bajo (el puerto está mapeado a 127.0.0.1) | **Aceptable** (no expuesto) |
| **Puertos expuestos** | `127.0.0.1:18789` (correcto) | Ninguno | ✅ Correcto |
| **allowInsecureAuth** | `true` | Medio (riesgo si se expone la UI) | Desactivar: `gateway.controlUi.allowInsecureAuth = false` |
| **trustedProxies** | Vacío | Bajo | Configurar si se usa reverse proxy |
| **Tools de exec** | Sin sandboxing | Esperado (personal assistant) | Aceptable (solo 1 operador) |
| **Telegram DM Policy** | `allowlist` con un solo ID | ✅ Seguro | Mantener |
| **Tailscale mode** | `off` | ✅ Correcto | Mantener (Tailscale en el host) |

### 8.2 Desactivar autenticación insegura

Edita `/opt/openclaw/data/openclaw.json` y cambia:

```json
"gateway": {
  "controlUi": {
    "allowInsecureAuth": false
  }
}
```

### 8.3 Configurar sandboxing (opcional, máxima seguridad)

Si deseas una capa extra de seguridad, puedes limitar las herramientas que el agente puede usar:

```bash
docker exec -it openclaw-gateway openclaw config set tools.fs.workspaceOnly true
docker exec -it openclaw-gateway openclaw config set sandbox.mode "all"
```

### 8.4 Verificar que no hay configuración de Tailscale dentro del contenedor

```bash
grep -i tailscale /opt/openclaw/data/openclaw.json
```

Debe mostrar `"mode": "off"`. Si aparece `"serve"` o `"funnel"`, cámbialo a `"off"`.

### 8.5 Ejecutar auditoría de seguridad de OpenClaw

```bash
docker exec -it openclaw-gateway openclaw security audit
```

Para una auditoría más profunda:

```bash
docker exec -it openclaw-gateway openclaw security audit --deep
```

---

## 9. Red privada con Tailscale (recomendado)

Para acceder al panel web de OpenClaw sin exponer puertos a internet, la mejor práctica es usar una red privada como Tailscale. Tailscale crea una red WireGuard gestionada, cifrada y sin necesidad de abrir puertos públicos.

### 9.1 Instalar Tailscale en el HOST (no en el contenedor)

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

### 9.2 Iniciar sesión y autenticar

```bash
sudo tailscale up
```

Sigue el enlace en tu navegador para autenticarte con Google/GitHub/Microsoft.

### 9.3 Verificar la IP de Tailscale

```bash
tailscale ip
```

> La IP será similar a `100.x.x.x`.

### 9.4 Exponer el puerto de OpenClaw a través de Tailscale

```bash
sudo tailscale serve --bg --http=18789 127.0.0.1:18789
```

### 9.5 Acceder desde tu navegador (con Tailscale activo en tu PC)

- URL: `http://100.x.x.x:18789` (o el nombre mágico `http://vps3421770.tailac0c4c.ts.net:18789`)
- Introduce el token de autenticación de OpenClaw.

### 9.6 Compartir acceso con otros usuarios

- Puedes invitar a otros usuarios a tu **Tailnet** desde https://login.tailscale.com/admin.
- El plan gratuito permite hasta 3 usuarios, suficiente para compartir con colaboradores de confianza.

---

## 10. Monitoreo, auditoría y actualizaciones automáticas

### 10.1 Configurar alertas de login por SSH

Instala y configura `rkhunter` para detección de rootkits:

```bash
sudo apt install rkhunter -y
sudo rkhunter --update
sudo rkhunter --check
```

### 10.2 Monitoreo básico con `auditd`

```bash
sudo apt install auditd -y
sudo systemctl enable auditd --now
```

### 10.3 Verificar logs regularmente

```bash
# Logs de autenticación
sudo tail -f /var/log/auth.log

# Logs del sistema
sudo journalctl -f

# Logs de OpenClaw
docker logs openclaw-gateway --tail 50 -f
```

### 10.4 Asegurar que las actualizaciones automáticas están activas

```bash
sudo systemctl status unattended-upgrades
```

---

## 11. Checklist de seguridad

Imprime esta lista y marca cada ítem al completarlo:

### Nivel 1 – Básico (obligatorio)
- [ ] Sistema operativo actualizado (`apt update && apt upgrade`)
- [ ] Usuario no root creado y con permisos sudo
- [ ] Autenticación SSH por clave configurada
- [ ] Login root por SSH deshabilitado (`PermitRootLogin no`)
- [ ] Autenticación por contraseña SSH deshabilitada (`PasswordAuthentication no`)
- [ ] Firewall UFW activado y configurado
- [ ] Fail2ban instalado y configurado para SSH
- [ ] Servicios innecesarios desactivados
- [ ] Actualizaciones automáticas de seguridad habilitadas

### Nivel 2 – OpenClaw específico
- [ ] Gateway bindeado a `loopback` (puerto expuesto solo en `127.0.0.1`)
- [ ] `allowInsecureAuth` desactivado (`false`)
- [ ] `dmPolicy` de Telegram en `allowlist` con tu ID
- [ ] Tailscale mode en `off` (dentro del contenedor)
- [ ] Token de autenticación seguro y no compartido
- [ ] Auditoría de seguridad ejecutada y revisada (`openclaw security audit`)

### Nivel 3 – Acceso externo seguro (recomendado)
- [ ] Tailscale instalado en el HOST
- [ ] Panel web accesible solo a través de Tailscale (no por IP pública)
- [ ] `tailscale serve` activo para el puerto 18789
- [ ] Usuarios invitados a la Tailnet (si se comparte acceso)

### Nivel 4 – Monitoreo y mantenimiento
- [ ] `rkhunter` instalado y ejecutándose periódicamente
- [ ] `auditd` activo
- [ ] Revisión periódica de logs (`auth.log`, `journalctl`, logs de Docker)
- [ ] Copias de seguridad externas configuradas

---

## 📊 Resumen visual

```
┌─────────────────────────────────────────────────────────────┐
│                      VPS HARDENING                           │
├─────────────────────────────────────────────────────────────┤
│  Nivel 1 (Obligatorio)                                      │
│  ├── Actualizaciones automáticas                             │
│  ├── Usuario no root + SSH key                               │
│  ├── UFW firewall + Fail2ban                                 │
│  └── Servicios innecesarios desactivados                     │
├─────────────────────────────────────────────────────────────┤
│  Nivel 2 (OpenClaw)                                          │
│  ├── Bind a loopback + allowInsecureAuth=false               │
│  ├── Telegram allowlist + Tailscale mode=off                 │
│  └── Seguridad auditada                                      │
├─────────────────────────────────────────────────────────────┤
│  Nivel 3 (Acceso externo)                                    │
│  └── Tailscale en host → panel solo en red privada           │
├─────────────────────────────────────────────────────────────┤
│  Nivel 4 (Monitoreo)                                         │
│  └── rkhunter + auditd + logs periódicos                     │
└─────────────────────────────────────────────────────────────┘
```

---

## ⚠️ Aviso de seguridad final

> Todo lo mencionado en esta guía representa un **endurecimiento significativo** respecto a una configuración por defecto. Sin embargo, ningún sistema es 100% invulnerable. No se recomienda dar acceso a OpenClaw a cuentas personales principales (email, banca, GitHub, etc.) hasta que se haya probado exhaustivamente en un entorno aislado.
