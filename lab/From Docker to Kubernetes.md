# From Docker to Kubernetes

## Overview

Docker Compose is a tool for running multi-container applications on a single machine. Kubernetes solves the same problem at scale, using multiple machines, with self-healing, rolling updates, and production-grade networking.

---

## The Docker Compose application

The starting point is a Spring Boot application backed by a PostgreSQL database, defined with Docker Compose:

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
      test: [ "CMD-SHELL", "pg_isready -U user -d jdbc_schema" ]
      interval: 30s
      timeout: 10s
      retries: 5

networks:
  my_network:
    driver: bridge

volumes:
  pg-data:
```

With Docker Compose, this entire stack is started with a single command:

```bash
docker compose up
```

The goal is to run the same stack on Kubernetes, using native Kubernetes objects.

---

## Concept mapping

This is how Docker Compose concepts map to Kubernetes objects:

| Docker Compose | Kubernetes | Notes |
|---|---|---|
| `service` | `Deployment` + `Service` | A Compose service becomes two K8s objects: one manages the Pods, one exposes them on the network |
| `image` | `spec.containers[].image` | Same image name and tag |
| `ports` | `Service.spec.ports` | Port exposure is handled by a Service, not the container directly |
| `environment` | `env` or `ConfigMap` / `Secret` | Plain values → `env`; sensitive values → `Secret` |
| `networks` | Kubernetes DNS | In K8s, every Service is reachable by its name within the cluster — no explicit network definition needed |
| `depends_on` | `readinessProbe` | K8s does not have `depends_on`; instead, each container declares when it is ready to receive traffic |
| `volumes` (named) | `PersistentVolumeClaim` | A PVC requests durable storage from the cluster |
| `healthcheck` | `livenessProbe` + `readinessProbe` | K8s has two separate health check mechanisms |

---

### Secret — database credentials

In Docker Compose, credentials are passed as plain environment variables. In Kubernetes, sensitive values should be stored in a `Secret`.

```yaml
# postgres-secret.yaml
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

> `stringData` accepts plain text — Kubernetes automatically base64-encodes the values before storing them.

### PersistentVolumeClaim — data volume

The `pg-data` named volume in Compose maps to a `PersistentVolumeClaim` in Kubernetes.

```yaml
# postgres-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce      # one node can mount this volume at a time
  resources:
    requests:
      storage: 1Gi
```

### Deployment — PostgreSQL container

```yaml
# postgres-deployment.yaml
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

          # Load credentials from the Secret
          envFrom:
            - secretRef:
                name: postgres-secret

          ports:
            - containerPort: 5432

          # Mount the persistent volume at the PostgreSQL data directory
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data

          # Equivalent to docker-compose healthcheck
          readinessProbe:
            exec:
              command:
                - pg_isready
                - -U
                - user
                - -d
                - jdbc_schema
            initialDelaySeconds: 10
            periodSeconds: 30
            timeoutSeconds: 10
            failureThreshold: 5

      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: postgres-pvc    # bind to the PVC defined above
```

### Service — expose PostgreSQL inside the cluster

In Docker Compose, services on the same network reach each other by service name. In Kubernetes, this is handled by a `Service` — the Spring Boot app will connect to PostgreSQL at `postgres:5432`.

```yaml
# postgres-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres          # this name is the DNS hostname other Pods will use
spec:
  selector:
    app: postgres         # routes traffic to Pods with this label
  ports:
    - port: 5432
      targetPort: 5432
  type: ClusterIP         # internal only — PostgreSQL should not be exposed externally
```

---

## Migrating the Spring Boot application

### ConfigMap — application configuration

The `SPRING_PROFILES_ACTIVE=docker` environment variable tells Spring Boot which configuration profile to use. In Kubernetes, non-sensitive configuration lives in a `ConfigMap`.

```yaml
# echo-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: echo-config
data:
  SPRING_PROFILES_ACTIVE: "kubernetes"   # use a dedicated K8s profile
```

> It is good practice to create a dedicated `kubernetes` Spring profile (in addition to the existing `docker` profile) that points to the Kubernetes Service DNS name (`postgres:5432`) for the database URL.

### Deployment — Spring Boot container

```yaml
# echo-deployment.yaml
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
      containers:
        - name: echo
          image: echo-server-logs-db-java:latest
          imagePullPolicy: IfNotPresent   # use local image if available

          # Load config from ConfigMap
          envFrom:
            - configMapRef:
                name: echo-config

          # Also inject DB credentials from the Secret
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

          # Kubernetes equivalent of depends_on:
          # the Pod will not receive traffic until this probe passes
          readinessProbe:
            httpGet:
              path: /actuator/health
              port: 5000
            initialDelaySeconds: 20
            periodSeconds: 10
            failureThreshold: 5

          # Restart the container if it becomes unresponsive
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 5000
            initialDelaySeconds: 30
            periodSeconds: 15
```

