# Tienda de Alimentos para Perritos 

Aplicación web de gestión CRUD de productos, desarrollada como parte de la asignatura **Introducción a Herramientas DevOps (ISY1101)**. El proyecto está compuesto por tres servicios (frontend, backend y base de datos) contenerizados con Docker y desplegados en un clúster **Amazon EKS**, con un pipeline de integración y entrega continua (CI/CD) automatizado mediante **GitHub Actions**.

## Descripción del proyecto

La aplicación permite gestionar el catálogo de productos de una tienda de alimentos para mascotas (crear, editar, eliminar y listar productos). Está pensada como un caso práctico de orquestación y automatización en la nube para Tienda Perritos.

## Arquitectura

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│  Frontend   │ ───► │   Backend   │ ───► │    MySQL    │
│  (Nginx)    │      │ (Node/Expr) │      │   (RDBMS)   │
└─────────────┘      └─────────────┘      └─────────────┘
      │                     │                     │
      └─────────────────────┴─────────────────────┘
                    Kubernetes (EKS)
```

- **Frontend:** servido con Nginx, expone la interfaz de usuario y consume la API del backend.
- **Backend:** API REST en Node.js/Express, gestiona la lógica de negocio y la conexión a la base de datos.
- **Base de datos:** MySQL, con persistencia y credenciales gestionadas mediante Kubernetes Secrets.
- **Clúster:** Amazon EKS (`tienda-perritos-eks-cluster`, región `us-east-1`), respaldado por una VPC con subredes públicas y privadas.
- **Registro de imágenes:** Amazon ECR (un repositorio por servicio).
- **Autoscaling:** Horizontal Pod Autoscaler (HPA) en frontend y backend, basado en uso de CPU.
- **CI/CD:** GitHub Actions automatiza build → push a ECR → deploy en EKS ante cada push a `main`.

## Estructura del repositorio

```
├── backend/                 # Código fuente y Dockerfile del backend
├── frontend/                # Código fuente y Dockerfile del frontend
├── k8s/                     # Manifiestos de Kubernetes
│   ├── namespace.yaml
│   ├── mysql-secret.yaml
│   ├── mysql-deployment.yaml
│   ├── mysql-service.yaml
│   ├── backend-deployment.yaml
│   ├── backend-service.yaml
│   ├── backend-hpa.yaml
│   ├── frontend-deployment.yaml
│   ├── frontend-service.yaml
│   └── frontend-hpa.yaml
├── .github/workflows/       # Definición del pipeline CI/CD
└── README.md
```

## Requisitos previos

- Cuenta de AWS Academy (Learner Lab) con permisos para EKS, ECR y VPC.
- Herramientas instaladas localmente: Docker Desktop, AWS CLI, kubectl, Git.
- Clúster EKS ya creado y configurado (ver informe técnico para el detalle paso a paso).

## Despliegue manual (resumen)

1. **Autenticarse en AWS y en ECR:**
   ```bash
   aws configure
   aws eks update-kubeconfig --region us-east-1 --name tienda-perritos-eks-cluster
   aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com
   ```

2. **Construir y subir las imágenes a ECR** (repetir para backend, frontend y db):
   ```bash
   docker build -t tienda-backend ./backend
   docker tag tienda-backend:latest <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/tienda-backend:latest
   docker push <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/tienda-backend:latest
   ```

3. **Aplicar los manifiestos de Kubernetes en orden:**
   ```bash
   kubectl apply -f k8s/namespace.yaml
   kubectl apply -f k8s/mysql-secret.yaml
   kubectl apply -f k8s/mysql-deployment.yaml
   kubectl apply -f k8s/mysql-service.yaml
   kubectl apply -f k8s/backend-deployment.yaml
   kubectl apply -f k8s/backend-service.yaml
   kubectl apply -f k8s/frontend-deployment.yaml
   kubectl apply -f k8s/frontend-service.yaml
   kubectl apply -f k8s/backend-hpa.yaml
   kubectl apply -f k8s/frontend-hpa.yaml
   ```

4. **Obtener la URL pública del frontend:**
   ```bash
   kubectl get svc tienda-frontend -n tienda
   ```
   Copiar el valor de `EXTERNAL-IP` y acceder desde el navegador.

## Despliegue automático (CI/CD)

El pipeline definido en `.github/workflows/` se ejecuta automáticamente con cada `push` a la rama `main` y realiza:

1. **Build** de las imágenes Docker de backend y frontend.
2. **Push** de las imágenes a Amazon ECR.
3. **Deploy** actualizando los Deployments en el clúster EKS (`kubectl set image` + `kubectl rollout status`).

### Configuración necesaria

El pipeline requiere los siguientes **Secrets** configurados en `Settings > Secrets and variables > Actions` del repositorio:

| Secret | Descripción |
|---|---|
| `AWS_ACCESS_KEY_ID` | Access Key del usuario/rol de AWS |
| `AWS_SECRET_ACCESS_KEY` | Secret Key correspondiente |
| `AWS_SESSION_TOKEN` | Token de sesión temporal (AWS Academy) |
| `AWS_REGION` | Región de despliegue (`us-east-1`) |
| `EKS_CLUSTER_NAME` | Nombre del clúster EKS |
| `EKS_NAMESPACE` | Namespace de Kubernetes (`tienda`) |

## Verificación y monitoreo

```bash
# Estado de los pods
kubectl get pods -n tienda

# Estado del autoscaling
kubectl get hpa -n tienda

# Logs del backend
kubectl logs -n tienda deployment/tienda-backend

# Estado del rollout tras un deploy
kubectl rollout status deployment/tienda-backend -n tienda
```

## Pruebas realizadas

- **Autoscaling (HPA):** simulación de carga con `curl` en bucle desde dentro del pod, escalando el backend de 2 a 8 réplicas.
- **Auto-healing:** eliminación manual de un pod (`kubectl delete pod`), verificando su recreación automática en ~6 segundos.
- **Zero-downtime deploy:** validado con `kubectl rollout status` tras actualizaciones vía pipeline.

## Integrantes

- Yannella Castilla Santi

## Asignatura

Introducción a Herramientas DevOps (ISY1101) — Sección 002D — DuocUC
