# Informe de Práctica: Contenerización de Aplicación React y Mock API

## 1\. Introducción y Objetivo

El objetivo de esta actividad fue contenerizar una aplicación **React** (frontend) y un servicio de **Backend Simulado (Mock API)**, utilizando un enfoque de desarrollo moderno. La práctica se centró en la creación de un `Dockerfile` optimizado para la aplicación React y el uso de Docker Compose para la orquestación y la interconexión de ambos servicios.

-----

## 2\. Dockerfile para la Aplicación React (Frontend)

Se creó un `Dockerfile` para la aplicación `suda-frontend-s6` utilizando una estrategia de construcción en **Múltiples Etapas (Multi-stage build)**. Esto permite separar el entorno de construcción (grande) del entorno de ejecución (pequeño y seguro), resultando en una imagen final optimizada y reducida.

### 2.1. Archivo `Dockerfile`

```dockerfile
# ETAPA 1: Construcción (Build Stage)
FROM node:20-alpine AS builder

# Establecer el directorio de trabajo
WORKDIR /app

# Copiar archivos de definición para optimizar el caché de npm
COPY package.json package-lock.json ./

# Instalar dependencias
RUN npm install

# Copiar el código fuente completo
COPY . .

# Generar la versión de producción de la aplicación React
RUN npm run build

# ETAPA 2: Producción (Production Stage)
# Utilizamos una imagen ligera de NGINX para servir los archivos estáticos
FROM nginx:alpine

# Copiar el resultado de la construcción (archivos estáticos) de la etapa 'builder'
COPY --from=builder /app/build /usr/share/nginx/html

# Puerto de exposición (NGINX usa el 80 por defecto)
EXPOSE 80

# Comando para iniciar NGINX
CMD ["nginx", "-g", "daemon off;"]
```

-----

## 3\. Orquestación con Docker Compose

Se utilizó un archivo `docker-compose.yml` para levantar tanto la Mock API (backend) como la imagen del Frontend construida localmente, conectándolos a una red común.

### 3.1. Archivo `docker-compose.yml`

```yaml
version: '3.8'

services:
  # 1. SERVICIO DE BACKEND (Mock API)
  mockapi:
    image: node:20-alpine
    container_name: mock-api
    restart: always
    working_dir: /usr/src/app
    volumes:
      - ./mockAPI:/usr/src/app # Mapeo del código de la Mock API
    networks:
      - app-network
    command: npm start
    expose:
      - 3001 # Puerto interno de la API

  # 2. SERVICIO DE FRONTEND (Aplicación React)
  frontend:
    depends_on:
      - mockapi # Asegurar que la API esté intentando arrancar
    image: suda-frontend-s6-image # Nombre de la imagen generada localmente
    container_name: suda-frontend
    restart: always
    ports:
      - "3000:80" # Mapea el puerto 80 de NGINX al 3000 del host
    networks:
      - app-network
    environment:
      # URL que React debe usar para conectarse a la API
      # Usamos el nombre del servicio 'mockapi' como hostname
      REACT_APP_API_BASE_URL: http://mockapi:3001

# DEFINICIÓN DE RED
networks:
  app-network:
    driver: bridge
```

### 3.2. Proceso de Generación y Ejecución

| **Acción** | **Comando Clave** | **Propósito** |
| :--- | :--- | :--- |
| **Generar Imagen (Frontend)** | `docker build -t suda-frontend-s6-image .` | Construye la imagen utilizando el `Dockerfile` de múltiples etapas creado. |
| **Ejecutar Entorno** | `docker-compose up -d` | Levanta ambos servicios (`mockapi` y `frontend`), gestionando la red y las dependencias. |
| **Acceso a la App** | `http://localhost:3000` | Acceso a la interfaz de usuario servida por el contenedor NGINX del Frontend. |

-----

## 4\. Conclusión

El uso de **Dockerfiles de Múltiples Etapas** permitió crear una imagen de producción de React **optimizada y ligera**, minimizando la superficie de ataque. Al integrar ambos servicios mediante **Docker Compose** y una **Red Personalizada**, se logró que la aplicación Frontend se conectara de forma exitosa y segura al Backend Simulado (Mock API) utilizando el nombre de servicio `mockapi` como hostname, demostrando una arquitectura contenerizada completa.
