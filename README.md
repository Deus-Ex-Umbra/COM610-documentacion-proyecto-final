<h1 align="center"> Despliegue de Sistema Clínico Dental Escalable en AWS con Arquitectura Hexagonal y Alta Disponibilidad </h1>

# 📋 Información del Proyecto

**Asignatura:** Trabajando en la Nube (COM610)  
**Docente:** Ing. Lucio Marcelo Quispe Ortega  
**Grupo:** 1  
**Estudiante:** Aparicio Llanquipacha Gabriel  
**Semestre:** 2-2025

# 🚀 Descripción del Software

Este proyecto consiste en el despliegue en la nube de una plataforma web para la gestión de una clínica dental. El sistema permite la administración de pacientes, citas y expedientes médicos mediante una interfaz moderna y segura.

A nivel de infraestructura, se ha diseñado una arquitectura Cloud-Native en Amazon Web Services (AWS) que prioriza la Alta Disponibilidad (HA), la Seguridad y la Escalabilidad Automática.

## 🛠 Tech Stack

| Componente | Tecnología | Servicio AWS |
|---|---|---|
| **Frontend** | React.js (Vite) | S3 + CloudFront |
| **Backend** | NestJS (Node.js) | EC2 + Auto Scaling Group |
| **Gestor Procesos**| PM2 (Process Manager) | EC2 (Ubuntu 24.04) |
| **Base de Datos** | PostgreSQL | Amazon RDS |
| **Proxy / CDN** | N/A | CloudFront + ALB |

# ☁️ Arquitectura del Despliegue

La arquitectura implementada sigue el patrón de Proxy Inverso con CDN, unificando tanto el frontend como el backend bajo un único dominio seguro (HTTPS) para evitar problemas de CORS y "Mixed Content".

<div align="center">
<img src="./imagenes/arquitectura.svg" alt="Diagrama de Arquitectura AWS" width="800"/>
</div>

## 🔄 Flujo de Datos

- **Cliente:** El usuario accede vía HTTPS a través de CloudFront.
- **CDN (CloudFront):**
  - Si la petición es contenido estático (JS, CSS), lo sirve desde el Bucket S3.
  - Si la ruta comienza con `/api`, redirige el tráfico al Application Load Balancer (ALB).
- **Manejo de Errores SPA:** CloudFront intercepta errores 403/404 del S3 y sirve `index.html` para permitir el enrutamiento del lado del cliente (React Router).
- **Balanceo:** El ALB distribuye la carga entre las instancias saludables en múltiples zonas de disponibilidad.
- **Cómputo (Auto Scaling):** Las instancias EC2 (Ubuntu) ejecutan el backend nativo gestionado por **PM2** para asegurar reinicios automáticos.
- **Persistencia:** Las instancias se conectan de forma privada a Amazon RDS.

---

## 🛡️ 1. Estrategia de Seguridad (Security Groups)

Se aplicó estrictamente el principio de **Mínimo Privilegio**. Los recursos no son accesibles directamente desde internet, salvo el punto de entrada.

| Grupo de Seguridad | Puerto | Origen Permitido | Descripción |
|---|---|---|---|
| **sg-alb-public** | 80 / 443 | 0.0.0.0/0 | Permite tráfico web público hacia el Balanceador. |
| **sg-backend-ec2** | 3000 | sg-alb-public | El backend solo acepta peticiones del ALB. |
| **sg-db-rds** | 5432 | sg-backend-ec2 | La base de datos solo acepta conexiones desde los servidores backend. |

---

## ⚙️ 2. Configuración de Automatización (Infrastructure as Code)

Para garantizar la elasticidad, no se configuran servidores manualmente. Se utiliza una **Launch Template** que aprovisiona automáticamente las instancias del Auto Scaling Group.

**Launch Template:**
- **AMI:** Ubuntu Server 24.04 LTS
- **Instance Type:** t2.micro / t3.micro
- **IAM Role:** Rol con permisos mínimos necesarios (S3 Read).

### 📜 Script de User Data (Bootstrapping)

Este script se ejecuta automáticamente al nacer cada instancia. Se encarga de **instalar Node.js**, **clonar el repositorio**, **inyectar las variables de entorno** y **desplegar con PM2**.

