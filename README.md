# Proyecto Práctico S12 | AP9 — Despliegue Cloud, Seguridad y Troubleshooting

Despliegue de una arquitectura distribuida (backend + base de datos) en **Kubernetes sobre AWS**,
aplicando **hardening de seguridad** (securityContext, usuario non-root, Secrets) y
**diagnóstico y resolución de fallas** en un entorno cloud real.

| | |
|---|---|
| **Namespace** | `practica-s12` |
| **Nube** | Amazon Web Services (EKS recomendado; también válido Kubernetes sobre EC2) |
| **Exposición** | `LoadBalancer` (AWS NLB) — alternativa `NodePort` + Security Group |
| **Institución / Curso** | Instituto Valle Grande — Orquestación y Automatización de Software |


---

## 1. Introducción: hardening de Kubernetes en AWS

En AWS rige el **modelo de responsabilidad compartida**: AWS protege la infraestructura física y el
plano de control de EKS; **el cliente es responsable de lo que corre dentro**: la configuración de
los Pods, los Secrets, las reglas de red (Security Groups) y los permisos. Un clúster con el plano de
control perfecto sigue siendo vulnerable si un contenedor corre como `root` o si la base de datos
está expuesta a Internet.

El *hardening* aplica el **principio de mínimo privilegio** en capas (defensa en profundidad), de modo
que si una capa falla (p. ej. una vulnerabilidad en la aplicación), la siguiente limita el daño.

## 2. Arquitectura

```
                          Internet
                             │  HTTP :80
                   ┌─────────▼──────────┐
                   │ AWS Network Load   │   (Security Group / loadBalancerSourceRanges:
                   │ Balancer (ELB)     │    restringir al IP de origen)
                   └─────────┬──────────┘
 ┌───────────────────────────│───────────────────────────────────────────────┐
 │ Clúster Kubernetes (EKS)  │          Namespace: practica-s12              │
 │   (PSA "restricted")      │                                               │
 │            ┌──────────────▼───────────────┐                               │
 │            │ Service app-service          │  LoadBalancer  80 → http      │
 │            └───────┬──────────────┬───────┘                               │
 │            ┌───────▼──────┐ ┌─────▼────────┐                              │
 │            │ app-backend  │ │ app-backend  │  UID 10001, FS solo lectura  │
 │            │  (réplica 1) │ │  (réplica 2) │  :8080                       │
 │            └───────┬──────┘ └─────┬────────┘                              │
 │                    │  envFrom: ConfigMap app-config + Secret db-credentials│
 │            ┌───────▼──────────────▼───────┐                               │
 │            │ Service redis-service        │  ClusterIP (solo interno)     │
 │            └──────────────┬───────────────┘  6379 → redis                 │
 │                    ┌──────▼───────┐                                       │
 │                    │    redis     │  UID 10001, FS solo lectura, AUTH     │
 │                    └──────────────┘                                       │
 └───────────────────────────────────────────────────────────────────────────┘
```

| Recurso | Archivo | Función |
|---|---|---|
| Namespace `practica-s12` | `00-namespace.yaml` | Aislamiento + Pod Security Admission `restricted` |
| ConfigMap `app-config` | `01-security-configmap.yaml` | Configuración **pública** (`APP_PORT`, `DB_HOST`, `DB_PORT`…) |
| Secret `db-credentials` | `02-db-secret.yaml` | Credenciales (`DB_USER`, `DB_PASS`) en **Base64** |
| ConfigMap `backend-code` + Deployments `redis` y `app-backend` | `03-app-deployment.yaml` | Código del backend, base de datos y backend con hardening completo |
| Services `redis-service` y `app-service` | `04-app-service.yaml` | BD interna (ClusterIP) y exposición externa (LoadBalancer) |

**Sobre el backend:** es un servicio HTTP en Python (solo biblioteca estándar) montado desde un
ConfigMap sobre la imagen oficial `python:3.12-slim`, por lo que **no necesita construir ni subir
ninguna imagen** para desplegar. Endpoints: `/` (estado + BD), `/health` (liveness), `/ready`
(readiness, valida la BD con AUTH + PING), `/whoami` (UID y estado del sistema de archivos).

**Sobre la base de datos:** Redis 7 autenticado con la contraseña del Secret. Se eligió porque su
imagen oficial admite un UID arbitrario (10001) con sistema de archivos de solo lectura, lo que
permite aplicar el hardening completo también a la BD.

## 3. Arquitectura de seguridad (defensa en profundidad)

