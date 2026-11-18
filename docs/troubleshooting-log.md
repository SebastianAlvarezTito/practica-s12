# Bitácora de Troubleshooting — S12 | AP9

**Proyecto:** Despliegue Cloud, Seguridad y Troubleshooting (Kubernetes en AWS)
**Namespace:** `practica-s12`  |  **Clúster:** `s12-cluster` (AWS, región `us-east-1`)
**Equipo / Integrantes:** `[COMPLETAR]`  |  **Fecha de la práctica:** `[COMPLETAR]`


---

## 0. Resumen ejecutivo

| # | Escenario | Síntoma observable | Causa raíz | Archivo corregido | Estado |
|---|-----------|--------------------|-----------|-------------------|--------|
| A | Configuración (ConfigMap/Secret) | Pods `CrashLoopBackOff`, `exit code 1` | La clave `DATABASE_HOST` en el ConfigMap no coincide con la variable `DB_HOST` que exige la app | `k8s/01-security-configmap.yaml` | ✅ Resuelto |
| C | Sondas de salud | Pods `0/1 Running` con `RESTARTS` en aumento; eventos `Liveness/Readiness probe failed: 404` | Las sondas apuntan a `/actuator/health` (ruta de Spring Boot) y esa ruta no existe en este backend | `k8s/03-app-deployment.yaml` | ✅ Resuelto |
| B | Red / conectividad | Pods `0/1 Running` (no Ready); `/ready` devuelve 503 `Connection refused` | `redis-service` enruta a `targetPort: 6380` pero Redis escucha en `6379` | `k8s/04-app-service.yaml` | ✅ Resuelto |


---

## 1. Metodología de diagnóstico

Se siguió siempre el mismo embudo, de lo general a lo específico:

```
1. kubectl get pods              ¿Qué estado tiene? (STATUS, READY, RESTARTS)
2. kubectl describe pod          ¿Qué dicen los EVENTOS de Kubernetes? (sondas, imagen, config)
3. kubectl logs [--previous]     ¿Qué dice la APLICACIÓN? (la salida del contenedor que murió)
4. kubectl get/describe svc,ep   ¿El Service apunta al Pod y al puerto correctos?
5. kubectl exec / port-forward   Probar desde adentro: ¿responde el endpoint? ¿conecta el puerto?
6. Corregir el MANIFIESTO (nunca "parches" manuales) → kubectl apply → volver al paso 1
```

Regla seguida: **la corrección se hace en el YAML versionado en Git**, no con `kubectl edit`,
para que la solución quede reproducible y documentada.

### 1.1 Cómo se reprodujo el estado "ANTES" (fallas inyectadas)

Se partió de los manifiestos del repositorio y se generó una copia rota en `/tmp/s12-broken/`
mediante tres `sed` (uno por escenario). Los manifiestos del repositorio **no se modifican**.

```bash
rm -rf /tmp/s12-broken && mkdir /tmp/s12-broken && cp k8s/*.yaml /tmp/s12-broken/

# Escenario A: clave del ConfigMap mal escrita (DB_HOST -> DATABASE_HOST)
sed -i 's/^  DB_HOST:/  DATABASE_HOST:/' /tmp/s12-broken/01-security-configmap.yaml

# Escenario B: targetPort del Service de la BD apunta a un puerto equivocado
sed -i 's/targetPort: redis/targetPort: 6380/' /tmp/s12-broken/04-app-service.yaml

# Escenario C: sondas con rutas copiadas de un proyecto Spring Boot
sed -i 's#path: /health$#path: /actuator/health#; s#path: /ready$#path: /actuator/health/readiness#' \
  /tmp/s12-broken/03-app-deployment.yaml

# Verificar los 4 cambios introducidos
diff -r k8s /tmp/s12-broken | grep '^[<>]'

# Desplegar el entorno ROTO
kubectl apply -f /tmp/s12-broken/
```

---

## 2. Estado inicial (ANTES) — base de la Evidencia 1

```bash
kubectl get pods -n practica-s12
```

**Salida esperada (ilustrativa):**

```text
NAME                          READY   STATUS             RESTARTS      AGE
app-backend-6c7d9f5b8-4xk2p   0/1     CrashLoopBackOff   3 (22s ago)   78s
app-backend-6c7d9f5b8-m9wqz   0/1     CrashLoopBackOff   3 (19s ago)   78s
redis-5f8b7c6d94-h7n2v        1/1     Running            0             78s
```

