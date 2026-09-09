# OpenClaw en MOS

Guía de referencia para preparar la futura migración de **OpenClaw desde Tower (Unraid) a MOS**.

> **Objetivo:** conservar estado, memoria, configuración y workspace, pero aprovechar la migración para adoptar el layout Docker actual de OpenClaw y evitar arrastrar decisiones antiguas de la plantilla de Unraid.

Última revisión: **2026-09-09**

---

## 1. Situación actual en Tower

La instalación actual usa:

```text
Imagen:
ghcr.io/openclaw/openclaw:2026.4.24

Red:
bridge

Puerto:
18789/tcp

Usuario:
root
```

Mounts actuales:

```text
/mnt/ssd_system/appdata/openclaw/config
    → /root/.openclaw

/mnt/ssd_system/appdata/openclaw/workspace
    → /home/node/clawd

/mnt/ssd_system/appdata/openclaw/projects
    → /projects

/mnt/ssd_system/appdata/openclaw/homebrew
    → /home/linuxbrew/.linuxbrew

/mnt/ssd_system/appdata/obsidian/Desktop/openClaw
    → /data/obsidian
```

El contenedor actual también crea automáticamente un `openclaw.json` si no existe y habilita una configuración antigua de acceso LAN.

### Importante

No copiar contraseñas, tokens ni API keys a este repositorio.

Antes del cutover se deben **rotar/regenerar** los secretos que hayan quedado expuestos durante pruebas o documentación.

---

## 2. Objetivo en MOS

La instalación nueva debe seguir el layout Docker actual de OpenClaw.

Esquema recomendado:

```text
MOS
└── /mnt/.../appdata/openclaw/
    ├── state/
    │   └── → /home/node/.openclaw
    │
    └── auth/
        └── → /home/node/.config/openclaw
```

OpenClaw utilizará:

```text
/home/node/.openclaw
├── openclaw.json
├── workspace/
├── sessions/
├── memory/
├── skills/
└── ...
```

y:

```text
/home/node/.config/openclaw
```

para secretos/perfiles de autenticación persistentes.

La instalación nueva debe ejecutarse como el usuario previsto por la imagen oficial, no forzar `root`.

---

## 3. Template MOS

Imagen:

```text
ghcr.io/openclaw/openclaw:latest
```

Puerto principal:

```text
18789/tcp
```

Comando del gateway:

```text
node openclaw.mjs gateway --bind lan --port 18789
```

Parámetros Docker recomendados:

```text
--restart=unless-stopped
--cap-drop=NET_RAW
--cap-drop=NET_ADMIN
--security-opt=no-new-privileges:true
--add-host=host.docker.internal:host-gateway
```

Los `cap-drop`, `no-new-privileges` y `host.docker.internal` siguen el enfoque del `docker-compose.yml` oficial actual.

---

## 4. Persistencia

### State

Host:

```text
/mnt/.../appdata/openclaw/state
```

Container:

```text
/home/node/.openclaw
```

Modo:

```text
rw
```

Aquí debe quedar el estado principal de OpenClaw.

### Auth profile secrets

Host:

```text
/mnt/.../appdata/openclaw/auth
```

Container:

```text
/home/node/.config/openclaw
```

Modo:

```text
rw
```

No versionar nunca el contenido de estas carpetas.

---

## 5. Gateway y Control UI

El Gateway de OpenClaw utiliza por defecto el puerto:

```text
18789
```

Ese mismo puerto sirve para Control UI, WebSocket/RPC, API HTTP, hooks y rutas HTTP de plugins.

Para un despliegue Docker accesible desde la LAN:

```text
gateway.mode = local
gateway.bind = lan
```

### Seguridad

No usar una configuración nueva sin autenticación.

Usar:

```text
OPENCLAW_GATEWAY_TOKEN
```

o un mecanismo de autenticación equivalente soportado por OpenClaw.

El token debe ser:

```text
vacío en el template público
mask: true
```

Nunca debe guardarse un token real en Git.

---

## 6. No usar `allowInsecureAuth=true`

La instalación antigua generaba una configuración equivalente a:

```text
controlUi.allowInsecureAuth = true
```

No queremos conservar esto como configuración predeterminada.

En MOS:

