# 🔍 Cómo crear un túnel de acceso para ver datos de OpenClaw en tiempo real – 3 opciones

> *Monitorea el estado, consumo y salud de tu bot de IA sin exponer tu red.*

---

## 🇪🇸 Español

### 🎯 Objetivo

Una vez que tienes Clawbot (OpenClaw) funcionando en tu VPS, necesitas formas prácticas de:

- Ver el panel de administración.
- Revisar logs y estadísticas de uso.
- Enviar comandos o prompts simples para depuración.
- Monitorear el consumo de recursos (CPU, RAM, sesiones activas).

Aquí te presentamos **tres métodos** para lograrlo, desde el más sencillo hasta el más avanzado.

---

### ⚠️ Nota de seguridad

Ninguno de estos métodos expone puertos peligrosos a internet. Todos usan **túneles cifrados** o **redes privadas virtuales** (Tailscale) o **autenticación vía token**, siempre y cuando hagas uso de las mejores prácticas y blindaje de tu VPS con openclaw, revisa el contenido del reposotorio para más información.

---

## 📌 Método 1: Túnel SSH (acceso al dashboard oficial de OpenClaw)

Este es el método que usamos durante la instalación. Es ideal para administración puntual.

### ¿Cómo funciona?

Creas un túnel SSH desde tu computadora local hacia el VPS, mapeando el puerto remoto `18789` al puerto local `18789`.

### Pasos

1. Abre una terminal en tu PC y ejecuta:

```bash
ssh -N -L 18789:127.0.0.1:18789 root@IP_DE_TU_VPS
```

2. Deja esa terminal abierta.
3. Abre tu navegador y ve a `http://127.0.0.1:18789`.
4. Ingresa el token de autenticación que definiste en la instalación.

> Si aun no defines un token, esto lo haces usando la terminal en tu PC, simplemente usa `ssh-keygen -t ed25519 -C "your_email@example.com"`

### ✅ Ventajas

- Súper seguro (usa el mismo SSH que ya usas).
- No requiere instalar nada extra.
- El panel muestra toda la configuración y logs en tiempo real.

### ❌ Desventajas

- El túnel debe estar activo cada vez que quieras ver el panel.
- No es práctico para compartir con otros usuarios.

---

## 📌 Método 2: Red privada con Tailscale + Dashboard Node.js personalizado

Tailscale crea una red privada virtual (WireGuard) gratuita para hasta 3 usuarios. Podemos exponer un pequeño servidor Node.js que muestre métricas de Clawbot (estado, salud, uso de memoria, etc.) y hasta permita enviar comandos simples.

### ¿Qué necesitas?

- Tailscale instalado en el VPS y en tu PC (o dispositivo móvil).
- Node.js en el VPS (ya viene con Ubuntu).
- Un archivo `server.js` sencillo.

### Pasos

#### 1. Instalar Tailscale en el VPS

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```

Sigue el enlace para autenticar con tu cuenta Google/GitHub.

#### 2. Crear el dashboard de métricas

Crea un directorio y el archivo `server.js`:

```bash
mkdir -p /opt/metrics-dashboard
cd /opt/metrics-dashboard
nano server.js
```

Pega el siguiente código (ajusta la clave secreta):

```javascript
const http = require('http');
const { exec } = require('child_process');

const PORT = 3333;
const AUTH_KEY = 'mi_clave_secreta_cambiala'; // Cámbiala por una segura

