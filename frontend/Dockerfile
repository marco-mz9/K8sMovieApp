# ============================================================
# Dockerfile - Frontend (React + Vite)
# Proyecto: Movie Review App - FinTech Solutions S.A.
# Capa: Presentación / Interfaz de usuario
# Puerto expuesto: 80 (Nginx sirve el build de producción)
# ============================================================

# ── Etapa 1: BUILD ──────────────────────────────────────────
# Usamos Node.js para compilar la aplicación React con Vite
FROM node:20-alpine AS builder

LABEL maintainer="TechOps Solutions"
LABEL description="Frontend React/Vite - Movie Review App (etapa de build)"

WORKDIR /app

# Copiar dependencias primero (optimización de caché)
COPY package.json package-lock.json ./

# Instalar todas las dependencias (incluyendo devDependencies para el build)
RUN npm ci

# Copiar el código fuente completo
COPY . .

# Compilar la aplicación para producción (genera la carpeta /app/dist)
RUN npm run build

# ── Etapa 2: SERVE ──────────────────────────────────────────
# Imagen final ligera: Nginx sirve los archivos estáticos del build
FROM nginx:stable-alpine

LABEL maintainer="TechOps Solutions"
LABEL description="Frontend React/Vite - Movie Review App (servidor Nginx)"
LABEL version="1.0.0"

# Eliminar la configuración por defecto de Nginx
RUN rm -rf /usr/share/nginx/html/*

# Copiar los archivos compilados desde la etapa builder
COPY --from=builder /app/dist /usr/share/nginx/html

# Copiar configuración personalizada de Nginx (SPA routing)
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Exponer el puerto HTTP estándar
EXPOSE 80

# Nginx arranca en primer plano (requerido por Docker)
CMD ["nginx", "-g", "daemon off;"]