**Lectura:** Redis (la BD) está sano; las dos réplicas del backend fallan. El `READY 0/1` y
`CrashLoopBackOff` indican que el contenedor arranca, termina con error y Kubernetes lo reinicia
con espera exponencial (10 s, 20 s, 40 s… hasta 5 min).


---

## 3. Escenario A — `CrashLoopBackOff` por inyección de configuración

### 3.1 Síntoma
Ambos Pods del backend en `CrashLoopBackOff`, `RESTARTS` creciente.

### 3.2 Diagnóstico

**Paso 1 — Eventos del Pod**

```bash
POD=$(kubectl get pods -n practica-s12 -l app.kubernetes.io/name=app-backend -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod $POD -n practica-s12
```

Fragmentos relevantes de la salida:

```text
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       Error
      Exit Code:    1
    Ready:          False
    Restart Count:  4
...
Events:
  Type     Reason   Age                From     Message
  ----     ------   ----               ----     -------
  Normal   Pulled   40s (x4 over 2m)   kubelet  Container image "public.ecr.aws/docker/library/python:3.12-slim" already present on machine
  Normal   Started  40s (x4 over 2m)   kubelet  Started container backend
  Warning  BackOff  12s (x8 over 2m)   kubelet  Back-off restarting failed container backend in pod app-backend-6c7d9f5b8-4xk2p_practica-s12
```

**Qué se concluye de `describe`:** la imagen se descargó bien (no es un problema de registro) y
el contenedor **sí arranca** (`Started`), pero termina con **`Exit Code: 1`** → la propia
aplicación decide morir. Hay que leer *por qué* en sus logs.

**Paso 2 — Logs de la ejecución que murió**

```bash
kubectl logs $POD -n practica-s12            # intento actual
kubectl logs $POD -n practica-s12 --previous # la ejecución anterior (útil si ya reinició)
```

```text
2026-11-18T15:02:11 FATAL: variables de entorno requeridas no definidas: DB_HOST. Revise los nombres de las claves en el ConfigMap 'app-config' y en el Secret 'db-credentials'.
```

**Paso 3 — Comparar lo que la app espera vs. lo que el ConfigMap entrega**

```bash
kubectl get configmap app-config -n practica-s12 -o yaml
```

```yaml
data:
  APP_ENV: aws-lab
  APP_PORT: "8080"
  DATABASE_HOST: redis-service     # <-- ¡clave equivocada! la app exige DB_HOST
  DB_PORT: "6379"
  DB_TIMEOUT_SECONDS: "2"
```

### 3.3 Causa raíz
El Deployment inyecta el ConfigMap con `envFrom`, y **cada clave se vuelve una variable de
entorno con el mismo nombre**. Como la clave es `DATABASE_HOST` y no `DB_HOST`, la variable
`DB_HOST` no existe, la aplicación falla su validación de arranque y termina con código 1.

### 3.4 Solución (ANTES → DESPUÉS) — `k8s/01-security-configmap.yaml`

```diff
 data:
   APP_ENV: "aws-lab"
   APP_PORT: "8080"
-  DATABASE_HOST: "redis-service"
+  DB_HOST: "redis-service"
   DB_PORT: "6379"
```

```bash
kubectl apply -f k8s/01-security-configmap.yaml
kubectl rollout restart deployment/app-backend -n practica-s12
kubectl rollout status deployment/app-backend -n practica-s12 --timeout=90s
```

### 3.5 Verificación

```text
NAME                          READY   STATUS    RESTARTS   AGE
app-backend-7f9b6c8d5-bh2rt   0/1     Running   2          35s     <- ya NO es CrashLoopBackOff
```

Ahora el contenedor se mantiene vivo, pero sigue `0/1` y con reinicios: apareció la
**siguiente** falla (Escenario C), que estaba oculta tras esta.

### 3.6 Variantes del mismo escenario (qué se vería con otros errores)

