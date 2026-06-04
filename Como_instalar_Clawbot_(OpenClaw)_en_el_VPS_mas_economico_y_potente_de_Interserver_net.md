## 📄 Archivo 1: `Instalacion_Clawbot_Interserver.md`

# 🧠 Cómo instalar Clawbot (OpenClaw) en el VPS más económico y potente de [Interserver](https://www.interserver.net/r/1149900) – Paso a Paso (ESP/ENG)

> *Una guía completa para tener tu propio agente de IA autónomo 24/7 por solo $6 al mes.*

---

## 🇪🇸 Español

### 🎯 ¿Qué vas a lograr?

Al final de esta guía tendrás:

- Un servidor virtual (VPS) con **Ubuntu 26.04** corriendo en Interserver.net.
- **Clawbot (OpenClaw)** instalado y funcionando con Docker.
- Tu bot conectado a **Telegram** y usando **DeepSeek** como modelo de IA.
- Acceso seguro al panel web mediante **túnel SSH** (sin exponer puertos públicos).
- Una base sólida para agregar métricas y monitoreo (ver segunda guía).

---

### 💰 ¿Por qué [Interserver.net](https://www.interserver.net/r/1149900)?

- **Precio**: $6/mes (KVM Linux VPS Slice).
- **Recursos reales**: 2 vCPUs, 3.3 GB RAM, 76 GB SSD.
- **Libertad**: Root completo, elige cualquier SO (nosotros usamos Ubuntu 26.04 LTS).
- **Ubicación**: Datacenter en Los Ángeles, buena latencia para América.

🔗 **Consigue tu VPS aquí:** `https://www.interserver.net/r/1149900`

---

### 📋 Requisitos previos