| Capa | Control | Dónde | Qué previene |
|---|---|---|---|
| Clúster | Pod Security Admission `restricted` | `00-namespace.yaml` | Que se cree un Pod root/privilegiado aunque alguien olvide el `securityContext` |
| Pod | `runAsNonRoot: true`, `runAsUser/Group: 10001`, `fsGroup: 10001`, `seccompProfile: RuntimeDefault` | `03` (nivel Pod) | Ejecutar como root; syscalls peligrosas |
| Contenedor | `runAsNonRoot`, `runAsUser: 10001`, `readOnlyRootFilesystem: true`, `allowPrivilegeEscalation: false`, `capabilities.drop: [ALL]` | `03` (nivel contenedor) | Persistencia de malware en disco, escalada a root, capabilities de Linux |
| Identidad | `automountServiceAccountToken: false` | `03` | Que un Pod comprometido use el token de la API de Kubernetes |
| Secretos | Secret Opaque (Base64) + inyección por `envFrom`/`secretKeyRef` | `02`, `03` | Contraseñas en texto plano en el ConfigMap o en la imagen |
| Recursos | `requests`/`limits` | `03` | Un Pod que agota CPU/memoria del nodo |
| Red | BD solo `ClusterIP`; `loadBalancerSourceRanges`; Security Group con IP `/32` | `04`, AWS | Exposición de la BD a Internet; acceso abierto a `0.0.0.0/0` |
| Salud | Liveness (sin dependencias) y Readiness (con BD) | `03` | Tráfico a Pods rotos; reinicios en cascada |


## 4. Estructura del repositorio

```
.
├── k8s/
│   ├── 00-namespace.yaml          # Namespace + Pod Security Admission
│   ├── 01-security-configmap.yaml # Configuración pública
│   ├── 02-db-secret.yaml          # Credenciales en Base64
│   ├── 03-app-deployment.yaml     # Código backend + Redis + backend (securityContext, sondas)
│   └── 04-app-service.yaml        # Services: ClusterIP (BD) + LoadBalancer (app)
├── docs/
│   └── troubleshooting-log.md     # Bitácora ANTES vs DESPUÉS (escenarios A, B, C)
└── README.md
```

## 5. Requisitos previos

| Herramienta | Verificación |
|---|---|
| Cuenta AWS con permisos para EKS/EC2/ELB/IAM | `aws sts get-caller-identity` |
| AWS CLI v2 | `aws --version` |
| `kubectl` | `kubectl version --client` |
| `eksctl` (para crear el clúster) | `eksctl version` |
| `git`, `curl` | — |

> **Entornos de laboratorio (p. ej. AWS Academy Learner Lab):** usan credenciales temporales
> (`aws configure` + `aws_session_token`) y pueden restringir la región (normalmente `us-east-1`)
> o la creación de EKS. Si EKS no está permitido, use Kubernetes sobre EC2 (k3s/kubeadm) y la
> variante **NodePort** (sección 7.2).

### 5.1 Crear el clúster EKS (omitir si ya tienen uno)

```bash
export AWS_REGION=us-east-1
eksctl create cluster \
  --name s12-cluster \
  --region $AWS_REGION \
  --nodegroup-name ng-s12 \
  --node-type t3.medium \
  --nodes 2 \
  --managed
# Demora ~15-20 minutos.

aws eks update-kubeconfig --name s12-cluster --region $AWS_REGION
kubectl get nodes -o wide        # 2 nodos en estado Ready
```

> 💸 **Costo:** el plano de control de EKS, las instancias EC2 y el balanceador se facturan por hora.
> Al terminar ejecute la **sección 11 (Limpieza)**.

## 6. Despliegue paso a paso

Se recomienda este orden porque produce **de forma natural las evidencias "Antes vs. Después"**.

### Paso 0 — Clonar el repositorio

```bash
git clone https://github.com/<USUARIO>/<REPOSITORIO>.git
cd <REPOSITORIO>
```

### Paso 1 — Validar los manifiestos (antes de aplicar)

```bash
kubectl apply --dry-run=client -f k8s/ -R >/dev/null && echo "YAML valido"
```

### Paso 2 — FASE "ANTES": desplegar con las 3 fallas inyectadas

Se crea una copia rota en `/tmp` (los archivos del repositorio **no se tocan**):

```bash
rm -rf /tmp/s12-broken && mkdir /tmp/s12-broken && cp k8s/*.yaml /tmp/s12-broken/

# A) ConfigMap: clave mal escrita
sed -i 's/^  DB_HOST:/  DATABASE_HOST:/' /tmp/s12-broken/01-security-configmap.yaml
# B) Service de la BD: targetPort equivocado
sed -i 's/targetPort: redis/targetPort: 6380/' /tmp/s12-broken/04-app-service.yaml
# C) Sondas con rutas de Spring Boot
sed -i 's#path: /health$#path: /actuator/health#; s#path: /ready$#path: /actuator/health/readiness#' \
  /tmp/s12-broken/03-app-deployment.yaml

diff -r k8s /tmp/s12-broken | grep '^[<>]'     # debe mostrar 6 líneas de diferencia

kubectl apply -f /tmp/s12-broken/00-namespace.yaml
kubectl apply -f /tmp/s12-broken/
kubectl get pods -n practica-s12 -w            # Ctrl+C tras ver CrashLoopBackOff
```