| Error en el manifiesto | Estado resultante | Dónde se ve |
|------------------------|-------------------|-------------|
| Clave del ConfigMap/Secret mal escrita (caso de esta práctica) | `CrashLoopBackOff` (la app valida y muere) | `kubectl logs` |
| `secretRef`/`configMapRef` con el **nombre** del objeto equivocado | `CreateContainerConfigError` | `describe pod` → `Error: secret "db-credentials-x" not found` |
| `secretKeyRef` con una **clave** inexistente | `CreateContainerConfigError` | `describe pod` → `couldn't find key DB_PASS in Secret` |
| Secret con valor no-Base64 válido | El `apply` es rechazado | `illegal base64 data at input byte N` |
| Contraseña distinta a la que usa Redis | Pod `Running` pero `/ready` 503 (`-WRONGPASS`) | `kubectl logs` + `curl /ready` |

### 3.7 Prevención
Validar con `kubectl apply --dry-run=server`, mantener los nombres de variables documentados
en el README y fallar rápido (*fail-fast*) con un mensaje claro, como hace el backend.

---

## 4. Escenario C — Sondas de salud con endpoints incorrectos

### 4.1 Síntoma

```bash
kubectl get pods -n practica-s12 -w
```

```text
NAME                          READY   STATUS    RESTARTS      AGE
app-backend-7f9b6c8d5-bh2rt   0/1     Running   3 (8s ago)    2m10s
app-backend-7f9b6c8d5-w5kzn   0/1     Running   3 (11s ago)  2m10s
```

El Pod dice **`Running`** (el proceso está vivo) pero **`0/1`** (nunca llega a Ready) y
`RESTARTS` no deja de subir. Distinto de A: aquí el contenedor **no** termina por sí solo;
**lo mata el kubelet**.

### 4.2 Diagnóstico

**Paso 1 — Eventos**

```bash
kubectl describe pod $POD -n practica-s12 | sed -n '/Liveness:/,/Environment/p;/Events:/,$p'
```

```text
    Liveness:   http-get http://:http/actuator/health delay=5s timeout=2s period=10s #success=1 #failure=3
    Readiness:  http-get http://:http/actuator/health/readiness delay=5s timeout=3s period=5s #success=1 #failure=3
...
Events:
  Type     Reason     Age                  From     Message
  ----     ------     ----                 ----     -------
  Warning  Unhealthy  95s (x9 over 2m)     kubelet  Readiness probe failed: HTTP probe failed with statuscode: 404
  Warning  Unhealthy  65s (x3 over 2m)     kubelet  Liveness probe failed: HTTP probe failed with statuscode: 404
  Normal   Killing    65s                  kubelet  Container backend failed liveness probe, will be restarted
```

**Qué se concluye:** `statuscode: 404` → el servidor **sí responde**, pero esa **ruta no existe**.
`Killing ... failed liveness probe` explica los reinicios.

**Paso 2 — Logs (ver las peticiones del kubelet)**

```bash
kubectl logs $POD -n practica-s12
```

```text
2026-11-18T15:09:40 Iniciando backend | uid=10001 | puerto=8080 | BD=redis-service:6379 | usuario BD=default
2026-11-18T15:09:45 10.0.1.57 - "GET /actuator/health HTTP/1.1" 404 -
2026-11-18T15:09:45 10.0.1.57 - "GET /actuator/health/readiness HTTP/1.1" 404 -
```

(`10.0.1.57` es la IP del nodo: las sondas las ejecuta el kubelet.) Y de paso queda confirmado
`uid=10001`: la aplicación corre sin privilegios.

**Paso 3 — Comprobar qué rutas SÍ existen (aislar la causa)**

```bash
kubectl port-forward deployment/app-backend 8080:8080 -n practica-s12 &
curl -s -o /dev/null -w "/actuator/health -> %{http_code}\n" localhost:8080/actuator/health
curl -s -o /dev/null -w "/health          -> %{http_code}\n" localhost:8080/health
kill %1
```

```text
/actuator/health -> 404
/health          -> 200
```

### 4.3 Causa raíz
Los `path` de `livenessProbe` y `readinessProbe` fueron copiados de un proyecto Spring Boot
(`/actuator/health`). Este backend expone **`/health`** (liveness) y **`/ready`** (readiness).
Con 404, el kubelet cuenta 3 fallos seguidos (`failureThreshold: 3`) y **reinicia** el
contenedor; mientras tanto el Pod nunca pasa a Ready.