const server = http.createServer((req, res) => {
  const url = new URL(req.url, `http://${req.headers.host}`);
  const key = url.searchParams.get('key');

  if (key !== AUTH_KEY) {
    res.writeHead(401, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ error: 'Unauthorized' }));
    return;
  }

  if (url.pathname === '/health') {
    // Endpoint de salud
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ status: 'ok', timestamp: new Date().toISOString() }));
    return;
  }

  if (url.pathname === '/metrics') {
    // Obtener estado completo de OpenClaw (formato JSON)
    exec('docker exec openclaw-gateway openclaw status --json', (error, stdout, stderr) => {
      if (error) {
        res.writeHead(500, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({ error: error.message }));
        return;
      }
      res.writeHead(200, { 'Content-Type': 'application/json' });
      res.end(stdout);
    });
    return;
  }

  if (url.pathname === '/prompt' && req.method === 'POST') {
    // Endpoint para enviar prompts simples al bot (requiere body)
    let body = '';
    req.on('data', chunk => body += chunk);
    req.on('end', () => {
      try {
        const { message } = JSON.parse(body);
        if (!message) throw new Error('Missing message');
        // Ejecuta un comando openclaw (ej: enviar mensaje a través del agente)
        exec(`docker exec openclaw-gateway openclaw agent run --message "${message}"`, (error, stdout) => {
          if (error) {
            res.writeHead(500);
            res.end(JSON.stringify({ error: error.message }));
            return;
          }
          res.writeHead(200);
          res.end(JSON.stringify({ response: stdout }));
        });
      } catch(e) {
        res.writeHead(400);
        res.end(JSON.stringify({ error: e.message }));
      }
    });
    return;
  }

  // Página HTML principal
  res.writeHead(200, { 'Content-Type': 'text/html' });
  res.end(`
    <!DOCTYPE html>
    <html>
    <head><title>Clawbot Metrics</title><meta http-equiv="refresh" content="10"></head>
    <body>
      <h1>📊 Clawbot en tiempo real</h1>
      <pre id="data">Cargando...</pre>
      <script>
        async function load() {
          try {
            const res = await fetch('/metrics?key=${AUTH_KEY}');
            const data = await res.json();
            document.getElementById('data').innerText = JSON.stringify(data, null, 2);
          } catch(err) {
            document.getElementById('data').innerText = 'Error: ' + err;
          }
        }
        load();
        setInterval(load, 10000);
      </script>
    </body>
    </html>
  `);
});

server.listen(PORT, '127.0.0.1', () => {
  console.log(`Dashboard running at http://127.0.0.1:${PORT}`);
});
```

> Nota importante: este servidor es un simple ejemplo donde no estamos considerando la seguridad con la seriedad que amerita, sino mas bien, haciendo una entrega rápida para probar. Lo ideal es que uses variables protegidas, colocadas en otra carpeta, en un archivo .env. Pidele a tu IA que lo mejore.

Guarda y cierra.

#### 3. Ejecutar el servidor en segundo plano

```bash
export AUTH_KEY="mi_clave_secreta_cambiala"
nohup node server.js > dashboard.log 2>&1 &
```

#### 4. Exponer el puerto con Tailscale

```bash
tailscale serve --bg --http=3333 127.0.0.1:3333
```

#### 5. Acceder desde cualquier dispositivo en tu Tailnet

- Obtén la IP de Tailscale del VPS: `tailscale ip` (ej: `100.123.193.3`).
- En tu navegador (con Tailscale activo en tu PC) ve a: `http://100.123.193.3:3333?key=mi_clave_secreta_cambiala`

Verás un panel con los mismos datos que el dashboard oficial, pero más ligero. También puedes consultar los endpoints directamente:

- `http://<IP_TAILSCALE>:3333/health?key=...`
- `http://<IP_TAILSCALE>:3333/metrics?key=...`

### ✅ Ventajas

- No necesitas túnel SSH cada vez.
- Puedes compartir el acceso con otros usuarios de tu Tailnet (hasta 3 gratis).
- El dashboard es personalizable y ligero.

### ❌ Desventajas

- Requiere instalar Tailscale (pero es gratuito y sencillo).
- Hay que mantener el proceso Node.js corriendo (puedes crear un servicio systemd).

---

## 📌 Método 3: Sitio web estático conectado a un repositorio GitHub (acceso restringido)

