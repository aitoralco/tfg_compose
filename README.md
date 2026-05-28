# Entorno de Infraestructura Local (TFG)

Este repositorio contiene la configuración de la infraestructura de datos y almacenamiento necesaria para el backend del Trabajo de Fin de Grado. El entorno está completamente contenedorizado, lo que garantiza la portabilidad y la paridad entre los entornos de desarrollo y producción.

---

## 🏗️ Arquitectura de Servicios

El archivo `compose.yaml` define tres servicios core independientes y persistentes:

| Servicio | Tecnología | Puerto Local | Descripción / Uso en el Proyecto |
| :--- | :--- | :--- | :--- |
| **`db-postgres`** | PostgreSQL 15 (Alpine) | `5432` | Base de datos relacional principal para persistencia de datos de usuario, entidades y lógica de negocio. |
| **`cache-redis`** | Redis (Latest) | `6379` | Almacenamiento en caché en memoria (`Key-Value`) para optimizar el rendimiento de las consultas y la gestión de sesiones. Requiere autenticación mediante contraseña y usa configuración ad-hoc. |
| **`filesystem-minio`** | MinIO (Latest) | `9000` / `9001` | Servidor de almacenamiento de objetos compatible con la API de Amazon S3. Utilizado para gestionar el sistema de archivos (imágenes, documentos, etc.). |

---

## 🚀 Requisitos e Instalación del Entorno

Para el despliegue de esta infraestructura se ha optado por un enfoque híbrido priorizando el uso de herramientas de código abierto y alineadas con los estándares empresariales actuales:

1. **Podman Engine:** Utilizado como motor de contenedores principal por su arquitectura *rootless* (sin demonios) impulsada por Red Hat, ofreciendo mayor seguridad y menor consumo de recursos frente a alternativas tradicionales.
2. **Docker Desktop (Compose Provider):** Instalado exclusivamente para delegar la interpretación del motor de orquestación de Docker Compose (`docker-compose.exe`), el cual es ejecutado internamente de forma transparente por Podman.

### 🏁 Instrucciones de Despliegue

1. **Clonar el repositorio:**
   ```bash
   git clone <url-del-repositorio>
   cd tfg_dockercompose
   ```
2. **Configurar las variables de entorno:**
Duplica el archivo de ejemplo y renómbralo a .env. Modifica las credenciales si lo consideras necesario (el archivo .env está excluido del control de versiones por seguridad):

    ```bash
    cp .env.example .env
    ```
3. **Levantar los servicios:**
Ejecuta el siguiente comando para compilar, configurar y arrancar los contenedores en segundo plano (detached mode):

    ```bash
    podman compose up -d
    ```
4. **Verificar el estado:**
Comprueba que los tres contenedores estén levantados de manera correcta:

    ```bash
    podman compose ps
    ```
5. **Detener el entorno:**
Para pausar la infraestructura sin perder los datos almacenados en los volúmenes locales:

    ```bash
    podman compose down
    ```
## 🔌 Conexión desde la API (Entorno de Desarrollo)
Cuando desarrolles tu API de forma local (ejecutándose directamente en tu máquina mediante Node.js, Python, .NET, etc.), debes apuntar a localhost (127.0.0.1) junto con las variables definidas en tu archivo .env.

### 🐘 1. PostgreSQL
Host: localhost o 127.0.0.1

Puerto: 5432

URL de conexión (URI): postgresql://${DB_USER}:${DB_PASSWORD}@localhost:5432/${DB_NAME}

### ⚡ 2. Redis
Host: localhost o 127.0.0.1

Puerto: 6379

Estrategia de conexión: Es obligatorio habilitar la autenticación de cliente pasando el parámetro de la contraseña (REDIS_PASSWORD).

URI estándar: redis://:${REDIS_PASSWORD}@localhost:6379

### 🪣 3. MinIO (Almacenamiento de Objetos S3)
MinIO expone dos interfaces con propósitos completamente distintos:

Acceso de la API (SDK de S3 / Backend):

Endpoint: http://localhost:9000 (este es el que debes configurar en tu cliente S3 de la API).

Access Key: Valor de ${MINIO_ROOT_USER}

Secret Key: Valor de ${MINIO_ROOT_PASSWORD}

Consola de Administración Web (UI):

URL: http://localhost:9001

Interfaz gráfica para la creación manual de buckets, gestión de políticas de acceso y visualización de archivos cargados.

### 💾 Persistencia de Datos
Para evitar la volatilidad de la información al destruir los contenedores, se han mapeado volúmenes dedicados administrados por el motor de Podman:

postgres_data -> Mapeado a /var/lib/postgresql/data

redis_data -> Mapeado a /data

minio_data -> Mapeado a /data

Adicionalmente, la configuración del servicio Redis se inyecta directamente de manera local a través del archivo ./redis/redis.conf.