```text
Gateway auth      → habilitada
LAN bind          → solo si es necesario
Firewall          → limitar acceso
Internet directo  → NO
```

No publicar `18789` directamente hacia Internet.

---

## 7. Allowed Origins

Si el Control UI se utiliza desde otro equipo de la LAN o detrás de un reverse proxy, revisar la configuración de orígenes permitidos.

Ejemplo conceptual:

```text
gateway.controlUi.allowedOrigins
```

Añadir únicamente los orígenes realmente utilizados.

Ejemplo LAN:

```text
http://IP-DE-MOS:18789
```

Ejemplo reverse proxy:

```text
https://openclaw.example.com
```

No usar comodines innecesarios.

---

## 8. Migración desde el layout antiguo

La migración **no** debe consistir en copiar ciegamente `/root/.openclaw` sobre el nuevo contenedor y arrancar.

Primero debemos revisar y reorganizar el contenido.

### Datos actuales

```text
UNRAID:
appdata/openclaw/config    → /root/.openclaw
appdata/openclaw/workspace → /home/node/clawd
appdata/openclaw/projects  → /projects
appdata/openclaw/homebrew  → /home/linuxbrew/.linuxbrew
obsidian/Desktop/openClaw  → /data/obsidian
```

---

## 9. Estrategia de migración

### Fase 1 — Backup

Parar OpenClaw en Unraid:

```bash
docker stop OpenClaw
```

Guardar copia completa de:

```text
appdata/openclaw/config
appdata/openclaw/workspace
appdata/openclaw/projects
appdata/openclaw/homebrew
```

y, si se desea preservar la integración:

```text
appdata/obsidian/Desktop/openClaw
```

No borrar nada del origen.

### Fase 2 — Inspección

Antes de copiar, revisar:

```bash
find /ruta/openclaw/config -maxdepth 3 -type f | sort
find /ruta/openclaw/workspace -maxdepth 3 -type f | sort
```

Especial atención a:

```text
openclaw.json
sessions
memory
skills
workspace
credentials
auth profiles
```

La estructura exacta puede cambiar entre versiones.

### Fase 3 — Crear estructura MOS

Crear:

```text
/mnt/.../appdata/openclaw/state
/mnt/.../appdata/openclaw/auth
```

### Fase 4 — Copiar estado

Copiar primero el contenido persistente necesario del antiguo `/root/.openclaw` hacia `/home/node/.openclaw`.

En el host:

```text
antiguo config/
        ↓
nuevo state/
```

El workspace antiguo debe revisarse antes de decidir si se integra dentro de `state/workspace/`.

No asumir que todo el layout antiguo coincide directamente con el nuevo.

---

## 10. Ownership

La instalación antigua fuerza `root`.

La nueva no debe depender de eso.

Después de preparar el appdata MOS:

```bash
chown -R 1000:1000 /mnt/.../appdata/openclaw/state
chown -R 1000:1000 /mnt/.../appdata/openclaw/auth
```

Si se añaden mounts opcionales que deban ser editables por OpenClaw, revisar también sus permisos.

No solucionar problemas de permisos volviendo a ejecutar todo el contenedor como root salvo que exista una razón muy concreta y documentada.

---

## 11. Obsidian

En Tower existe actualmente una integración con Obsidian:

```text
/mnt/ssd_system/appdata/obsidian/Desktop/openClaw
    → /data/obsidian
```

Esto es específico de nuestra instalación.

No debe aparecer en el template público.

En MOS se puede volver a añadir manualmente:

```text
HOST:
ruta-real-del-vault

CONTAINER:
/data/obsidian
```

La ruta debe ser compartida con Obsidian solo si ambos servicios necesitan trabajar sobre el mismo contenido.

Revisar permisos antes de activar escritura desde ambos contenedores.

---

## 12. Projects

El mount `/projects` también es específico de nuestra instalación.

No incluirlo en el template público.

Si se necesita:

```text
/mnt/.../openclaw/projects
    → /projects
```

y revisar ownership.

---

## 13. Homebrew

La instalación antigua tiene un Homebrew persistente en:

```text
/home/linuxbrew/.linuxbrew
```

No migrarlo automáticamente.

Durante la migración:

```text
1. guardar backup
2. arrancar OpenClaw sin Homebrew
3. comprobar qué skills realmente necesitan herramientas externas
4. reinstalar solo lo necesario
```

No incluir Homebrew en el template público.

---

## 14. API keys

El template MOS puede ofrecer inputs opcionales para proveedores, por ejemplo:

```text
OPENAI_API_KEY
ANTHROPIC_API_KEY
OPENROUTER_API_KEY
```

Todos deben ser:

```json
"value": "",
"mask": true
```

No incluir secretos actuales ni tokens exportados desde Unraid.

---

## 15. Gateway Token

Crear un token nuevo para la instalación MOS.

Ejemplo:

```bash
openssl rand -hex 24
```

Configurar el resultado únicamente en MOS.

No reutilizar tokens que hayan aparecido en capturas, terminales compartidas, documentación, commits o chats.

---

## 16. Primer arranque

Antes del primer arranque definitivo:

```text
[ ] state preparado
[ ] auth preparado
[ ] ownership correcto
[ ] token nuevo
[ ] gateway.mode revisado
[ ] gateway.bind revisado
[ ] allowedOrigins revisado
[ ] puerto 18789 no expuesto a Internet
```

Arrancar el contenedor.

---

## 17. Validación

Comprobar:

```text
[ ] Container running
[ ] Control UI accesible
[ ] Gateway autenticado
[ ] Configuración cargada
[ ] Workspace visible
[ ] Memory disponible
[ ] Sessions disponibles si se conservaron
[ ] Skills disponibles
[ ] Provider LLM funciona
[ ] Obsidian funciona si se añadió
[ ] Projects funciona si se añadió
[ ] No necesita root
[ ] Reinicio del contenedor correcto
[ ] Reinicio completo de MOS correcto
```

---

## 18. Comandos útiles

Logs:

```bash
docker logs -f OpenClaw
```

Shell:

```bash
docker exec -it OpenClaw sh
```

Estado del gateway, si el CLI de la imagen lo permite en esa versión:

```bash
docker exec -it OpenClaw openclaw gateway status
```

Diagnóstico:

```bash
docker exec -it OpenClaw openclaw doctor
```

Si una versión cambia los comandos disponibles, consultar la documentación de esa versión antes de modificar el template.

---

## 19. Rollback

No eliminar la instalación de Unraid inmediatamente.

Rollback:

```text
1. detener OpenClaw MOS
2. mantener intacto appdata antiguo
3. arrancar OpenClaw en Unraid
4. comprobar acceso
```

El origen se conserva hasta validar varios reinicios y el funcionamiento real de las integraciones.

---

## 20. Qué NO debe hacer el template público

El template `OpenClaw.json` no debe:

- montar el vault personal de Obsidian;
- montar `/projects`;
- persistir Homebrew por defecto;
- incluir API keys;
- incluir gateway tokens;
- activar `allowInsecureAuth`;
- forzar `--user root`;
- incluir rutas específicas de Tower;
- copiar configuraciones antiguas de Unraid.

El template debe proporcionar una instalación limpia.

Este documento explica cómo migrar una instalación existente.

---

## 21. Resumen del cambio

```text
UNRAID ACTUAL
────────────────────────────────────
image 2026.4.24
root
/root/.openclaw
/home/node/clawd
/projects
/home/linuxbrew/.linuxbrew
/data/obsidian
allowInsecureAuth=true

                 ↓ migración

MOS
────────────────────────────────────
imagen oficial actual
usuario de la imagen
/home/node/.openclaw
/home/node/.config/openclaw
token nuevo
auth habilitada
sin insecure auth
mounts personales opcionales
```

La prioridad de la migración es:

```text
CONSERVAR LOS DATOS
+
MODERNIZAR EL CONTENEDOR
+
NO ARRASTRAR INSEGURIDADES ANTIGUAS
```

---

## Referencias upstream

- OpenClaw Docker: https://docs.openclaw.ai/install/docker
- Repositorio: https://github.com/openclaw/openclaw
- Docker Compose oficial: https://github.com/openclaw/openclaw/blob/main/docker-compose.yml
- Gateway: https://github.com/openclaw/openclaw/blob/main/docs/gateway/index.md
- Seguridad del Gateway: https://github.com/openclaw/openclaw/blob/main/docs/gateway/security/index.md
