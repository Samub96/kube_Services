# 📘 Proyecto: Arquitectura de Frontend, Backend y Base de Datos con Nginx en Kubernetes

## 🧩 Descripción general

Este proyecto implementa una arquitectura básica de **tres capas** (frontend, backend y base de datos) sobre un **cluster Kubernetes**, aplicando políticas de red (*NetworkPolicies*) para aislar el tráfico entre componentes según buenas prácticas de seguridad.

La solución se desplegó en un entorno **Minikube dentro de GitHub Codespaces**, utilizando `kubectl` y archivos YAML organizados por capas.

---

## 🧱 Estructura de la solución

| Capa | Descripción | Imagen base | Exposición |
|------|--------------|--------------|-------------|
| **Frontend** | Servidor Nginx que actúa como interfaz de usuario. | `nginx:1.25` | `NodePort (30080)` |
| **Backend** | Servidor Nginx (simulación del API). | `nginx:1.25` | `ClusterIP` |
| **Base de datos** | Servidor PostgreSQL con variables de entorno configuradas. | `postgres:15` | `ClusterIP` |

Cada servicio se despliega como **Deployment** y **Service** dentro del namespace `app`.

---

## ⚙️ Archivos de configuración

### 1️⃣ `00-namespace.yaml`
Crea el namespace de trabajo:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app
```

### 2️⃣ `10-deployments-services.yaml`
Define los **Deployments** y **Services** para:
- `frontend` → Nginx expuesto por NodePort `30080`.
- `backend` → Nginx interno (ClusterIP).
- `db` → PostgreSQL interno (ClusterIP).

Incluye variables de entorno seguras para el contenedor de base de datos.

### 3️⃣ `20-networkpolicies.yaml`
Implementa el control de tráfico entre pods según las reglas definidas en la imagen de referencia:

| Política | Descripción |
|-----------|--------------|
| **frontend-egress-to-backend** | El frontend **solo puede comunicarse con el backend** (TCP/80). |
| **backend-ingress-from-frontend** | El backend **solo acepta tráfico desde el frontend** y **solo puede conectarse al DB** (TCP/5432). |
| **db-ingress-from-backend-no-egress** | La base de datos **solo acepta tráfico desde el backend** y **no tiene salida**. |

### 4️⃣ (Opcional) `30-ingress-frontend.yaml`
Permite exponer el frontend mediante un Ingress (si se desea habilitar un dominio interno como `myapp.local`).

---

## 🚀 Pasos de despliegue

### 1. Iniciar el cluster
Si usas Minikube:
```bash
minikube start
```

### 2. Aplicar los manifiestos
```bash
kubectl apply -f 00-namespace.yaml
kubectl apply -f 10-deployments-services.yaml
kubectl apply -f 20-networkpolicies.yaml
```

### 3. Verificar el estado
```bash
kubectl -n app get pods -o wide
kubectl -n app get svc
kubectl -n app get netpol
```

### 4. Acceder al frontend desde Codespaces
Crea un túnel al servicio de frontend:
```bash
kubectl -n app port-forward svc/frontend-svc 8080:80
```

Luego, en GitHub Codespaces:
- Abre la pestaña **Ports**.
- Cambia el puerto `8080` a **Public**.
- Accede a la URL generada, por ejemplo:  
  ```
  https://8080-<usuario>-<repo>.app.github.dev
  ```

Deberías ver la página:  
> **Welcome to nginx!**

---

## 🧪 Pruebas de conectividad

Para validar las políticas de red:

```bash
# Probar acceso desde el frontend al backend
kubectl run test --rm -it --image=curlimages/curl -n app --   curl -v http://backend-svc

# Intentar acceder al DB desde el frontend (debe fallar)
kubectl run test --rm -it --image=curlimages/curl -n app --   curl -v http://db-svc:5432
```

El segundo comando debe fallar (bloqueo correcto de NetworkPolicy).

---

## 🔒 Políticas de Seguridad

El esquema de comunicación final queda así:

```
[Internet]
     ↓
 [Frontend - Nginx]
     ↓ (TCP/80)
 [Backend - Nginx]
     ↓ (TCP/5432)
 [Database - PostgreSQL]
```

- Todo tráfico lateral entre pods no permitidos está bloqueado.
- La base de datos no puede iniciar conexiones externas.
- Solo el frontend tiene salida hacia el backend.

---

## 🧭 Comandos útiles

| Acción | Comando |
|---------|----------|
| Ver pods del namespace | `kubectl -n app get pods -o wide` |
| Ver servicios | `kubectl -n app get svc` |
| Ver políticas de red | `kubectl -n app get netpol` |
| Desplegar todos los manifiestos | `kubectl apply -f .` |
| Eliminar todo | `kubectl delete ns app` |

---

## 🧠 Recomendaciones

- Si usas Codespaces, **mantén el `port-forward` activo** mientras pruebas el frontend.  
- Si trabajas localmente con Minikube, también puedes usar:
  ```bash
  minikube service frontend-svc -n app
  ```
  para abrir el servicio en el navegador.  
- Para contenido personalizado del frontend, se puede montar un `ConfigMap` con tu propio `index.html`.

---

## 📌 Estado del despliegue final
✅ Cluster funcionando (Minikube en Codespaces)  
✅ Frontend/Backend/DB activos  
✅ NetworkPolicies aplicadas y verificadas  
✅ Frontend accesible vía `port-forward`
