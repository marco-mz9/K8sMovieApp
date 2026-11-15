# Despliegue de Aplicación Segura para Gestión Financiera en Kubernetes (EKS)

**Cliente:** FinTech Solutions S. A.  
**Proveedor:** TechOps Solutions  
**Arquitectura:** 3 Capas en Clúster de 2 Nodos (AWS EKS)

---

## Arquitectura y Manifiestos Declarativos

| Archivo | Recurso(s) | Propósito |
| :--- | :--- | :--- |
| `00-namespace.yaml` | `Namespace` (`fintech`) | Aislamiento lógico de todos los recursos de la aplicación. |
| `01-secrets-config.yaml` | `Secret`, `ConfigMap` | Gestión segura de credenciales (`mongodb-secret`) y variables de entorno (`fintech-config`). |
| `02-storage.yaml` | `PersistentVolume`, `PersistentVolumeClaim` | Almacenamiento persistente estático (`mongo-pv` y `mongo-pvc`, 2Gi) para los datos de MongoDB. |
| `03-mongodb.yaml` | `StatefulSet`, `Service (ClusterIP)` | Base de datos MongoDB 7.0 con probes, límites de recursos y persistencia en `/data/db`. |
| `04-backend.yaml` | `Deployment` (2 réplicas), `Service (ClusterIP)` | API REST (`markm9/fintech-backend:1.0`) expuesta internamente en `backend:5000`. |
| `05-frontend.yaml` | `Deployment` (2 réplicas), `Service (LoadBalancer)` | Interfaz React/Nginx (`markm9/fintech-frontend:1.0`) publicada con balanceador AWS ELB. |
| `06-security.yaml` | `NetworkPolicy`, `ServiceAccount`, `Role`, `RoleBinding` | Aislamiento de red entre capas (Frontend -> Backend -> MongoDB) y control de acceso RBAC. |

---

## Paso 1: Subir este proyecto a GitHub (Desde tu PC)

Abre una terminal dentro de esta carpeta (`K8sMovieApp`) y ejecuta:

```bash
git init
git add .
git commit -m "feat: manifiestos declarativos Kubernetes para FinTech Solutions"
git branch -M main
git remote add origin https://github.com/<TU-USUARIO>/<TU-REPOSITORIO>.git
git push -u origin main
```

---

## Paso 2: Clonar y desplegar en AWS EC2 (Clúster EKS de 2 Nodos)

Conéctate por SSH a tu instancia bastion de AWS y ejecuta los siguientes comandos:

```bash
# 1. Verificar que los 2 worker nodes estén Ready
kubectl get nodes -o wide

# 2. Clonar tu repositorio desde GitHub
git clone https://github.com/<TU-USUARIO>/<TU-REPOSITORIO>.git
cd <TU-REPOSITORIO>

# 3. Aplicar primero el Namespace y luego todos los manifiestos
kubectl apply -f 00-namespace.yaml
kubectl apply -f .
```

---

## Paso 3: Verificar el despliegue y capturar evidencias para el PDF

```bash
# Ver todos los recursos, pods, servicios y almacenamiento en el namespace fintech
kubectl get all,pvc -n fintech -o wide

# Ver las políticas de red y RBAC aplicadas
kubectl get networkpolicy,sa,role,rolebinding -n fintech

# Obtener la URL pública del LoadBalancer de AWS para abrir en el navegador
kubectl get svc frontend -n fintech
```

### Prueba de resiliencia y persistencia (Recomendada para el informe)
1. Abre la URL del servicio `frontend` (`EXTERNAL-IP`) en tu navegador y crea un registro de prueba.
2. Elimina el pod de MongoDB para simular una caída:
   ```bash
   kubectl delete pod mongodb-0 -n fintech
   ```
3. Espera unos segundos a que el `StatefulSet` lo recree (`kubectl get pods -n fintech`) y recarga el navegador: los datos seguirán intactos gracias al `PersistentVolumeClaim`.

---

## Solución de Problemas: Compatibilidad de Arquitectura (`arm64` vs `amd64`) y Almacenamiento

Si tus imágenes en Docker Hub (`markm9/fintech-backend:1.0` y `markm9/fintech-frontend:1.0`) fueron compiladas originalmente en Mac Apple Silicon (`linux/arm64`), los nodos worker de AWS EKS (`linux/amd64` / `x86_64`) mostrarán `ErrImagePull`. Además, si el clúster EKS de laboratorio no tiene un provisionador CSI dinámico instalado, el `mongo-pvc` quedará en `Pending`.

### 1. Reconstruir y subir las imágenes en arquitectura `linux/amd64` desde tu instancia EC2

Una vez actualices este repositorio en tu EC2 (`git pull`), reconstruye y publica ambas imágenes nativas para `linux/amd64`:

```bash
cd ~/K8sMovieApp

# Iniciar sesión en Docker Hub
docker login -u markm9

# Construir y subir Backend (linux/amd64)
docker build --platform linux/amd64 -t markm9/fintech-backend:1.0 ./backend
docker push markm9/fintech-backend:1.0

# Construir y subir Frontend (linux/amd64)
docker build --platform linux/amd64 -t markm9/fintech-frontend:1.0 ./frontend
docker push markm9/fintech-frontend:1.0
```

*(Alternativa desde tu Mac con Apple Silicon: usa `docker buildx build --platform linux/amd64,linux/arm64 -t markm9/fintech-backend:1.0 --push ./backend` para generar imágenes multi-arquitectura).*

### 2. Reaplicar almacenamiento y reiniciar los Deployments

Como el `mongo-pvc` anterior no tenía `storageClassName: manual`, elimínalo para recrearlo junto al `PersistentVolume` `mongo-pv` y reinicia los pods:

```bash
# Reemplazar el PVC anterior y crear el PV + PVC con storageClassName: manual
kubectl delete pvc mongo-pvc -n fintech
kubectl apply -f 02-storage.yaml

# Reiniciar los despliegues para que vuelvan a descargar las imágenes amd64
kubectl rollout restart deployment backend frontend -n fintech

# Verificar que PV/PVC estén Bound y todos los pods en Running
kubectl get pv,pvc,pods -n fintech -o wide
```
