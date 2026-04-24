# 🏢 Infraestructura Web con Proxy Inverso Nginx

Despliegue de una infraestructura web empresarial usando **Docker Compose**, con proxy inverso Nginx, balanceo de carga entre dos backends y red interna aislada.

---

## 📐 Diagrama de Arquitectura

```
                        INTERNET / HOST
                              │
                         Puerto 80
                              │
                    ┌─────────▼──────────┐
                    │    PROXY (Nginx)    │  ← Red: frontend + backend
                    │   [proxy:80]        │
                    └────┬──────────┬────┘
                         │          │
              Round-Robin balanceo de carga
                         │          │
           ┌─────────────▼─┐   ┌───▼─────────────┐
           │  backend1:80  │   │  backend2:80     │
           │  (Nginx)      │   │  (Nginx)         │
           └───────┬───────┘   └──────┬───────────┘
                   │                  │
                   └────────┬─────────┘
                            │
                    ┌───────▼────────┐
                    │ Volumen Docker │   ← ./web/ compartido
                    │   "webdata"    │      (index.html, imagen.jpg,
                    └────────────────┘       video.mp4)

Red "frontend":  proxy ↔ host         (bridge, con puerto expuesto)
Red "backend":   proxy ↔ backend1/2   (bridge, internal: true → aislada)
```

---

## 📁 Estructura del Repositorio

```
.
├── docker-compose.yml        # Definición completa de la infraestructura
├── nginx/
│   └── nginx.conf            # Configuración del proxy inverso
├── web/
│   ├── index.html            # Página web principal
│   ├── imagen.jpg            # Imagen estática (añadir manualmente)
│   └── video.mp4             # Vídeo de demostración (añadir manualmente)
└── README.md                 # Este fichero
```

> **Nota:** `imagen.jpg` y `video.mp4` deben añadirse manualmente a la carpeta `web/` antes de levantar el entorno.

---

## ⚙️ Decisiones de Diseño

### ¿Por qué Nginx para todos los contenedores?

Se utiliza la imagen oficial `nginx:alpine` tanto para el proxy como para los backends por las siguientes razones:

- **Unificación**: usar la misma imagen simplifica el mantenimiento y reduce la variedad de dependencias.
- **Ligereza**: la variante `alpine` minimiza el tamaño del contenedor (~25MB).
- **Fiabilidad**: imagen oficial de Docker Hub, mantenida y auditada por la comunidad.
- **Versatilidad**: Nginx funciona perfectamente tanto como servidor web estático como proxy inverso, solo cambiando la configuración.

### ¿Por qué dos redes separadas?

Se definen dos redes Docker con roles distintos:

| Red | Tipo | Propósito |
|---|---|---|
| `frontend` | bridge (normal) | Conecta el proxy con el host. Permite exponer el puerto 80. |
| `backend` | bridge + `internal: true` | Conecta proxy con backends. **Sin acceso desde el exterior.** |

Esta separación garantiza que los backends **no son accesibles directamente desde el host**, solo a través del proxy. Esto mejora la seguridad y refleja una arquitectura real de producción.

### ¿Por qué un volumen bind mount?

El volumen `webdata` usa `driver_opts` con `type: none` y `o: bind`, lo que equivale a un bind mount gestionado por Docker Compose. Esto permite:

- **Compartir el mismo contenido** entre `backend1` y `backend2` sin duplicar ficheros.
- **Editar el contenido web desde el host** sin reconstruir imágenes.
- El proxy sirve siempre el mismo HTML independientemente del backend que responda.

### Balanceo de carga

El proxy usa el bloque `upstream` de Nginx con los dos backends. Por defecto, Nginx aplica **round-robin** (peticiones alternadas): backend1, backend2, backend1...

Para identificar qué backend ha respondido cada petición, se añade la cabecera HTTP personalizada:
```
X-Backend-Server: <IP_del_backend>
```

---

## 🚀 Cómo Levantar el Entorno

### Requisitos previos

- Docker >= 20.x
- Docker Compose >= 2.x

### Pasos

```bash
# 1. Clonar el repositorio
git clone https://github.com/TU_USUARIO/TU_REPO.git
cd TU_REPO

# 2. Añadir los ficheros multimedia (si no están ya)
#    Colocar imagen.jpg y video.mp4 dentro de web/

# 3. Levantar toda la infraestructura
docker compose up -d

# 4. Acceder a la web
# Abrir en el navegador: http://localhost
```

---

## ✅ Verificación del Entorno

### 1. Ver que los contenedores están en marcha

```bash
docker compose ps
```

Deberías ver `proxy`, `backend1` y `backend2` en estado `running`.

---

### 2. Verificar el balanceo de carga (round-robin)

Ejecuta varias veces el siguiente comando y observa cómo cambia la cabecera `X-Backend-Server`:

```bash
curl -s -I http://localhost | grep X-Backend-Server
```

Ejemplo de salida alternada:
```
X-Backend-Server: 172.18.0.3:80   ← backend1
X-Backend-Server: 172.18.0.4:80   ← backend2
X-Backend-Server: 172.18.0.3:80   ← backend1 (de nuevo)
```

O con un bucle para verlo claramente:
```bash
for i in $(seq 1 6); do curl -s -I http://localhost | grep X-Backend-Server; done
```

---

### 3. Verificar que los backends NO son accesibles desde el host

Los backends no exponen ningún puerto al host. Para comprobarlo:

```bash
# Intentar acceder directamente a backend1 (debe fallar)
curl http://localhost:8081   # No hay ningún puerto mapeado → Connection refused

# Verificar que backend1 no tiene puertos publicados
docker inspect backend1 | grep -A 10 "Ports"
# El resultado mostrará "Ports": {} → sin puertos expuestos
```

También puedes verificar las redes:
```bash
# Ver que backend1 y backend2 solo están en la red "backend" (internal)
docker network inspect proyecto_backend
# Los backends aparecen, el host NO

docker network inspect proyecto_frontend
# Solo aparece el proxy
```

---

### 4. Verificar el volumen compartido

```bash
# Ver que ambos backends montan el mismo volumen
docker inspect backend1 | grep -A 5 "Mounts"
docker inspect backend2 | grep -A 5 "Mounts"
# Ambos apuntan al mismo directorio ./web/
```

---

### 5. Ver los logs del proxy en tiempo real

```bash
docker compose logs -f proxy
```

Cada petición muestra qué backend la ha procesado.

---

## 🛑 Parar el Entorno

```bash
docker compose down
```

---

## 📸 Capturas de Pantalla

*(Añadir aquí capturas de: navegador en http://localhost, salida de `curl -I`, salida de `docker compose ps`, y resultado de la verificación de red aislada)*
