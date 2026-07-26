# 📖 Guía de Estudio Completa — Simulacro de Examen

## Índice
1. [Reto 1: Dockerfile con puerto incorrecto](#reto-1)
2. [Reto 2: Labels de K8s desalineados](#reto-2)
3. [Reto 3: Pipeline sin dependencia entre jobs](#reto-3)
4. [Reto 4: Giro Final — Escalado y Zero-Downtime Deploy](#reto-4)
5. [Buenas Prácticas (las 3 que mencionó tu profesor)](#buenas-practicas)
6. [Errores Alternativos que podrían aparecer](#errores-alternativos)

---

## <a id="reto-1"></a>Reto 1: El contenedor que "corre" pero no responde

### 🔴 El problema
```dockerfile
EXPOSE 4000          # Dockerfile dice 4000
```
```javascript
const PORT = process.env.PORT || 3000;   // La app escucha en 3000
```
El contenedor arranca sin error. `docker ps` muestra que está "Up". Pero `curl http://localhost:4000/health` no responde.

### 🔍 Cómo diagnosticar

**Paso 1: Ver los logs del contenedor**
```bash
docker logs <id_contenedor>
# Salida esperada: "Servidor escuchando en puerto 3000"
# ¡Ahí está la pista! Dice 3000, no 4000
```

**Paso 2: Entrar al contenedor y verificar**
```bash
docker exec -it <id_contenedor> sh

# Dentro del contenedor:
wget -qO- http://localhost:3000/health
# Responde: {"status":"ok"}

wget -qO- http://localhost:4000/health
# No responde — confirma que la app NO escucha en 4000
```

**Paso 3: Verificar puertos desde afuera**
```bash
# Si corriste: docker run -p 4000:4000 ...
# El mapeo es: host:4000 → contenedor:4000
# Pero la app escucha en contenedor:3000
# Resultado: el tráfico llega al 4000 del contenedor, donde nadie escucha
```

### ✅ La corrección

**Opción A: Corregir el Dockerfile**
```dockerfile
EXPOSE 3000          # Cambiar a 3000
```
Y correr: `docker run -p 3000:3000 inventario-app`

**Opción B: Corregir el docker run (sin cambiar Dockerfile)**
```bash
docker run -p 4000:3000 inventario-app
# Mapea host:4000 → contenedor:3000 (donde SÍ escucha la app)
```

### ⚡ Concepto clave
> `EXPOSE` es solo documentación. Lo que importa es el `-p` de `docker run`.
> El puerto real es el que usa `app.listen(PORT)` en el código.

---

## <a id="reto-2"></a>Reto 2: Pods Running, Service sin respuesta

### 🔴 El problema
```yaml
# Deployment template (lo que tienen los pods)
template:
  metadata:
    labels:
      app: web           # ← Los pods tienen "web"

# Deployment selector
selector:
  matchLabels:
    app: webapp          # ← Busca "webapp" (diferente!)

# Service selector
selector:
  app: webapp            # ← Busca "webapp" (los pods tienen "web")
```

### 🔍 Cómo diagnosticar

**Paso 1: Verificar que los pods están corriendo**
```bash
kubectl get pods -l app=web
# Muestra 2 pods en Running ✓

kubectl get pods -l app=webapp
# No muestra nada ← PISTA: el Service busca "webapp" pero no hay pods con esa label
```

**Paso 2: Describir el Service**
```bash
kubectl describe service inventario-service
# En la sección "Endpoints": <none>
# ← PISTA: No tiene endpoints = no hay pods que coincidan con su selector
```

**Paso 3: Ver endpoints directamente**
```bash
kubectl get endpoints inventario-service
# ENDPOINTS: <none>
# Confirma: el Service no tiene a quién enviar tráfico
```

**Paso 4: Comparar labels**
```bash
kubectl get pods --show-labels
# NAME                    LABELS
# inventario-xxx-abc      app=web     ← tiene "web"
# inventario-xxx-def      app=web     ← tiene "web"

kubectl get service inventario-service -o yaml | grep -A2 selector
# selector:
#   app: webapp                       ← busca "webapp" ≠ "web"
```

### ✅ La corrección

Cambiar el template del Deployment para que las labels coincidan:
```yaml
template:
  metadata:
    labels:
      app: webapp        # ← Cambiar "web" a "webapp" (coincidir con selector)
```

**O** cambiar todo a `inventario-app` (como en tu proyecto real):
```yaml
selector:
  matchLabels:
    app: inventario-app
template:
  metadata:
    labels:
      app: inventario-app
# Y en el Service:
selector:
  app: inventario-app
```

### ⚡ Concepto clave
> El **selector del Service** busca pods por labels. Si las labels del pod no coinciden
> exactamente con el selector, el Service no tiene endpoints y no puede enrutar tráfico.
>
> La cadena es: **Service.selector** → debe coincidir con → **Pod.labels**
> (que se definen en **Deployment.template.metadata.labels**)

---

## <a id="reto-3"></a>Reto 3: Pipeline que despliega con tests rotos

### 🔴 El problema
```yaml
jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test           # Tests que pueden fallar

  deploy:
    runs-on: ubuntu-latest
    # ← FALTA: needs: build-test
    steps:
      - run: kubectl set image...   # Se ejecuta SIEMPRE
```

Sin `needs`, GitHub Actions ejecuta ambos jobs **en paralelo**. El deploy no espera a que build-test termine.

### 🔍 Cómo diagnosticar

**Paso 1: Observar la ejecución del pipeline**
En GitHub Actions, verás que `build-test` y `deploy` arrancan al mismo tiempo (en paralelo), sin flechas de dependencia.

**Paso 2: Romper un test a propósito**
```javascript
// En server.test.js, cambiar:
assert.strictEqual(res.status, 200);
// A:
assert.strictEqual(res.status, 999);  // Siempre falla
```

**Paso 3: Verificar que deploy se ejecuta aunque build-test falle**
Hacer push → en Actions → `build-test` falla ❌ pero `deploy` sigue en verde ✓

### ✅ La corrección

Agregar `needs: build-test` al job deploy:
```yaml
  deploy:
    runs-on: ubuntu-latest
    needs: build-test           # ← AGREGAR ESTA LÍNEA
    steps:
      - run: kubectl set image...
```

### 🔍 Verificar la corrección
1. Romper un test → push → `build-test` falla → `deploy` **NO se ejecuta** ✓
2. Arreglar el test → push → `build-test` pasa → `deploy` se ejecuta ✓

### ⚡ Concepto clave
> En GitHub Actions, los **jobs son independientes por defecto** y se ejecutan en paralelo.
> Para crear una dependencia secuencial, debes usar la palabra clave `needs`.
>
> ```yaml
> deploy:
>   needs: build-test    # deploy solo corre si build-test fue exitoso
> ```
> Si `build-test` falla, `deploy` se salta automáticamente.

---

## <a id="reto-4"></a>Reto 4: Giro Final — Escalado + Zero-Downtime Deploy

### 📋 Contexto
"El tráfico se va a triplicar y el despliegue no puede causar interrupción."

### ✅ Solución Parte 1: Escalar réplicas

**Opción A: Manual — Cambiar replicas en el YAML**
```yaml
spec:
  replicas: 6    # Triplicar de 2 a 6
```

**Opción B: Comando directo**
```bash
kubectl scale deployment inventario-app --replicas=6
```

**Opción C: HorizontalPodAutoscaler (más avanzado)**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: inventario-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: inventario-app
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

> **IMPORTANTE**: Para que HPA funcione, necesitas `resources.requests` en el container:
> ```yaml
> resources:
>   requests:
>     cpu: 100m
>     memory: 128Mi
> ```

### ✅ Solución Parte 2: Rolling Update sin downtime

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0    # CLAVE: nunca tener 0 pods disponibles
    maxSurge: 2          # Crear hasta 2 pods extra durante el update
```

**¿Cómo funciona?**
1. K8s crea 2 pods nuevos (con la nueva versión)
2. Espera a que estén Ready (pasan readinessProbe)
3. Termina 2 pods viejos
4. Repite hasta que todos los pods sean nuevos
5. **En ningún momento hay 0 pods sirviendo** (maxUnavailable: 0)

### 🔍 Verificar zero-downtime

**Generar tráfico de prueba mientras se despliega:**
```bash
# Terminal 1: Generar tráfico continuo
while true; do curl -s http://localhost/health && echo " OK" || echo " FAIL"; sleep 0.5; done

# Terminal 2: Aplicar el nuevo deployment
kubectl apply -f deployment-escalado.yaml

# Observar Terminal 1: NO debe haber ningún "FAIL"
```

**Monitorear el rollout:**
```bash
kubectl rollout status deployment/inventario-app
# Waiting for deployment "inventario-app" rollout to finish: 2 of 6 updated replicas are available...
# deployment "inventario-app" successfully rolled out
```

### ⚡ Conceptos clave
> - `maxUnavailable: 0` = zero downtime (siempre hay pods sirviendo)
> - `maxSurge: 2` = velocidad del rollout (más surge = más rápido)
> - `readinessProbe` es ESENCIAL: sin ella, K8s envía tráfico a pods que aún no están listos
> - `kubectl rollout undo deployment/inventario-app` = rollback si algo sale mal

---

## <a id="buenas-practicas"></a>Buenas Prácticas (las 3 que mencionó tu profesor)

### 1. Manejo de Secretos

**❌ MAL: Hardcodear secretos**
```yaml
env:
- name: API_KEY
  value: "mi-secreto-123"     # ← Visible en el YAML y en git
```

**✅ BIEN: Usar Kubernetes Secrets**
```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  API_KEY: "mi-secreto-123"
```
```yaml
# deployment.yaml
env:
- name: API_KEY
  valueFrom:
    secretKeyRef:
      name: app-secrets
      key: API_KEY
```

> **Tip**: En producción real, los secretos se manejan con herramientas como
> Sealed Secrets, HashiCorp Vault, o el secret manager del cloud provider.

### 2. Escaneo de Seguridad en CI (Trivy)

```yaml
# En el pipeline de CI/CD
- name: Escanear imagen con Trivy
  uses: aquasecurity/trivy-action@v0.36.0
  with:
    image-ref: mi-imagen:latest
    format: table
    exit-code: '1'           # Falla el pipeline si hay vulnerabilidades
    ignore-unfixed: true     # Ignora vulnerabilidades sin fix disponible
    severity: CRITICAL       # Solo reporta vulnerabilidades CRITICAL
```

> `exit-code: '1'` hace que Trivy **falle el pipeline** si encuentra vulnerabilidades.
> Sin esto, solo reporta pero deja pasar.

### 3. Readiness Probe con arranque lento

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 12    # Esperar 12s antes del primer chequeo
  periodSeconds: 5           # Chequear cada 5s después
  failureThreshold: 3        # 3 fallos seguidos = marcar como "no ready"
```

> **¿Por qué importa?** Si la app tarda en arrancar (conectar a BD, cargar cache, etc.),
> sin readinessProbe K8s enviaría tráfico a un pod que aún no puede responder.
> Con `initialDelaySeconds` le das tiempo para arrancar.

---

## <a id="errores-alternativos"></a>Errores Alternativos que Podrían Aparecer

Tu profesor podría cambiar los errores específicos. Aquí están las variaciones más comunes:

### Variaciones del Reto 1 (Docker)

| Error | Síntoma | Diagnóstico |
|-------|---------|-------------|
| `EXPOSE` incorrecto | Contenedor "Up" pero sin respuesta | `docker logs` muestra puerto real |
| `CMD ["node", "app.js"]` (archivo no existe) | Contenedor se cae inmediatamente | `docker logs` muestra `Cannot find module` |
| Falta `COPY . .` | App arranca pero sin código | `docker exec` → `ls` no muestra archivos |
| `WORKDIR /wrong` | Dependencias no encontradas | `docker logs` muestra `MODULE_NOT_FOUND` |
| No instalar dependencias (`RUN npm install` falta) | Build falla o app crash | Error `Cannot find module 'express'` |
| Puerto correcto en EXPOSE pero `-p` mal en docker run | Contenedor OK internamente, no accesible desde fuera | `docker exec` + `wget` funciona, `curl` desde host no |

### Variaciones del Reto 2 (Kubernetes)

| Error | Síntoma | Diagnóstico |
|-------|---------|-------------|
| Labels no coinciden (selector vs template) | Endpoints vacíos | `kubectl get endpoints` → `<none>` |
| `targetPort` incorrecto en Service | Connection refused | `kubectl describe svc` → targetPort ≠ containerPort |
| `containerPort` diferente al puerto real de la app | Pod Running pero liveness falla | `kubectl describe pod` → probe failed |
| Namespace diferente entre Deployment y Service | Service no encuentra pods | `kubectl get pods -n <ns>` en namespace correcto |
| `imagePullPolicy: Never` con imagen remota | Pod en `ImagePullBackOff` | `kubectl describe pod` → pull error |

### Variaciones del Reto 3 (CI/CD)

| Error | Síntoma | Diagnóstico |
|-------|---------|-------------|
| Falta `needs` | Deploy corre aunque test falle | Ver ejecución paralela en Actions |
| `on: pull_request` en vez de `push` | Pipeline no se dispara en push a main | Verificar triggers en el YAML |
| Secret no configurado en GitHub | Job falla en paso de login/push | Error "secret not found" en logs |
| `continue-on-error: true` en tests | Pipeline pasa aunque test falle | Buscar la directiva en el YAML |
| Tag de imagen mal formada | Push de imagen falla | Error de formato en Docker |

---

## 🧠 Mentalidad para el Examen

1. **Lee los logs primero**: `docker logs`, `kubectl logs`, `kubectl describe`
2. **Compara siempre**: puerto en código vs Dockerfile vs docker run vs K8s
3. **Verifica la cadena completa**: labels → selector → endpoints → service
4. **No cambies todo**: identifica el error específico y haz el cambio mínimo
5. **Documenta**: haz commits antes y después para mostrar el historial