### 4.4 Solución (ANTES → DESPUÉS) — `k8s/03-app-deployment.yaml`

```diff
           livenessProbe:
             httpGet:
-              path: /actuator/health
+              path: /health
               port: http
...
           readinessProbe:
             httpGet:
-              path: /actuator/health/readiness
+              path: /ready
               port: http
```

```bash
kubectl apply -f k8s/03-app-deployment.yaml
kubectl rollout status deployment/app-backend -n practica-s12 --timeout=120s
```

> Nota de diseño: **liveness `/health` no consulta la base de datos** a propósito. Si lo hiciera,
> una caída de la BD reiniciaría *todos* los backends en cascada sin arreglar nada. La BD se
> valida en **readiness** (`/ready`), que solo retira al Pod del Service sin matarlo.

### 4.5 Verificación

```text
NAME                          READY   STATUS    RESTARTS   AGE
app-backend-8d4f6b7c9-2jvlp   0/1     Running   0          40s    <- RESTARTS dejó de crecer
app-backend-8d4f6b7c9-qm8tx   0/1     Running   0          40s    <- aún 0/1: falta la falla B
```

Los reinicios se detuvieron (liveness ya pasa), pero el Pod sigue `0/1`: la readiness
**ahora sí se ejecuta** y destapa la última falla.

### 4.6 Prevención
Probar cada ruta con `curl`/`port-forward` **antes** de ponerla en una sonda; no copiar sondas
entre stacks distintos; usar `startupProbe` para aplicaciones de arranque lento.

---

## 5. Escenario B — Falla de red/conectividad al puerto de la base de datos

### 5.1 Síntoma
Pods `Running`, **sin reinicios**, pero `0/1`. El Service público no tiene a quién enviar tráfico.

```bash
kubectl get pods,svc,endpoints -n practica-s12
```

```text
NAME                              READY   STATUS    RESTARTS   AGE
pod/app-backend-8d4f6b7c9-2jvlp   0/1     Running   0          3m
pod/app-backend-8d4f6b7c9-qm8tx   0/1     Running   0          3m
pod/redis-5f8b7c6d94-h7n2v        1/1     Running   0          9m

NAME                    TYPE           CLUSTER-IP      EXTERNAL-IP                            PORT(S)        AGE
service/app-service     LoadBalancer   172.20.45.112   a1b2c3...elb.us-east-1.amazonaws.com   80:31842/TCP   9m
service/redis-service   ClusterIP      172.20.88.23    <none>                                 6379/TCP       9m

NAME                      ENDPOINTS           AGE
endpoints/app-service     <none>              9m    <- ningún backend Ready
endpoints/redis-service   10.0.2.41:6380      9m    <- ¡puerto 6380!
```

### 5.2 Diagnóstico

**Paso 1 — Eventos y logs del backend**

```bash
kubectl describe pod $POD -n practica-s12 | sed -n '/Events:/,$p'
kubectl logs $POD -n practica-s12 | tail -5
```

```text
  Warning  Unhealthy  8s (x30 over 2m30s)  kubelet  Readiness probe failed: HTTP probe failed with statuscode: 503

2026-11-18T15:20:05 10.0.1.57 - "GET /ready HTTP/1.1" 503 -
```

`503` = el backend **está sano y responde**, pero declara que *no está listo* (su dependencia falla).

**Paso 2 — Preguntarle a la propia app por qué**

```bash
kubectl port-forward deployment/app-backend 8080:8080 -n practica-s12 &
curl -s localhost:8080/ready ; echo
kill %1
```

```json
{"status": "NOT_READY", "db": {"stage": "connect", "target": "redis-service:6379", "error": "Connection refused (el Service no apunta al puerto donde escucha la BD)"}}
```

`stage: connect` + `Connection refused` → el DNS **sí** resolvió `redis-service` (no es DNS) y la
conexión TCP llegó a alguien que **rechazó** el puerto (no es un bloqueo/timeout).

**Paso 3 — Inspeccionar el Service y comparar con el Pod**

```bash
kubectl describe svc redis-service -n practica-s12
```

```text
Port:              redis  6379/TCP
TargetPort:        6380/TCP          <-- el Service reenvía a 6380
Endpoints:         10.0.2.41:6380
```