Este método es ideal si quieres que **Clawbot pueda publicar datos** (por ejemplo, estadísticas diarias) en un sitio web público, pero sin dar acceso de escritura a cualquiera. Usaremos una cuenta de GitHub dedicada exclusivamente para el bot.

### ¿Cómo funciona?

1. Creas un repositorio público en GitHub (por ejemplo, `clawbot-stats`).
2. Configuras un **GitHub Actions** o un **webhook** para que, cuando Clawbot quiera publicar algo, genere un commit automático.
3. El sitio se sirve usando GitHub Pages (gratuito, con HTTPS).
4. Clawbot solo tiene permisos para hacer push a ese repositorio mediante un token de acceso limitado.

### Pasos resumidos

#### 1. Crear cuenta de GitHub para el bot

- Registra una nueva cuenta (ej: `clawbot-publisher`).
- Genera un **Personal Access Token (classic)** con permisos `repo` y `workflow`.

#### 2. Crear repositorio `clawbot-stats`

- Marca como público.
- Habilita GitHub Pages en la rama `main`.

#### 3. En el VPS, instalar git y configurar credenciales

```bash
apt install git -y
git config --global user.name "clawbot-publisher"
git config --global user.email "clawbot@example.com"
```

#### 4. Dar a Clawbot la capacidad de publicar

Puedes crear un pequeño script en el VPS que ejecute `openclaw agent run` para recopilar datos y luego hacer commit + push al repositorio. Ejemplo:

```bash
#!/bin/bash
# /opt/openclaw/publish_stats.sh
cd /opt/openclaw/stats_repo
git pull
docker exec openclaw-gateway openclaw status --json > stats.json
git add stats.json
git commit -m "Actualización automática $(date)"
git push origin main
```

Y luego programar una tarea cron para que se ejecute cada hora.

#### 5. Ver el sitio web

Accede a `https://clawbot-publisher.github.io/clawbot-stats/stats.json` para ver las estadísticas en bruto, o construye un HTML que las visualice.

### ✅ Ventajas

- Totalmente público (puedes compartir el enlace con quien quieras).
- No necesitas tener tu PC encendida ni usar Tailscale.
- El bot no necesita acceso a tu infraestructura privada.

### ❌ Desventajas

- Los datos son públicos (no apto para información sensible).
- Requiere configuración adicional (GitHub Actions, cron, etc.).
- Latencia: los datos no son en tiempo real (se actualizan cada cierto tiempo).

### Mejoras que se pueden hacer?

- Implementar un sistema de auth sencillo pero robusto con supabase.
- Agregar separación entre rutas públicas y privadas.

---

## 🧪 Resumen y recomendación

| Método | Dificultad | Seguridad | Tiempo real | Ideal para... |
|--------|------------|-----------|-------------|----------------|
| **Túnel SSH** | Baja | Máxima | Sí | Administración ocasional. |
| **Tailscale + Node.js** | Media | Alta | Sí | Monitoreo continuo, varios usuarios. |
| **GitHub Pages + cron** | Media-Alta | Pública (solo lectura) | Casi real (según cron) | Publicar estadísticas públicas. |

**Mi recomendación**: comienza con el **túnel SSH** para lo básico. Si necesitas monitoreo constante, instala **Tailscale + dashboard Node.js**. Si quieres compartir estadísticas públicamente, el método de **GitHub Pages** es excelente.

---

## 🔗 Recursos útiles

- [Tailscale – redes privadas gratuitas](https://tailscale.com)
- [GitHub Pages – alojamiento estático gratuito](https://pages.github.com)
- [OpenClaw documentación oficial](https://docs.openclaw.ai)

---

## ❓ ¿Necesitas ayuda con la instalación previa?

Revisa la guía principal:

👉 **[Cómo instalar Clawbot en el VPS más económico de Interserver – Paso a Paso](Como_instalar_Clawbot_(OpenClaw)_en_el_VPS_mas_economico_y_potente_de_Interserver_net.md)**
