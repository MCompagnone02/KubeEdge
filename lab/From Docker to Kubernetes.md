# From Docker to Kubernetes

## Overview

Docker Compose is a tool for running multi-container applications on a **single machine**. Kubernetes solves the same problem at scale: multiple machines, self-healing, rolling updates, and production-grade networking.

---

## General Migration

Every Docker Compose → Kubernetes migration follows the same five steps, regardless of the application:

| Step | What you do |
|------|-------------|
| **1. Secrets** | Move sensitive env vars (`passwords`, `tokens`) into a `Secret` |
| **2. ConfigMaps** | Move non-sensitive env vars (`profiles`, `URLs`) into a `ConfigMap` |
| **3. Storage** | Replace named volumes with `PersistentVolumeClaims` |
| **4. Workloads** | Each Compose `service` becomes a `Deployment` + `Service` pair |
| **5. Startup order** | Replace `depends_on` with `initContainers` on the dependent pod |

---

## Concept mapping

| Docker Compose | Kubernetes | Notes |
|----------------|------------|-------|
| `service` | `Deployment` + `Service` | One Compose service becomes two K8s objects: one manages Pods, one exposes them |
| `image` | `spec.containers[].image` | Same image name and tag |
| `ports` | `Service.spec.ports` | Port exposure is handled by the Service, not the container directly |
| `environment` (non-sensitive) | `ConfigMap` | Injected via `envFrom` |
| `environment` (sensitive) | `Secret` | Injected via `envFrom` or individual `secretKeyRef` |
| `networks` | Kubernetes DNS | Every Service is reachable by its name within the cluster — no explicit network needed |
| `depends_on` | `initContainer` | An init container runs to completion before the main container starts |
| `healthcheck` | `readinessProbe` + `livenessProbe` | K8s has two separate mechanisms: readiness (ready for traffic?) and liveness (still alive?) |
| `volumes` (named) | `PersistentVolumeClaim` | A PVC requests durable storage from the cluster |

---

## The Docker Compose application

The starting point is a Spring Boot application backed by PostgreSQL, defined with Docker Compose:

```yaml
# docker-compose.yml
services:
  echo:
    build: echo-server-logs-db-java
    ports:
      - "5000:5000"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
    networks:
      - my_network
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: jdbc_schema
    volumes:
      - pg-data:/var/lib/postgresql/data
    networks:
      - my_network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d jdbc_schema"]
      interval: 30s
      timeout: 10s
      retries: 5

networks:
  my_network:
    driver: bridge

volumes:
  pg-data:
```

```bash
docker compose up   # the entire stack in one command
```

---

## Step 1 — Secrets and ConfigMaps

### Secret for database credentials

In Docker Compose, credentials are plain environment variables. In Kubernetes, sensitive values live in a `Secret`.

```yaml
# k8s/postgres/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
type: Opaque
stringData:
  POSTGRES_USER: user
  POSTGRES_PASSWORD: secret
  POSTGRES_DB: jdbc_schema
```

### ConfigMap for application configuration

Non-sensitive configuration (Spring profile, feature flags, URLs) lives in a `ConfigMap`.

```yaml
# k8s/echo/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: echo-config
data:
  SPRING_PROFILES_ACTIVE: "kubernetes"
```

---

## Step 2 — Persistent storage for PostgreSQL

### PersistentVolumeClaim

The `pg-data` named volume in Compose maps to a `PersistentVolumeClaim`.

```yaml
# k8s/postgres/pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce       # one node can mount this volume at a time
  resources:
    requests:
      storage: 1Gi
```

---

## Step 3 — Deploying PostgreSQL

### Deployment

```yaml
# k8s/postgres/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:17-alpine

          # Step 1: load credentials from the Secret
          envFrom:
            - secretRef:
                name: postgres-secret

          ports:
            - containerPort: 5432

          # Step 3: mount the persistent volume
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data

          # Readiness probe: Pod receives traffic only when DB is accepting connections
          readinessProbe:
            exec:
              command: ["pg_isready", "-U", "user", "-d", "jdbc_schema"]
            initialDelaySeconds: 10
            periodSeconds: 30
            timeoutSeconds: 10
            failureThreshold: 5

          # Liveness probe: restart the container if it becomes permanently unresponsive
          livenessProbe:
            exec:
              command: ["pg_isready", "-U", "user", "-d", "jdbc_schema"]
            initialDelaySeconds: 30
            periodSeconds: 30
            timeoutSeconds: 10
            failureThreshold: 3

      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: postgres-pvc
```

### Service

In Kubernetes, inter-service networking is handled by DNS. The Spring Boot app connects to PostgreSQL at `postgres:5432`.

```yaml
# k8s/postgres/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres         # this becomes the DNS hostname other Pods use
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
  type: ClusterIP        # internal only — never expose a database externally
```

**Deployment vs StatefulSet for databases**: this example uses a `Deployment` for simplicity (one replica, single PVC). For production PostgreSQL — especially with replication — use a `StatefulSet`, which assigns a stable network identity and a dedicated PVC to each replica. For a single-instance dev database, `Deployment` + PVC is acceptable.

---

## Step 4 — Deploying the Spring Boot application

### Deployment with initContainer

The `depends_on: condition: service_healthy` in Compose is replaced by an `initContainer` that polls PostgreSQL until it is ready. Only after the initContainer exits successfully does Kubernetes start the main application container.

