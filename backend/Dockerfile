# ============================================================
# Dockerfile - Backend (Node.js / Express)
# Proyecto: Movie Review App - FinTech Solutions S.A.
# Capa: Lógica / API REST
# Puerto expuesto: 5000
# ============================================================

# Imagen base oficial de Node.js (LTS Alpine para menor tamaño)
FROM node:20-alpine

# Metadatos de la imagen
LABEL maintainer="TechOps Solutions"
LABEL description="Backend API REST - Movie Review App (Express + Mongoose)"
LABEL version="1.0.0"

# Establecer el directorio de trabajo dentro del contenedor
WORKDIR /app

# Copiar primero los archivos de dependencias (aprovecha caché de capas Docker)
COPY package.json package-lock.json ./

# Instalar dependencias de producción únicamente
RUN npm ci --omit=dev

# Copiar el resto del código fuente
COPY . .

# Exponer el puerto que usa Express
EXPOSE 5000

# Variable de entorno por defecto (puede sobreescribirse con --env o docker-compose)
ENV NODE_ENV=production
ENV PORT=5000

# Comando de arranque de la aplicación
CMD ["node", "index.js"]