```bash
kubectl get pod -l app.kubernetes.io/name=redis -n practica-s12 \
  -o jsonpath='{.items[0].spec.containers[0].ports[0].containerPort}{"\n"}'
```

```text
6379                                  <-- pero Redis escucha en 6379
```

**Paso 4 — Prueba de aislamiento (¿falla el Service o el Pod?)**

```bash
REDIS_IP=$(kubectl get pod -l app.kubernetes.io/name=redis -n practica-s12 -o jsonpath='{.items[0].status.podIP}')

# Via Service (puerto 6379): FALLA
kubectl exec -n practica-s12 deploy/app-backend -- python -c \
  "import socket; socket.create_connection(('redis-service',6379),2); print('OK via Service')"

# Directo al Pod (puerto 6379): FUNCIONA
kubectl exec -n practica-s12 deploy/app-backend -- python -c \
  "import socket; socket.create_connection(('$REDIS_IP',6379),2); print('OK directo al Pod')"
```

```text
ConnectionRefusedError: [Errno 111] Connection refused     <- via Service
OK directo al Pod                                          <- Redis está sano
```

Conclusión del aislamiento: **Redis y la red del clúster están bien; el defecto está en la
traducción del Service.** (En clústeres recientes `kubectl get endpoints` puede mostrar un aviso de
*deprecated*; el equivalente moderno es `kubectl get endpointslices -n practica-s12`.)

### 5.3 Causa raíz
`redis-service` declaraba `port: 6379` pero `targetPort: 6380`. El tráfico llegaba al Pod de Redis
por un puerto en el que **nadie escucha**, de ahí el `Connection refused`. Como la readiness del
backend depende de la BD, ningún backend pasó a Ready y `app-service` quedó sin endpoints (el
balanceador de AWS no tenía destinos sanos).

### 5.4 Solución (ANTES → DESPUÉS) — `k8s/04-app-service.yaml`

```diff
   ports:
     - name: redis
       protocol: TCP
       port: 6379
-      targetPort: 6380
+      targetPort: redis     # puerto NOMBRADO definido en el Deployment (containerPort: 6379)
```

```bash
kubectl apply -f k8s/04-app-service.yaml
kubectl get endpoints redis-service -n practica-s12
```

```text
NAME            ENDPOINTS        AGE
redis-service   10.0.2.41:6379   10m
```

Usar el **puerto nombrado** elimina esta clase de error: el Service sigue al nombre, no a un
número escrito a mano en dos sitios.

### 5.5 Verificación

```bash
kubectl get pods,endpoints -n practica-s12
kubectl logs deploy/app-backend -n practica-s12 --tail=3
```

```text
NAME                              READY   STATUS    RESTARTS   AGE
pod/app-backend-8d4f6b7c9-2jvlp   1/1     Running   0          6m
pod/app-backend-8d4f6b7c9-qm8tx   1/1     Running   0          6m
pod/redis-5f8b7c6d94-h7n2v        1/1     Running   0          12m

endpoints/app-service     10.0.1.18:8080,10.0.2.33:8080   12m
endpoints/redis-service   10.0.2.41:6379                  12m
```

### 5.6 Prevención
`port` y `targetPort` iguales o, mejor, puertos nombrados; revisar `kubectl get endpoints` tras
cada cambio de Service; mantener `DB_PORT` (ConfigMap) = `port` del Service.

---

## 6. Estado final (DESPUÉS) — base de la Evidencia 1

```bash
kubectl get pods -n practica-s12
```

```text
NAME                          READY   STATUS    RESTARTS   AGE
app-backend-8d4f6b7c9-2jvlp   1/1     Running   0          6m
app-backend-8d4f6b7c9-qm8tx   1/1     Running   0          6m
redis-5f8b7c6d94-h7n2v        1/1     Running   0          12m
```


### 6.1 Validación del hardening (Evidencia 2)

```bash
APP=$(kubectl get pods -n practica-s12 -l app.kubernetes.io/name=app-backend -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $APP -n practica-s12 -- id
kubectl exec -it $APP -n practica-s12 -- whoami
kubectl exec -it $APP -n practica-s12 -- touch /prueba.txt
```

```text
uid=10001 gid=10001 groups=10001
whoami: cannot find name for user ID 10001
touch: cannot touch '/prueba.txt': Read-only file system
```