- Una cuenta en [Interserver.net](https://www.interserver.net/r/1149900) y un VPS activo.
- Acceso SSH a tu VPS (te llegarán las credenciales por correo).
- Si no te llegan, a veces sucede, puedes simplemente reinstalar el sistema operativo y colocarle una clave de root.
- Un token de bot de Telegram (crea uno con [@BotFather](https://t.me/BotFather)).
- Una API Key de DeepSeek (obtenla en [platform.deepseek.com](https://platform.deepseek.com)) o de la IA que estés usando.

---

### 🛠️ Paso a paso

#### 1. Conectar al VPS por SSH

Desde tu terminal (Linux/macOS) o PowerShell (Windows):

```bash
ssh root@IP_DE_TU_VPS
```

*(Reemplaza `IP_DE_TU_VPS` con la IP que te asignó Interserver)*

> Acá en la terminal se te pedirá tu clave de acceso root.

#### 2. Actualizar el sistema e instalar Docker

```bash
apt update && apt upgrade -y
apt install docker.io docker-compose -y
systemctl enable docker --now
```

#### 3. Crear memoria swap adicional (opcional, pero recomendado)

Clawbot requiere compilar algunos módulos y puede agotar la RAM física. Añadimos 5 GB de swap:

```bash
fallocate -l 5G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

#### 4. Instalar Clawbot usando la imagen pre-construida

> **Nota**: Intentamos compilar la imagen localmente, pero el VPS de 3.3 GB RAM no tenía suficiente memoria para el proceso de compilación (error `exit code 137`). La solución oficial es usar la imagen ya compilada desde GitHub Container Registry.

```bash
mkdir -p /opt/openclaw && cd /opt/openclaw
```

Crea el archivo `docker-compose.yml`:

```bash
nano docker-compose.yml
```

Pega el siguiente contenido:

```yaml
services:
  openclaw:
    image: ghcr.io/openclaw/openclaw:latest
    container_name: openclaw-gateway
    restart: unless-stopped
    ports:
      - "127.0.0.1:18789:18789"
    volumes:
      - ./data:/home/node/.openclaw
    environment:
      - NODE_ENV=production
      - OPENCLAW_GATEWAY_TOKEN=TU_TOKEN_SEGURO
    command: ["openclaw", "gateway", "--bind", "loopback"]
```

Reemplaza `TU_TOKEN_SEGURO` por una cadena aleatoria (ej: `openssl rand -hex 16`). Guarda (`Ctrl+O`, `Enter`, `Ctrl+X`).

Levanta el contenedor:

```bash
docker compose up -d
```

#### 5. Configurar Telegram

Dentro del contenedor, ejecuta:

```bash
docker exec -it openclaw-gateway openclaw config set channels.telegram.botToken "TOKEN_DE_TU_BOT"
docker exec -it openclaw-gateway openclaw config set channels.telegram.dmPolicy "allowlist"
docker exec -it openclaw-gateway openclaw config set channels.telegram.allowFrom "[TU_ID_TELEGRAM]"
```

Para obtener tu `TU_ID_TELEGRAM`, envía `/start` a [@userinfobot](https://t.me/userinfobot) en Telegram.

Reinicia el contenedor:

```bash
docker compose restart
```

#### 6. Conectar DeepSeek (modelo rápido y económico)

```bash
docker exec -it openclaw-gateway openclaw config set models.providers.deepseek '{"baseUrl":"https://api.deepseek.com","apiKey":"TU_API_KEY_DEEPSEEK","models":["deepseek-chat"]}'
docker exec -it openclaw-gateway openclaw config set agents.default.model "deepseek/deepseek-chat"
docker compose restart
```

#### 7. Acceso al panel web de Clawbot (por túnel SSH)

El panel está configurado para escuchar solo en `127.0.0.1`. Para acceder desde tu navegador, abre un túnel SSH desde tu PC:

```bash
ssh -N -L 18789:127.0.0.1:18789 root@IP_DE_TU_VPS
```

(Mantén esta terminal abierta). Luego en tu navegador ve a `http://127.0.0.1:18789`. Te pedirá el token que definiste como `TU_TOKEN_SEGURO`. ¡Ya estás dentro!

---

### 🧪 Verificación

Envía un mensaje a tu bot de Telegram. Debería responder usando DeepSeek. Si todo funciona, ¡felicidades! Tienes tu propio asistente de IA funcionando 24/7.

---

> Nota importante: yo suelo mantener una ventana de algun chat de IA abierta para ir llevando a cabo los pasos uno por uno. Así evitamos errores y podemos corregir todo lo que vaya saliendo mal sobre la marcha. Recuerda que siempre puedes contactarme y con gusto te ayudo!

### 📚 Siguiente paso

Para monitorear el rendimiento de tu bot en tiempo real, consulta la guía:

👉 **[Cómo crear un túnel de acceso para ver datos de OpenClaw en tiempo real – 3 opciones](enlace-a-la-segunda-guia.md)**

### 🧠Nota de Seguridad
> Todo lo mencionado arriba es para un rápido despliegue pero a toda esta configuración le falta un rezuerzo de seguridad importante que he detallado en la guía de las mejores prácticas de seguridad en VPS Clawbot.

----

ENGLISH

<details>
<summary>Click to expand</summary>

# 🧠 How to install Clawbot (OpenClaw) on the most affordable and powerful VPS from [Interserver](https://www.interserver.net/r/1149900) – Step by Step (ESP/ENG)

> *A complete guide to have your own 24/7 autonomous AI agent for just $6 per month.*

---

## 🇺🇸 English

### 🎯 What will you achieve?

By the end of this guide you will have:

- A virtual private server (VPS) running **Ubuntu 26.04** on Interserver.net.
- **Clawbot (OpenClaw)** installed and running with Docker.
- Your bot connected to **Telegram** using **DeepSeek** as the AI model.
- Secure access to the web panel via **SSH tunnel** (no public ports exposed).
- A solid foundation to add metrics and monitoring (see the second guide).

---

### 💰 Why [Interserver.net](https://www.interserver.net/r/1149900)?

- **Price**: $6/month (KVM Linux VPS Slice).
- **Real resources**: 2 vCPUs, 3.3 GB RAM, 76 GB SSD.
- **Freedom**: Full root access, choose any OS (we use Ubuntu 26.04 LTS).
- **Location**: Data center in Los Angeles, good latency for the Americas.

🔗 **Get your VPS here:** `https://www.interserver.net/r/1149900`

---

### 📋 Prerequisites

- An account on [Interserver.net](https://www.interserver.net/r/1149900) and an active VPS.
- SSH access to your VPS (credentials will be sent by email).
- If you don't receive them, it sometimes happens; you can simply reinstall the operating system and set a root password.
- A Telegram bot token (create one with [@BotFather](https://t.me/BotFather)).
- A DeepSeek API Key (get it at [platform.deepseek.com](https://platform.deepseek.com)) or from the AI provider you are using.

---

### 🛠️ Step by step

#### 1. Connect to the VPS via SSH

From your terminal (Linux/macOS) or PowerShell (Windows):

```bash
ssh root@YOUR_VPS_IP
```

*(Replace `YOUR_VPS_IP` with the IP assigned by Interserver)*

> Here the terminal will ask for your root password.

#### 2. Update the system and install Docker

```bash
apt update && apt upgrade -y
apt install docker.io docker-compose -y
systemctl enable docker --now
```

#### 3. Create additional swap memory (optional, but recommended)

Clawbot needs to compile some modules and may exhaust physical RAM. We add 5 GB of swap:

```bash
fallocate -l 5G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

#### 4. Install Clawbot using the pre-built image

> **Note**: We tried to build the image locally, but the 3.3 GB RAM VPS did not have enough memory for the compilation process (error `exit code 137`). The official solution is to use the pre-built image from the GitHub Container Registry.

```bash
mkdir -p /opt/openclaw && cd /opt/openclaw
```

Create the `docker-compose.yml` file:

```bash
nano docker-compose.yml
```

Paste the following content:

```yaml
services:
  openclaw:
    image: ghcr.io/openclaw/openclaw:latest
    container_name: openclaw-gateway
    restart: unless-stopped
    ports:
      - "127.0.0.1:18789:18789"
    volumes:
      - ./data:/home/node/.openclaw
    environment:
      - NODE_ENV=production
      - OPENCLAW_GATEWAY_TOKEN=YOUR_SECURE_TOKEN
    command: ["openclaw", "gateway", "--bind", "loopback"]
```

Replace `YOUR_SECURE_TOKEN` with a random string (e.g., `openssl rand -hex 16`). Save (`Ctrl+O`, `Enter`, `Ctrl+X`).

Start the container:

```bash
docker compose up -d
```

#### 5. Configure Telegram

Inside the container, run:

```bash
docker exec -it openclaw-gateway openclaw config set channels.telegram.botToken "YOUR_BOT_TOKEN"
docker exec -it openclaw-gateway openclaw config set channels.telegram.dmPolicy "allowlist"
docker exec -it openclaw-gateway openclaw config set channels.telegram.allowFrom "[YOUR_TELEGRAM_ID]"
```

To get your `YOUR_TELEGRAM_ID`, send `/start` to [@userinfobot](https://t.me/userinfobot) on Telegram.

Restart the container:

```bash
docker compose restart
```

#### 6. Connect DeepSeek (fast and affordable model)

```bash
docker exec -it openclaw-gateway openclaw config set models.providers.deepseek '{"baseUrl":"https://api.deepseek.com","apiKey":"YOUR_DEEPSEEK_API_KEY","models":["deepseek-chat"]}'
docker exec -it openclaw-gateway openclaw config set agents.default.model "deepseek/deepseek-chat"
docker compose restart
```

#### 7. Access the Clawbot web panel (via SSH tunnel)

The panel is configured to listen only on `127.0.0.1`. To access it from your browser, open an SSH tunnel from your PC:

```bash
ssh -N -L 18789:127.0.0.1:18789 root@YOUR_VPS_IP
```

(Keep this terminal open). Then in your browser go to `http://127.0.0.1:18789`. It will ask for the token you set as `YOUR_SECURE_TOKEN`. You are now inside!

---

### 🧪 Verification

Send a message to your Telegram bot. It should reply using DeepSeek. If everything works, congratulations! You have your own 24/7 AI assistant.

---

> Important note: I usually keep an AI chat window open to carry out the steps one by one. This way we avoid errors and can fix anything that goes wrong along the way. Remember you can always contact me and I will gladly help you!

### 📚 Next step

To monitor your bot's performance in real time, see the guide:

👉 **[How to create an access tunnel to view OpenClaw data in real time – 3 options](link-to-second-guide.md)**

### 🧠 Security Note
> Everything mentioned above is for a quick deployment, but this configuration still lacks important security hardening. I have detailed the best security practices for a Clawbot VPS in a separate guide.

</details>
