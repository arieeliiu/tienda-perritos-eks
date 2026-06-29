# Tienda Perritos EKS

Proyecto DevOps desarrollado para desplegar la aplicación **Tienda Perritos** en AWS utilizando contenedores, Kubernetes, Amazon EKS, Amazon ECR y GitHub Actions.

La solución permite construir imágenes Docker, publicarlas en Amazon ECR, desplegar servicios en Amazon EKS, exponer el frontend mediante Load Balancer, aplicar autoscaling con HPA y automatizar el despliegue mediante un pipeline CI/CD.

## Arquitectura general

La aplicación está compuesta por tres servicios principales:

* **Frontend**: interfaz web accesible públicamente mediante un Load Balancer.
* **Backend**: API interna desplegada en Kubernetes.
* **Base de datos**: MySQL desplegado dentro del clúster.

El frontend se comunica con el backend dentro del clúster, y el backend se comunica con la base de datos mediante servicios internos de Kubernetes.

## Tecnologías utilizadas

* AWS EKS
* AWS ECR
* AWS VPC
* Kubernetes
* Docker
* GitHub Actions
* CloudWatch
* kubectl
* AWS CLI

## Estructura del proyecto

```text
tienda-perritos-eks/
├── backend/
├── frontend/
├── db/
├── k8s/
│   ├── namespace.yaml
│   ├── mysql-secret.yaml
│   ├── mysql-deployment.yaml
│   ├── mysql-service.yaml
│   ├── backend-deployment.yaml
│   ├── backend-service.yaml
│   ├── frontend-deployment.yaml
│   ├── frontend-service.yaml
│   ├── backend-hpa.yaml
│   └── frontend-hpa.yaml
└── .github/
    └── workflows/
        └── deploy-eks.yml
```

## Requisitos

Para ejecutar el proyecto se requiere:

* Cuenta AWS Academy activa.
* AWS CLI configurado.
* Docker Desktop instalado.
* kubectl instalado.
* Git instalado.
* Repositorios ECR creados:

  * tienda-frontend
  * tienda-backend
  * tienda-db
* Clúster EKS creado y operativo.
* Namespace `tienda` creado en Kubernetes.

## Configuración de AWS CLI

```powershell
aws configure
```

Se deben ingresar las credenciales temporales entregadas por AWS Academy:

* AWS Access Key ID
* AWS Secret Access Key
* AWS Session Token
* Región: us-east-1

Validar conexión:

```powershell
aws sts get-caller-identity
```

## Construcción y publicación manual de imágenes

Obtener Account ID:

```powershell
$ACCOUNT_ID = (aws sts get-caller-identity --query Account --output text).Trim()
```

Login en ECR:

```powershell
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin "$ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com"
```

Construir imágenes:

```powershell
docker build -t tienda-frontend:eks-v1 ./frontend
docker build -t tienda-backend:eks-v1 ./backend
docker build -t tienda-db:eks-v1 ./db
```

Etiquetar imágenes:

```powershell
docker tag tienda-frontend:eks-v1 "$ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/tienda-frontend:eks-v1"
docker tag tienda-backend:eks-v1 "$ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/tienda-backend:eks-v1"
docker tag tienda-db:eks-v1 "$ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/tienda-db:eks-v1"
```

Subir imágenes a ECR:

```powershell
docker push "$ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/tienda-frontend:eks-v1"
docker push "$ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/tienda-backend:eks-v1"
docker push "$ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/tienda-db:eks-v1"
```

## Despliegue manual en EKS

Conectar kubectl al clúster:

```powershell
aws eks update-kubeconfig --region us-east-1 --name test-eks
```

Crear namespace:

```powershell
kubectl apply -f k8s/namespace.yaml
```

Desplegar base de datos:

```powershell
kubectl apply -f k8s/mysql-secret.yaml -n tienda
kubectl apply -f k8s/mysql-deployment.yaml -n tienda
kubectl apply -f k8s/mysql-service.yaml -n tienda
```

Desplegar backend:

```powershell
kubectl apply -f k8s/backend-deployment.yaml -n tienda
kubectl apply -f k8s/backend-service.yaml -n tienda
```

Desplegar frontend:

```powershell
kubectl apply -f k8s/frontend-deployment.yaml -n tienda
kubectl apply -f k8s/frontend-service.yaml -n tienda
```

Aplicar HPA:

```powershell
kubectl apply -f k8s/backend-hpa.yaml -n tienda
kubectl apply -f k8s/frontend-hpa.yaml -n tienda
```

## Validación del despliegue

Ver pods:

```powershell
kubectl get pods -n tienda
```

Ver servicios:

```powershell
kubectl get svc -n tienda
```

Ver autoscaling:

```powershell
kubectl get hpa -n tienda
```

Ver métricas:

```powershell
kubectl top nodes
kubectl top pods -n tienda
```

Ver logs del backend:

```powershell
kubectl logs -n tienda -l app=tienda-backend
```

## Acceso a la aplicación

Para obtener la URL pública del frontend:

```powershell
kubectl get svc tienda-frontend -n tienda
```

El valor de `EXTERNAL-IP` corresponde al Load Balancer público generado por AWS.

## Pipeline CI/CD

El proyecto incluye un workflow de GitHub Actions ubicado en:

```text
.github/workflows/deploy-eks.yml
```

El pipeline se ejecuta automáticamente al realizar un push a la rama `main`.

El flujo realiza:

1. Autenticación en AWS.
2. Login en Amazon ECR.
3. Build de imágenes Docker.
4. Push de imágenes a Amazon ECR.
5. Conexión al clúster EKS.
6. Aplicación de manifiestos Kubernetes.
7. Actualización de deployments.
8. Validación de pods, servicios y HPA.

## Secrets requeridos en GitHub Actions

En el repositorio se deben configurar los siguientes secrets:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_SESSION_TOKEN
AWS_REGION
EKS_CLUSTER_NAME
EKS_NAMESPACE
```

Valores utilizados:

```text
AWS_REGION = us-east-1
EKS_CLUSTER_NAME = test-eks
EKS_NAMESPACE = tienda
```

Las credenciales AWS deben obtenerse desde AWS Academy Learner Lab.

## Validación del pipeline

Luego de ejecutar el pipeline, se puede validar que los deployments utilizan el tag del commit:

```powershell
kubectl get deployment tienda-backend tienda-frontend -n tienda -o custom-columns=NAME:.metadata.name,IMAGE:.spec.template.spec.containers[0].image
```

También se puede validar en Amazon ECR que las imágenes fueron publicadas con el tag correspondiente al commit ejecutado por GitHub Actions.

## Autoscaling y recuperación

El proyecto utiliza Horizontal Pod Autoscaler para frontend y backend.

Para validar escalamiento:

```powershell
kubectl get hpa -n tienda -w
```

Para validar auto-healing:

```powershell
$POD = kubectl get pods -n tienda -l app=tienda-backend -o jsonpath="{.items[0].metadata.name}"
kubectl delete pod $POD -n tienda
kubectl get pods -n tienda -l app=tienda-backend -w
```

Kubernetes crea automáticamente un nuevo pod para mantener el número de réplicas definido.

## Estado final esperado

Al finalizar, el entorno debe mostrar:

* Pods de base de datos, backend y frontend en estado `Running`.
* Servicio frontend expuesto mediante `LoadBalancer`.
* Backend y base de datos comunicándose internamente.
* HPA activo para frontend y backend.
* Pipeline CI/CD ejecutado correctamente desde GitHub Actions.
* Imágenes publicadas en Amazon ECR con tag del commit.
