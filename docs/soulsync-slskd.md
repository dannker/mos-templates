# SoulSync + slskd en MOS

Guía de referencia para preparar la futura migración de **Tower (Unraid) → MOS** y recordar cómo deben relacionarse `slskd`, `SoulSync` y `Navidrome`.

> Esta guía no contiene contraseñas, API keys ni rutas personales definitivas. Los secretos deben configurarse únicamente en la instalación local.

## 1. Arquitectura prevista

```text
Soulseek
   │
   ▼
 slskd
   │
   │ descarga
   ▼
incoming / downloads
   │
   ▼
SoulSync
   │
   │ organiza / mueve
   ▼
Music Library
   │
   ▼
Navidrome
```

La idea importante es que **slskd y SoulSync deben ver la misma carpeta física de descargas del host**, aunque dentro de cada contenedor tenga una ruta distinta.

Ejemplo:

```text
HOST
/mnt/.../music/incoming
        │
        ├── slskd    → /downloads
        └── SoulSync → /app/downloads
```

Y la biblioteca final:

```text
HOST
/mnt/.../music/albums
        │
        ├── SoulSync  → /app/Transfer
        └── Navidrome → /music
```

## 2. slskd

### Imagen

```text
slskd/slskd:latest
```

### Puertos

```text
5030/tcp   Web UI HTTP
5031/tcp   Web UI HTTPS
50300/tcp  Soulseek
```

### Persistencia

```text
/app        Configuración y datos de slskd
/downloads  Descargas
/music      Biblioteca compartida con Soulseek
```

Recomendación:

```text
/app        rw
/downloads  rw
/music      ro
```

### Variables principales

```text
SLSKD_REMOTE_CONFIGURATION=true
SLSKD_DOWNLOADS_DIR=/downloads
SLSKD_SHARED_DIR=[Music]/music
```

Las credenciales de Soulseek, usuario/password de la Web UI y la API key se configuran localmente.

## 3. SoulSync

### Imagen

```text
boulderbadgedad/soulsync:latest
```

### Puertos

```text
8008/tcp  Web UI
8888/tcp  Spotify OAuth
8889/tcp  Tidal OAuth
```

### Persistencia recomendada

```text
/app/config       Configuración
/app/data         Bases de datos
/app/logs         Logs
/app/downloads    Descargas compartidas con slskd
/app/Transfer     Biblioteca musical organizada
/app/MusicVideos  Vídeos musicales opcionales
```

Ejemplo de rutas MOS:

```text
/mnt/cache/appdata/soulsync/config  → /app/config
/mnt/cache/appdata/soulsync/data    → /app/data
/mnt/cache/appdata/soulsync/logs    → /app/logs

/mnt/.../music/incoming             → /app/downloads
/mnt/.../music/albums               → /app/Transfer
```

## 4. Comunicación SoulSync → slskd

SoulSync necesita poder acceder a la API de slskd.

Hay dos formas razonables.

### Opción A — red Docker compartida

Crear una red bridge de usuario, por ejemplo:

```text
music-stack
```

y conectar ambos contenedores a ella.

Se puede dar a slskd un alias estable:

```text
slskd
```

SoulSync podría entonces usar:

```text
http://slskd:5030
```

Esta es la opción preferida si ambos contenedores viven permanentemente en el mismo host MOS.

### Opción B — puerto publicado del host

Si SoulSync dispone de:

```text
--add-host=host.docker.internal:host-gateway
```

puede acceder al puerto publicado de slskd mediante:

```text
http://host.docker.internal:5030
```

## 5. API Key

En slskd se genera/configura una API key.

SoulSync necesita esa API key para comunicarse con slskd.

Nunca debe incluirse una API key real en:

```text
dannker/mos-templates
```

La API key se configura únicamente en la instalación local de Tower.

## 6. Navidrome

Navidrome utilizará la biblioteca final organizada por SoulSync.

Ejemplo:

```text
SoulSync
/mnt/.../music/albums → /app/Transfer:rw

Navidrome
/mnt/.../music/albums → /music:ro
```

