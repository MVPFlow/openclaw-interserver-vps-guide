# Guía de Solución: Error 1006 (Disconnected) en OpenClaw Docker

Esta guía documenta la solución definitiva para el error de desconexión del WebSocket (`1006: no reason`) al desplegar OpenClaw (Clawbot) dentro de contenedores Docker en un VPS, asegurando el acceso seguro al Dashboard.

---

## 🛑 El Problema de Raíz
El error ocurría debido a tres factores concurrentes:
1. **Bloqueo Interno (`openclaw.json`):** El gateway estaba configurado en modo `loopback`, lo que hacía que solo escuchara tráfico nacido dentro del mismo contenedor.
2. **Pisado de Memoria:** Modificar el JSON con el contenedor corriendo no funcionaba, ya que OpenClaw mantiene la configuración en la RAM y sobrescribe el archivo en el disco al apagarse o reiniciarse.
3. **Exposición Insegura:** Exponer el puerto directamente al mundo mediante HTTP plano (`http://IP_PUBLICA:18789`) dejaba el token de autenticación vulnerable a intercepciones.

---

## 🛠️ Solución Paso a Paso (Aplicada)

### Paso 1: Detener el contenedor en frío
Para evitar que OpenClaw pise tus cambios manuales desde la memoria RAM, el primer paso obligatorio es tumbar el servicio por completo:
```bash
docker compose down
```

### Paso 2: Modificar los archivos de configuración
Con el entorno totalmente apagado, edita los siguientes archivos en tu VPS:

1. **Editar `openclaw.json`** (Ubicado en tu volumen `./data/openclaw.json`):
   Busca el bloque `"gateway"` y cambia el parámetro `"bind"` de `"loopback"` a `"lan"` (parámetro oficial para Docker que equivale a escuchar en `0.0.0.0`):
   ```json
   "gateway": {
     "mode": "local",
     "auth": { ... },
     "port": 18789,
     "bind": "lan"
   }
   ```

### Este paso es delicado y pude probarlo y blindar posibles accesos cuando abrimos el tunnel. 

2. **Editar `docker-compose.yml`**:
   Limpia el amarre local en la sección de puertos para permitir conexiones externas controladas:
   ```yaml
   ports:
     - "18789:18789"
   ```
  -> Lo recomendado es obligar a que solo acepte desde tu localhost
  
   ```yaml
  ports:
     - "127.0.0.1:18789"
   ```

### Paso 3: Levantar el contenedor
Arranca el servicio de nuevo para que OpenClaw se vea forzado a leer la nueva configuración del disco:
```bash
docker compose up -d
```

### Paso 4: Crear el puente seguro (Túnel SSH)
Para no enviar tu token maestro por HTTP plano a través de internet, realizamos un redireccionamiento de puertos seguro. 

**En la terminal de tu computadora local** (no en el VPS), ejecuta:
```bash
ssh -N -L 18789:127.0.0.1:18789 usuario@IP_DE_TU_VPS
```
*Deja esta terminal abierta.* Ahora puedes abrir tu navegador web e ingresar de forma 100% segura mediante:
👉 `http://localhost:18789` (El navegador tratará `localhost` como un entorno seguro y el tráfico viajará cifrado por el puerto 22 de SSH).

---

## 🚀 Futuro: Opción B (Producción con HTTPS Nativo)

Si en el futuro adquieres un dominio (ej. `://tudominio.com`) y deseas entrar directo desde cualquier dispositivo sin necesidad de abrir terminales ni túneles SSH, debes implementar un Proxy Inverso automático con **Caddy**.

### Cambios a realizar en el futuro:

1. **Cerrar el puerto público de OpenClaw:** Cambia `ports` por `expose` en el servicio de OpenClaw para que el bot quede oculto del mundo exterior y solo sea accesible internamente por el Proxy.
2. **Agregar el contenedor de Caddy:** Caddy se encargará de gestionar los certificados SSL de Let's Encrypt de forma automática.

Tu nuevo `docker-compose.yml` deberá lucir así:

```yaml
services:
  openclaw:
    image: ghcr.io/openclaw/openclaw:2026.4.29
    container_name: openclaw-gateway
    restart: unless-stopped
    expose:
      - "18789"
    volumes:
      - ./data:/home/node/.openclaw
    environment:
      - NODE_ENV=production

  caddy:
    image: caddy:2-alpine
    container_name: openclaw-proxy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - caddy_data:/data
      - caddy_config:/config
    command: caddy reverse-proxy --from ://tudominio.com --to openclaw:18789

volumes:
  caddy_data:
  caddy_config:
```

Una vez que configures esto con tu dominio apuntando a la IP de tu VPS, podrás entrar directamente a `https://://tudominio.com` de forma nativa y segura.