```yaml
# k8s/echo/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:

      # Step 5: wait for PostgreSQL before starting the app
      # This is the correct Kubernetes equivalent of depends_on: service_healthy
      initContainers:
        - name: wait-for-postgres
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              until nc -z postgres 5432; do
                echo "Waiting for postgres..."; sleep 2
              done
              echo "PostgreSQL is ready."

      containers:
        - name: echo
          image: echo-server-logs-db-java:latest
          imagePullPolicy: IfNotPresent

          # Step 2: load non-sensitive config from ConfigMap
          envFrom:
            - configMapRef:
                name: echo-config

          # Step 1: inject DB credentials from the Secret individually
          env:
            - name: SPRING_DATASOURCE_USERNAME
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_USER
            - name: SPRING_DATASOURCE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_PASSWORD

          ports:
            - containerPort: 5000

          # Readiness: Pod does not receive traffic until Spring Boot is fully started
          readinessProbe:
            httpGet:
              path: /actuator/health
              port: 5000
            initialDelaySeconds: 20
            periodSeconds: 10
            failureThreshold: 5

          # Liveness: restart the container if the health endpoint stops responding
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 5000
            initialDelaySeconds: 30
            periodSeconds: 15
            failureThreshold: 3
```

### Service

```yaml
# k8s/echo/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: echo
spec:
  selector:
    app: echo
  ports:
    - port: 5000
      targetPort: 5000
  type: NodePort        # exposes on a port of each node
                       
```

---

## Step 5 - Deploying the full stack

```bash
# Apply in dependency order: storage and DB first, then the app
kubectl apply -f k8s/postgres/
kubectl apply -f k8s/echo/

# Or with Kustomize (applies everything in the correct order)
kubectl apply -k k8s/
```

Verify the stack:

```bash
# All Pods should reach Running (the echo Pod starts after the initContainer exits)
kubectl get pods -w
# NAME                        READY   STATUS       AGE
# postgres-6d8f9b8c4-abc12    0/1     Running      5s
# postgres-6d8f9b8c4-abc12    1/1     Running      15s   ← readinessProbe passed
# echo-7f9d6c5b3-xyz99        0/1     Init:0/1     0s    ← initContainer running
# echo-7f9d6c5b3-xyz99        0/1     PodInitializing  8s
# echo-7f9d6c5b3-xyz99        1/1     Running      30s   ← app started

# Services
kubectl get services
# NAME       TYPE        CLUSTER-IP     PORT(S)
# postgres   ClusterIP   10.96.0.10     5432/TCP
# echo       NodePort    10.96.0.11     5000:31234/TCP
```

### Accessing the application

```bash
# Option A: via NodePort (requires knowing the node IP)
curl http://<node-ip>:31234

# Option B: cport-forward (simpler for local development, no NodePort needed)
kubectl port-forward service/echo 5000:5000
curl http://localhost:5000
```

`kubectl port-forward` is the easiest way to access a service locally during development. It creates a direct tunnel between your machine and the Pod — no need to expose a NodePort or configure an Ingress.

---

## Scaling

One of the key advantages of Kubernetes over Docker Compose is **horizontal scaling** with a single command:

```bash
# Scale the echo service to 3 replicas
kubectl scale deployment echo --replicas=3

# Kubernetes automatically distributes traffic across all replicas via the Service
kubectl get pods
# NAME                    READY   STATUS    AGE
# echo-7f9d6c5b3-xyz99   1/1     Running   5m
# echo-7f9d6c5b3-abc12   1/1     Running   10s
# echo-7f9d6c5b3-def34   1/1     Running   10s
```

---

## Rolling updates

Kubernetes performs zero-downtime rolling updates automatically:

```bash
# Deploy a new image version
kubectl set image deployment/echo echo=echo-server-logs-db-java:v2

# Watch the rollout: new Pods come up before old ones are terminated
kubectl rollout status deployment/echo
# Waiting for deployment "echo" rollout to finish: 1 out of 3 new replicas have been updated...
# Waiting for deployment "echo" rollout to finish: 2 out of 3 new replicas have been updated...
# deployment "echo" successfully rolled out

# If something went wrong, the roll back is instant with this command
kubectl rollout undo deployment/echo
```

---

## Command reference: Docker Compose vs Kubernetes

| Operation | Docker Compose | Kubernetes |
|-----------|---------------|------------|
| Start the stack | `docker compose up` | `kubectl apply -f k8s/` |
| Stop the stack | `docker compose down` | `kubectl delete -f k8s/` |
| Scale a service | `docker compose up --scale echo=3` | `kubectl scale deployment echo --replicas=3` |
| Update an image | `docker compose pull && docker compose up` | `kubectl set image deployment/echo echo=<new-image>` |
| View logs | `docker compose logs echo` | `kubectl logs -l app=echo -f` |
| Exec into container | `docker compose exec echo bash` | `kubectl exec -it <pod-name> -- bash` |
| Access a service locally | `localhost:<port>` | `kubectl port-forward service/echo 5000:5000` |
| Persistent storage | Named volume | `PersistentVolumeClaim` |
| Inter-service networking | Service name on shared network | Service DNS name (automatic) |
| Health checks | `healthcheck` block | `readinessProbe` + `livenessProbe` |
| Startup ordering | `depends_on` | `initContainer` |
| Sensitive config | Plain env vars | `Secret` |
| Non-sensitive config | Plain env vars | `ConfigMap` |

---

## Summary

Migrating from Docker Compose to Kubernetes requires more files and more explicit configuration, but each piece unlocks capabilities that Docker Compose cannot provide: self-healing, horizontal scaling, rolling updates, and production-grade secret management.

The core recipe is always the same:

1. Sensitive environment variables → **Secret**;
2. Non-sensitive environment variables → **ConfigMap**;
3. Named volumes → **PersistentVolumeClaim**;
4. Each Compose service → **Deployment** + **Service**;
5. `depends_on` → **initContainer** on the dependent pod;

---

*Next: [KubeEdge](./KubeEdge.md)*
