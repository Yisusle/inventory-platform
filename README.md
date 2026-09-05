# Inventory Platform

Entorno reproducible para desplegar el dashboard Angular, la Inventory API .NET
y SQL Server como una única plataforma. El dashboard se sirve desde Nginx y
redirige internamente las solicitudes `/api` a la API, por lo que el navegador
no necesita conocer una URL separada ni depender de CORS en producción.

## Arquitectura

```text
Browser :8080
     |
Nginx / Angular dashboard
     | /api
Inventory API (.NET 9)
     |
SQL Server
```

Los tres servicios comparten una red Docker privada. Sólo el dashboard expone
un puerto al equipo anfitrión. SQL Server y la API no son accesibles desde fuera
de la red de contenedores.

## Requisitos

- Docker Desktop con Docker Compose v2.
- Los repositorios `inventory-api`, `inventory-dashboard` e
  `inventory-platform` clonados como carpetas hermanas.

```text
MejoraDeVida/
├── inventory-api/
├── inventory-dashboard/
└── inventory-platform/
```

## Arranque local

```powershell
Copy-Item .env.example .env
# Edita .env y asigna secretos locales seguros.
docker compose up --build
```

Abre `http://localhost:8080`. La API queda accesible para el dashboard mediante
`/api`; no se publica directamente en el host. Para detener los servicios:

```powershell
docker compose down
```

Para detenerlos y borrar la base de datos local:

```powershell
docker compose down --volumes
```

`MSSQL_SA_PASSWORD` debe cumplir los requisitos de complejidad de SQL Server y
`JWT_SECRET` debe tener al menos 32 caracteres. `.env` está ignorado por Git.

## Estado de validación

- Docker Compose fue probado localmente con SQL Server, la API y el dashboard.
- Nginx fue validado como proxy de `/api` desde el dashboard hacia la API.
- Kubernetes fue probado localmente con SQL Server, la API y el dashboard.
- El flujo de login desde el dashboard hacia la API funcionó correctamente.
- El manifiesto `kubernetes/local.yaml` se usa únicamente para la prueba local y
  permanece excluido del repositorio porque contiene secretos de prueba.

## Imágenes publicadas

Los repositorios de API y dashboard construyen y publican, desde `main` o
`master`, estas imágenes en GitHub Container Registry:

```text
ghcr.io/<github-owner>/inventory-api:<tag>
ghcr.io/<github-owner>/inventory-dashboard:<tag>
```

Antes del despliegue Kubernetes, sustituye `replace-with-github-owner` por el
propietario real. Para versiones reproducibles, sustituye `latest` por el SHA
de un commit publicado.

## Kubernetes

Kubernetes usa una base de datos externa o administrada; no es recomendable
ejecutar SQL Server como un StatefulSet sin una estrategia de almacenamiento,
backup y recuperación adecuada.

1. Copia `kubernetes/secrets.example.yaml` fuera del repositorio como
   `secrets.yaml`, completa la cadena de conexión y el secreto JWT.
2. Reemplaza `REPLACE_WITH_GITHUB_OWNER` en `kubernetes/api.yaml` y
   `kubernetes/dashboard.yaml`.
3. Reemplaza `inventory.example.com` en `kubernetes/api.yaml` e
   `kubernetes/ingress.yaml` por el dominio público real.
4. Asegura que el clúster tenga un controlador NGINX Ingress y acceso a las
   imágenes de GHCR. Si las imágenes son privadas, crea un `imagePullSecret`.
5. Aplica los manifiestos:

```powershell
kubectl apply -f kubernetes/namespace.yaml
kubectl apply -f kubernetes/secrets.yaml
kubectl apply -f kubernetes/api.yaml
kubectl apply -f kubernetes/dashboard.yaml
kubectl apply -f kubernetes/ingress.yaml
```

El Ingress expone únicamente el dashboard. Nginx dentro del contenedor del
dashboard reenvía `/api` al servicio `inventory-api`, que permanece interno.
