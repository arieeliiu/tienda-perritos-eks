# Tienda Perritos EKS

Proyecto de orquestación y CI/CD desplegado en AWS EKS.

La aplicación está compuesta por:

- Frontend
- Backend
- Base de datos MySQL
- Manifiestos Kubernetes
- Pipeline CI/CD con GitHub Actions

## Arquitectura general

La solución utiliza Amazon EKS para ejecutar los contenedores, Amazon ECR para almacenar las imágenes Docker y GitHub Actions para automatizar el proceso de build, push y deploy.

El frontend se expone públicamente mediante un servicio LoadBalancer, mientras que el backend y la base de datos se comunican internamente dentro del clúster mediante servicios de Kubernetes.

## Tecnologías utilizadas

- AWS EKS
- AWS ECR
- AWS VPC
- Kubernetes
- Docker
- GitHub Actions
- CloudWatch

## Estado

Proyecto desarrollado como parte de la Evaluación Parcial N°3 de Introducción a Herramientas DevOps.