- `id` demuestra **UID 10001** (≠ 0 → non-root).
- `whoami` imprime un error **esperado**: UID 10001 es un usuario "anónimo" sin entrada en
  `/etc/passwd`. Que no tenga nombre es parte del hardening; **la prueba válida es `id`**.
- `touch` falla con `Read-only file system` → `readOnlyRootFilesystem: true` funciona.

### 6.2 Prueba de servicio externo (Evidencia 4)

```bash
LB=$(kubectl get svc app-service -n practica-s12 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl -i http://$LB/
curl -s http://$LB/whoami
```

```text
HTTP/1.0 200 OK
Content-Type: application/json; charset=utf-8

{"service": "practica-s12-backend", "environment": "aws-lab", "pod": "app-backend-8d4f6b7c9-2jvlp", "uid": 10001, "database": {"connected": true, "target": "redis-service:6379", "authenticated_as": "default", "latency_ms": 1.2}}
{"pod": "app-backend-8d4f6b7c9-qm8tx", "uid": 10001, "gid": 10001, "root_fs_writable": false}
```

El campo `"connected": true` con `authenticated_as` demuestra que el **Secret** se inyectó
correctamente y que la BD autentica. Repetir el `curl` varias veces muestra distintos valores de
`"pod"`: el balanceador reparte entre las dos réplicas.

---

## 7. Script de diagnóstico reutilizable (`diagnose.sh`)

Pegue este script en su terminal (o guárdelo localmente como `diagnose.sh`; no es necesario
versionarlo). Reúne en una sola ejecución todo el embudo de la sección 1.

```bash
#!/usr/bin/env bash
# diagnose.sh - Diagnostico rapido del namespace practica-s12
# Uso: bash diagnose.sh [namespace]
NS="${1:-practica-s12}"
line() { printf '\n\033[1;36m=== %s ===\033[0m\n' "$1"; }

line "1. PODS (STATUS / READY / RESTARTS)"
kubectl get pods -n "$NS" -o wide

line "2. SERVICES y ENDPOINTS (un Service sin endpoints = nadie recibe trafico)"
kubectl get svc,endpoints -n "$NS"

line "3. EVENTOS recientes del namespace (los mas nuevos al final)"
kubectl get events -n "$NS" --sort-by=.lastTimestamp | tail -n 15

for POD in $(kubectl get pods -n "$NS" -o jsonpath='{.items[*].metadata.name}'); do
  line "4. DESCRIBE (estado y eventos) -> $POD"
  kubectl describe pod "$POD" -n "$NS" | grep -E "State:|Reason:|Exit Code:|Restart Count:|Liveness:|Readiness:|Warning|Unhealthy|BackOff|Killing|Failed" || true

  line "5. LOGS (ultimas 8 lineas) -> $POD"
  kubectl logs "$POD" -n "$NS" --tail=8 2>&1 || true

  if kubectl get pod "$POD" -n "$NS" -o jsonpath='{.status.containerStatuses[0].restartCount}' | grep -qv '^0$'; then
    line "5b. LOGS de la ejecucion ANTERIOR (--previous) -> $POD"
    kubectl logs "$POD" -n "$NS" --previous --tail=8 2>&1 || true
  fi
done

line "6. CONFIGURACION inyectada (claves, SIN valores secretos)"
echo "ConfigMap app-config:"; kubectl get configmap app-config -n "$NS" -o jsonpath='{.data}' ; echo
echo "Claves del Secret db-credentials:"; kubectl get secret db-credentials -n "$NS" -o jsonpath='{.data}' | sed -E 's/:"[^"]*"/:"***"/g'; echo

line "7. SEGURIDAD: usuario efectivo de cada backend"
for POD in $(kubectl get pods -n "$NS" -l app.kubernetes.io/name=app-backend -o jsonpath='{.items[*].metadata.name}'); do
  printf '%s -> ' "$POD"; kubectl exec "$POD" -n "$NS" -- id 2>&1 || true
done
```

---

## 8. Bonus de seguridad: el clúster rechaza un Pod root

El namespace tiene Pod Security Admission en modo `restricted`. Para demostrarlo:

```bash
kubectl run root-test --image=public.ecr.aws/docker/library/nginx:alpine -n practica-s12
```
