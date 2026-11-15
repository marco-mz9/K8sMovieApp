# Despliegue de Aplicación en Kubernetes (EKS)

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

## Paso 1: Clonar y desplegar en AWS EC2 (Clúster EKS de 2 Nodos)

Conéctate por SSH a tu instancia bastion de AWS y ejecuta los siguientes comandos:

```bash
# 1. Verificar que los 2 worker nodes estén Ready
kubectl get nodes -o wide

# 2. Clonar tu repositorio desde GitHub
git clone https://github.com/marco-mz9/K8sMovieApp.git
cd K8sMovieApp

# 3. Aplicar primero el Namespace y luego todos los manifiestos
kubectl apply -f 00-namespace.yaml
kubectl apply -f .
```

---

## Paso 2: Verificar el despliegue y capturar evidencias para el PDF

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
