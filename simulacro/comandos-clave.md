# 🛠️ Cheat Sheet — Comandos Clave para el Examen

## Docker

## Crear e iniciar el nuevo perfil
```bash
#Crear un nuevo perfil
minikube start -p simulacro

# ver que perfiles tienes activos
minikube profile list

# activar el perfil de simulacro
& minikube -p simulacro docker-env | Invoke-Expression
```

### Construcción y Ejecución
```bash
# Construir imagen
docker build -t mi-imagen:tag .
docker build -t inventario-app:v1 .
docker build -t inventario-app:v2 .

# Construir con Dockerfile específico
docker build -f Dockerfile.roto -t mi-imagen .

# Ejecutar contenedor
docker run -d -p <host_port>:<container_port> --name mi-app mi-imagen

# --name mi-app es para darle un nombre al contenedor
# mi-imagen es la imagen que se va a ejecutar

# Ejecutar con variable de entorno
docker run -d -p 3000:3000 -e PORT=3000 --name mi-app mi-imagen

# Para forzar otro puerto
docker run -d -p 4000:4000 -e PORT=4000 --name mi-app mi-imagen
```

### Diagnóstico
```bash
# Ver contenedores activos
docker ps

# Ver TODOS los contenedores (incluye detenidos)
docker ps -a

# Ver logs del contenedor
docker logs <id_o_nombre>

# Seguir logs en tiempo real
docker logs -f <id_o_nombre>

# Entrar al contenedor (terminal interactiva)
docker exec -it <id_o_nombre> sh

# Inspeccionar detalles del contenedor
docker inspect <id_o_nombre>

# Ver puertos mapeados
docker port <id_o_nombre>
```

### Limpieza
```bash
# Detener contenedor
docker stop <id_o_nombre>

# Eliminar contenedor
docker rm <id_o_nombre>

# Eliminar imagen
docker rmi mi-imagen:tag

# Detener y eliminar todo
docker stop $(docker ps -q) && docker rm $(docker ps -aq)
```

---

## Kubernetes

### Pods
```bash
# Ver pods
kubectl get pods

# Ver pods con labels
kubectl get pods --show-labels

# Ver pods con selector específico
kubectl get pods -l app=inventario-app

# Ver detalles de un pod
kubectl describe pod <nombre_pod>

# Ver logs de un pod
kubectl logs <nombre_pod>

# Entrar a un pod
kubectl exec -it <nombre_pod> -- sh
```

### Deployments
```bash
# Ver deployments
kubectl get deployments

# Aplicar/actualizar deployment
kubectl apply -f deployment.yaml

# Escalar manualmente
kubectl scale deployment inventario-app --replicas=6

# Ver estado del rollout
kubectl rollout status deployment/inventario-app

# Historial de rollouts
kubectl rollout history deployment/inventario-app

# Rollback al deployment anterior
kubectl rollout undo deployment/inventario-app

# Actualizar imagen directamente
kubectl set image deployment/inventario-app inventario-app=nueva-imagen:tag
```

### Services
```bash
# Ver services
kubectl get services

# Describir service (ver selector, endpoints, etc.)
kubectl describe service inventario-service

# Ver endpoints del service
kubectl get endpoints inventario-service

# Acceder al service (port-forward a tu máquina)
kubectl port-forward service/inventario-service 8080:80
# Luego: curl http://localhost:8080/health
```

### HPA (Autoescalado)
```bash
# Ver HPAs
kubectl get hpa

# Describir HPA (ver métricas, estado)
kubectl describe hpa inventario-hpa

# Crear HPA rápido
kubectl autoscale deployment inventario-app --min=3 --max=10 --cpu-percent=50
```

### Secrets
```bash
# Ver secrets
kubectl get secrets

# Crear secret desde YAML
kubectl apply -f secret.yaml

# Ver contenido de un secret (base64)
kubectl get secret app-secrets -o yaml

# Decodificar un valor
echo "bXktc2VjcmV0" | base64 --decode
```

### Debugging General
```bash
# Ver eventos recientes del cluster
kubectl get events --sort-by=.metadata.creationTimestamp

# Ver TODOS los recursos
kubectl get all

# Eliminar y recrear
kubectl delete -f deployment.yaml && kubectl apply -f deployment.yaml
```

---

## GitHub Actions / CI-CD

### Estructura clave del workflow
```yaml
name: ci-cd
on:
  push:
    branches: [main]

jobs:
  # Job 1
  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test

  # Job 2 — DEPENDE del Job 1
  deploy:
    runs-on: ubuntu-latest
    needs: build-test          # ← CLAVE: sin esto, corre en paralelo
    steps:
      - run: echo "Deployando..."
```

### Palabras clave importantes
```yaml
needs: build-test              # Dependencia entre jobs
continue-on-error: true        # El job pasa aunque falle (PELIGROSO)
if: success()                  # Solo corre si los anteriores pasaron
if: failure()                  # Solo corre si algo falló
if: always()                   # Corre siempre
```

### Variables y Secretos en Actions
```yaml
${{ github.sha }}              # SHA del commit actual
${{ github.actor }}            # Usuario que hizo push
${{ secrets.GITHUB_TOKEN }}    # Token automático de GitHub
${{ secrets.MI_SECRETO }}      # Secret configurado en Settings
```

---

## Generar Tráfico de Prueba (para verificar zero-downtime)

### Con curl en loop
```bash
# Tráfico continuo cada 0.5 segundos
while true; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/health)
  echo "$(date +%H:%M:%S) - Status: $STATUS"
  sleep 0.5
done
```

### Con watch
```bash
# Monitorear pods cada 2 segundos
watch -n 2 kubectl get pods

# Monitorear endpoints cada 2 segundos
watch -n 2 kubectl get endpoints inventario-service
```

---

## Flujo del Examen (Resumen)

```
┌─────────────────────────────────────────────┐
│  MINUTO 0-15: Leer y entender los archivos  │
│  → docker logs, kubectl describe, leer YAML │
├─────────────────────────────────────────────┤
│  MINUTO 15-25: Reto 1 (Docker)              │
│  → Verificar puerto, corregir Dockerfile    │
├─────────────────────────────────────────────┤
│  MINUTO 25-35: Reto 2 (K8s labels)          │
│  → kubectl get endpoints, corregir labels   │
├─────────────────────────────────────────────┤
│  MINUTO 35-45: Reto 3 (CI/CD)              │
│  → Agregar needs, verificar con test roto   │
├─────────────────────────────────────────────┤
│  MINUTO 45: ¡GIRO FINAL!                   │
│  → Escalar réplicas + rolling update        │
│  → maxUnavailable: 0 + readinessProbe      │
├─────────────────────────────────────────────┤
│  MINUTO 45-85: Implementar + verificar      │
│  → Generar tráfico + desplegar              │
├─────────────────────────────────────────────┤
│  MINUTO 85-90: Commits + evidencia          │
│  → git commit para antes/después            │
│  → Capturas de pantalla                     │
└─────────────────────────────────────────────┘
```

---

## Git (para historial de evidencia)

```bash
# Commit del estado "roto" (antes de corregir)
git add .
git commit -m "Estado roto: archivos con defectos para diagnóstico"

# Después de cada corrección
git add .
git commit -m "Reto 1: Corregido puerto en Dockerfile (4000→3000)"
git commit -m "Reto 2: Corregidos labels en deployment (web→webapp)"
git commit -m "Reto 3: Agregado needs al job deploy"
git commit -m "Giro final: 6 réplicas + rolling update zero-downtime"
```