### Service — expose the application

```yaml
# echo-service.yaml
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
  type: NodePort      # exposes the service on a port of each node
                      # use LoadBalancer on cloud providers
```

---

## Deploying the full stack

With all manifests in place, the full stack is deployed with:

```bash
# Apply all manifests in order
kubectl apply -f postgres-secret.yaml
kubectl apply -f postgres-pvc.yaml
kubectl apply -f postgres-deployment.yaml
kubectl apply -f postgres-service.yaml
kubectl apply -f echo-configmap.yaml
kubectl apply -f echo-deployment.yaml
kubectl apply -f echo-service.yaml

# Or apply everything in the directory at once
kubectl apply -f .
```

Verify the stack is running:

```bash
# Check all Pods are Running
kubectl get pods
# NAME                        READY   STATUS    AGE
# postgres-6d8f9b8c4-abc12    1/1     Running   2m
# echo-7f9d6c5b3-xyz99        1/1     Running   1m

# Check Services
kubectl get services
# NAME       TYPE        CLUSTER-IP      PORT(S)
# postgres   ClusterIP   10.96.0.10      5432/TCP
# echo       NodePort    10.96.0.11      5000:31234/TCP

# Access the application
curl http://<node-ip>:31234
```

---

## Scaling

One of the key advantages of Kubernetes over Docker Compose is horizontal scaling. With Compose, scaling requires manual configuration. With Kubernetes, it is a single command:

```bash
# Scale the echo service to 3 replicas
kubectl scale deployment echo --replicas=3

# Verify
kubectl get pods
# NAME                        READY   STATUS    AGE
# echo-7f9d6c5b3-xyz99        1/1     Running   5m
# echo-7f9d6c5b3-abc12        1/1     Running   10s
# echo-7f9d6c5b3-def34        1/1     Running   10s
```

Kubernetes automatically distributes traffic across all three replicas via the `echo` Service.

> Note: PostgreSQL should not be scaled this way — a StatefulSet with a shared PVC or a managed database service should be used instead for production database deployments.

---

## Rolling updates

Updating the application image in Docker Compose requires stopping and restarting the container, causing downtime. Kubernetes performs rolling updates with zero downtime:

```bash
# Update the application image
kubectl set image deployment/echo echo=echo-server-logs-db-java:v2

# Watch the rollout
kubectl rollout status deployment/echo
# Waiting for deployment "echo" rollout to finish: 1 out of 3 new replicas have been updated...
# Waiting for deployment "echo" rollout to finish: 2 out of 3 new replicas have been updated...
# deployment "echo" successfully rolled out

# Roll back if something goes wrong
kubectl rollout undo deployment/echo
```

---

## Side-by-side comparison

| Concern | Docker Compose | Kubernetes |
|---|---|---|
| Start the stack | `docker compose up` | `kubectl apply -f .` |
| Stop the stack | `docker compose down` | `kubectl delete -f .` |
| Scale a service | `docker compose up --scale echo=3` | `kubectl scale deployment echo --replicas=3` |
| Update an image | `docker compose pull && docker compose up` | `kubectl set image deployment/echo echo=<new-image>` |
| View logs | `docker compose logs echo` | `kubectl logs -l app=echo` |
| Exec into container | `docker compose exec echo bash` | `kubectl exec -it <pod-name> -- bash` |
| Persistent storage | Named volume | PersistentVolumeClaim |
| Inter-service networking | Service name on shared network | Service DNS name |
| Health checks | `healthcheck` block | `readinessProbe` + `livenessProbe` |
| Startup ordering | `depends_on` | `readinessProbe` (traffic withheld until ready) |
| Sensitive config | Plain env vars | `Secret` |
| Non-sensitive config | Plain env vars | `ConfigMap` |

---

## Summary

Migrating from Docker Compose to Kubernetes requires more files and more explicit configuration, but each piece of that configuration unlocks capabilities that Docker Compose cannot provide: self-healing, horizontal scaling, rolling updates, and production-grade secret management.

The core migration pattern is always the same:

1. Each Compose `service` becomes a `Deployment` + `Service` pair;
2. Named volumes become `PersistentVolumeClaims`;
3. Plain environment variables become `ConfigMaps` (non-sensitive) or `Secrets` (sensitive);
4. `depends_on` becomes a `readinessProbe` on the dependent container;
5. Shared networks disappear because Kubernetes DNS handles service discovery automatically;