```bash
#!/bin/bash
exec > >(tee /var/log/user-data.log|logger -t user-data -s 2>/dev/console) 2>&1
echo "--- INICIO DEL DESPLIEGUE ---"
apt-get update -y
apt-get install -y git curl build-essential
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt-get install -y nodejs
npm install -g pm2
mkdir -p /home/ubuntu/app
cd /home/ubuntu/app
echo "--- CLONANDO REPOSITORIO ---"
git clone https://github.com/Deus-Ex-Umbra/COM610-0d3nt-backend.git .

echo "--- CREANDO .ENV ---"
cat <<EOF > .env
DB_TYPE=
DB_HOST=
DB_PORT=
DB_USERNAME=
DB_PASSWORD=
DB_DATABASE=
PORT=
JWT_SECRET=
JWT_EXPIRATION_TIME=3
GEMINI_API_KEY=
ARCHIVOS_PATH=
AWS_REGION=
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_S3_BUCKET_NAME=
AWS_S3_CDN=
EOF

echo "--- INSTALANDO DEPENDENCIAS ---"
npm install
echo "--- CONSTRUYENDO PROYECTO (BUILD) ---"
npm run build
chown -R ubuntu:ubuntu /home/ubuntu/app
echo "--- INICIANDO PM2 ---"
su - ubuntu -c "cd /home/ubuntu/app && pm2 start dist/main.js --name 'backend-api'"
su - ubuntu -c "pm2 save"
su - ubuntu -c "pm2 startup"
echo "--- FIN DEL DESPLIEGUE ---"
```

<div align="center">
<img src="./imagenes/plantilla_lanzamiento.png" alt="Configuración User Data" width="700"/>
</div>

## 📈 3. Escalabilidad y Auto Scaling Group (ASG)

El sistema está preparado para reaccionar automáticamente ante picos de demanda, garantizando alta disponibilidad.

### ⚙️ Configuración del ASG

- **Mínimo:** 2 instancias
- **Máximo:** 4 instancias
- **Política de Escalado (Target Tracking):**
  - Se configuró un monitoreo de **CPU Promedio** mediante CloudWatch.
  - Si el CPU supera el **40%**, el sistema añade automáticamente nuevas instancias.

---

### 🧪 Prueba de Estrés (Resiliencia)

Para validar la elasticidad del entorno, se ejecutó una prueba de carga utilizando la herramienta `stress`.

**Comando utilizado:**

```bash
stress --cpu 2 --timeout 600
```

Resultado: El ASG detectó la carga y lanzó nuevas instancias automáticamente sin intervención humana.

## 🌐 4. CloudFront como Proxy Inverso (Solución CORS / SSL)

Uno de los mayores desafíos fue la integración segura entre el Frontend (HTTPS) y el Backend. La solución fue configurar **CloudFront como punto de entrada único**, actuando como *reverse proxy*.

### 🔧 Configuración de Behaviors

- **Ruta Default (`*`)**: Sirve el Frontend (React) desde el Bucket S3.
- **Ruta API (`/api/*`)**: Redirige el tráfico al Application Load Balancer.
- **Política de Solicitud:** `AllViewerExceptHostHeader`. Esto fue crítico para que el ALB aceptara las peticiones sin rechazar el encabezado Host del CDN.

Esto permite que el navegador interprete **todo el tráfico como local y seguro**, eliminando problemas de CORS y asegurando compatibilidad SSL end-to-end.

### 🏗️ Configuración en Origins

- **Origin S3:** Acceso restringido mediante **OAC (Origin Access Control)**.
- **Origin ALB:** Conexión interna vía **HTTP (Puerto 80)** dentro de AWS (no expuesto a internet).

<div align="center">
<img src="./imagenes/cloudfront_configuracion.png" alt="Configuración CloudFront Behaviors" width="700"/>
</div>

---

## 💻 5. Despliegue del Frontend

El frontend se construyó localmente y se desplegó como sitio estático en S3.

### 📦 Archivo `.env` de React

Gracias a la configuración del proxy, la API utiliza el **mismo dominio del CDN**, evitando rutas absolutas inseguras.

```bash
VITE_API_URL=[https://d3ftme9hh1yrd9.cloudfront.net](https://d3ftme9hh1yrd9.cloudfront.net)

```

---

## ✅ Conclusiones

Este despliegue demuestra cómo una arquitectura moderna en AWS resuelve problemas tradicionales de infraestructura:

- **Costo-Efectividad:** Uso de instancias ligeras gestionadas por PM2 en lugar de Docker para maximizar recursos en capa gratuita.
- **Seguridad:** El backend y la base de datos están completamente aislados de internet.
- **Experiencia de Usuario:** CloudFront distribuye el contenido estático y dinámico, resolviendo latencia y certificados SSL en un solo punto.