Así SoulSync puede escribir y organizar la biblioteca mientras que Navidrome únicamente necesita leerla.

# Migración futura desde Unraid

## 7. slskd actual

Actualmente Tower usa aproximadamente:

```text
/mnt/ssd_system/appdata/slskd       → /app
/mnt/user/data/media/music/incoming → /app/downloads
/mnt/user/data/media/music/albums   → /app/uploads
```

Antes de migrar:

1. detener `slskd`;
2. copiar `/app` completo;
3. conservar configuración y base de datos;
4. adaptar los mounts al nuevo esquema MOS;
5. comprobar permisos;
6. iniciar slskd;
7. verificar Web UI;
8. comprobar login Soulseek;
9. comprobar puerto 50300;
10. probar una descarga.

No borrar el appdata antiguo hasta terminar las pruebas.

## 8. SoulSync actual

La instalación actual de Unraid tiene una mezcla de bind mounts y volúmenes Docker anónimos.

Bindings importantes:

```text
.../soulsync/config.json → /app/config/config.json
.../soulsync/logs        → /app/logs
.../soulsync/database    → /app/data
music/albums             → /host/music
music/playlists          → /host/playlist
music/incoming           → /downloads
```

Además existen volúmenes Docker anónimos para:

```text
/app/config
/app/downloads
/app/Transfer
/app/MusicVideos
/app/scripts
```

### IMPORTANTE

Antes de eliminar el contenedor viejo hay que revisar esos volúmenes.

Ejemplo:

```bash
docker inspect SoulSync
```

y después inspeccionar cada volumen relevante:

```bash
docker volume inspect <VOLUMEN>
```

No debemos asumir que están vacíos.

Especial atención a:

```text
/app/config
/app/Transfer
/app/MusicVideos
```

## 9. Migración de config.json al nuevo esquema

La instalación antigua monta directamente:

```text
config.json → /app/config/config.json
```

La futura plantilla MOS montará el directorio completo:

```text
.../soulsync/config → /app/config
```

Por tanto el resultado en MOS debe quedar:

```text
/mnt/.../soulsync/config/
└── config.json
```

No montar de nuevo únicamente el fichero si usamos la plantilla nueva.

## 10. Orden recomendado de migración

```text
1. Crear los directorios finales de música
2. Migrar slskd
3. Validar slskd
4. Crear/configurar red Docker music-stack
5. Migrar SoulSync
6. Configurar SoulSync → slskd
7. Validar una descarga completa
8. Validar organización/movimiento de archivos
9. Migrar/configurar Navidrome
10. Validar biblioteca en Navidrome
11. Mantener la copia Unraid hasta comprobar todo
```

# Checklist de validación

## slskd

- [ ] Web UI accesible
- [ ] Login Soulseek correcto
- [ ] API operativa
- [ ] Puerto 50300 accesible
- [ ] Descargas llegan a la carpeta compartida
- [ ] Biblioteca compartida visible
- [ ] Permisos correctos

## SoulSync

- [ ] Web UI accesible
- [ ] Configuración conservada
- [ ] Base de datos conservada
- [ ] Comunicación con slskd
- [ ] API key correcta
- [ ] Ve las descargas de slskd
- [ ] Organiza/mueve archivos
- [ ] Biblioteca final correcta
- [ ] OAuth Spotify/Tidal si se usan

## Navidrome

- [ ] Web UI accesible
- [ ] Biblioteca visible
- [ ] Escaneo correcto
- [ ] Usuarios/configuración conservados
- [ ] Música reproducible

# Filosofía de los templates públicos

En `dannker/mos-templates`:

- usar documentación **upstream** como referencia;
- mantener templates sencillos;
- no copiar rutas personales de Tower;
- no publicar contraseñas;
- no publicar API keys;
- no incluir configuraciones específicas de una instalación;
- los detalles de migración y casos especiales pertenecen a `docs/`.

El template debe servir para instalar la aplicación.

La documentación debe explicar cómo migrarla y conectarla con el resto del stack.
