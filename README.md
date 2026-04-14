# 🚀 Práctica: Flask + RDS MySQL + AWS Lambda en EC2

Guía paso a paso para desplegar una aplicación Flask en Amazon EC2 conectada a una base de datos RDS MySQL y a funciones Lambda con dos ambientes (prod/dev) a través de API Gateway.

## 📐 Arquitectura

```
Internet
   │
   ▼
[EC2 — Nginx :80]          ← IP pública auto-detectada en cada arranque
   │
   ▼
[Gunicorn :8000]
   │
   ├──► [RDS MySQL]                   /db-check
   │
   └──► [API Gateway]
             ├──► [Lambda alias: prod] → "Hola prod"   /lambda/prod
             └──► [Lambda alias: dev]  → "Hola dev"    /lambda/dev
```

## ✅ Prerrequisitos

- Una instancia **EC2** con Amazon Linux (limpia)
- Una base de datos **RDS MySQL** (limpia)
- Acceso a la **consola de AWS** con permisos para Lambda, API Gateway, EC2 y RDS
- Archivo `.pem` para conectarte a la EC2 por SSH

> 💡 Todos los recursos deben estar en la **misma región**. Esta guía usa `us-east-2` (Ohio). Si usas otra región, reemplaza `us-east-2` en todos los comandos.

---

## PASO 1 — Configurar Security Groups

Este paso se hace antes que todo. Sin las reglas correctas, nada conecta.

### 1.1 Security Group de la EC2

Ve a **EC2 → Security Groups**, selecciona el SG de tu instancia EC2 → **Edit inbound rules**. Agrega las siguientes reglas si no existen:

| Type | Port | Source | Para qué |
|---|---|---|---|
| SSH | 22 | 0.0.0.0/0 | Conectarte por terminal al servidor |
| HTTP | 80 | 0.0.0.0/0 | Tráfico web a través de Nginx |
| HTTPS | 443 | 0.0.0.0/0 | Tráfico web seguro (opcional) |

Clic en **Save rules**. Anota el **Security Group ID** de tu EC2 (formato `sg-xxxxxxxxxxxxxxxxx`), lo necesitas en el siguiente paso.

### 1.2 Security Group de la RDS

Ve a **Aurora and RDS → Databases → tu-base-de-datos → Connectivity & security → VPC security groups**. Haz clic en el SG de la RDS → **Edit inbound rules**. Agrega:

| Type | Port | Source | Para qué |
|---|---|---|---|
| MySQL/Aurora | 3306 | SG ID de tu EC2 | Solo la EC2 puede acceder a la RDS |

> ⚠️ El source debe ser el **Security Group ID de la EC2** (ej: `sg-02fa28922186fa488`), no `0.0.0.0/0`. Así solo tu instancia tiene acceso a la base de datos.

Clic en **Save rules**.

---

## PASO 2 — Crear la función Lambda

### 2.1 Crear la función

**Lambda → Functions → Create function**

| Campo | Valor |
|---|---|
| Option | Author from scratch |
| Function name | `flask-hello` |
| Runtime | **Python 3.12** |
| Architecture | x86_64 |

Clic en **Create function**.

### 2.2 Pegar el código

En la pestaña **Code**, haz clic en `lambda_function.py`, borra todo el contenido y pega:

```python
import json

def lambda_handler(event, context):
    stage = event.get("requestContext", {}).get("stage", "dev")
    mensaje = f"Hola {stage}" if stage in ("prod", "dev") else "Hola desconocido"
    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps({"mensaje": mensaje, "stage": stage})
    }
```

Clic en **Deploy**.

### 2.3 Publicar una versión

1. Clic en **Actions → Publish new version**
2. Description: `v1` → **Publish**
3. Anota el número de versión (normalmente `1`)

### 2.4 Crear alias `prod`

**Configuration → Aliases → Create alias**

| Campo | Valor |
|---|---|
| Name | `prod` |
| Version | `1` |

Clic en **Save**.

### 2.5 Crear alias `dev`

**Configuration → Aliases → Create alias**

| Campo | Valor |
|---|---|
| Name | `dev` |
| Version | `$LATEST` |

Clic en **Save**.

---

## PASO 3 — Crear API Gateway

### 3.1 Crear la API

**API Gateway → Create API → REST API → Build**

| Campo | Valor |
|---|---|
| API name | `flask-lambda-api` |
| Endpoint type | Regional |

Clic en **Create API**.

### 3.2 Crear el recurso `/hello`

Con `/` seleccionado → **Create resource** → Resource name: `hello` → **Create resource**

### 3.3 Crear el método GET

Con `/hello` seleccionado → **Create method**

| Campo | Valor |
|---|---|
| Method type | GET |
| Integration type | Lambda function |
| Lambda proxy integration | ✅ Activar |
| Lambda function | `flask-hello` |

### 3.4 Configurar el ARN con stage variable — CRÍTICO

En el campo **Lambda function**, escribe el ARN completo con el sufijo de stage variable:

```
arn:aws:lambda:us-east-2:TU_ACCOUNT_ID:function:flask-hello:${stageVariables.lambdaAlias}
```

> ⚠️ Si el campo muestra solo `flask-hello` o el ARN sin `:${stageVariables.lambdaAlias}`, edítalo y pon el ARN completo. Sin este sufijo, API Gateway no sabe qué alias invocar y siempre devuelve `Internal server error`.

Clic en **Create method** → clic **OK** en el popup de permisos.

### 3.5 Desplegar los stages por primera vez

**Deploy API → New stage → `prod` → Deploy**

**Deploy API → New stage → `dev` → Deploy**

Anota tu **API ID** desde la URL generada (los caracteres antes de `.execute-api`):
```
https://TU_API_ID.execute-api.us-east-2.amazonaws.com/prod
```

### 3.6 Configurar stage variables desde CloudShell

Abre **CloudShell** desde la barra inferior de la consola. Reemplaza `TU_API_ID` con tu valor real.

```bash
API_ID="TU_API_ID"
REGION="us-east-2"

# Stage variable prod + redeploy
aws apigateway update-stage \
  --rest-api-id $API_ID \
  --stage-name prod \
  --patch-operations op=replace,path=/variables/lambdaAlias,value=prod \
  --region $REGION

aws apigateway create-deployment \
  --rest-api-id $API_ID \
  --stage-name prod \
  --region $REGION

# Stage variable dev + redeploy
aws apigateway update-stage \
  --rest-api-id $API_ID \
  --stage-name dev \
  --patch-operations op=replace,path=/variables/lambdaAlias,value=dev \
  --region $REGION

aws apigateway create-deployment \
  --rest-api-id $API_ID \
  --stage-name dev \
  --region $REGION
```

### 3.7 Dar permisos a los aliases con wildcard `/*`

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
API_ID="TU_API_ID"
REGION="us-east-2"

aws lambda add-permission \
  --function-name "flask-hello:prod" \
  --statement-id "apigw-invoke-prod" \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:${REGION}:${ACCOUNT_ID}:${API_ID}/*/GET/hello" \
  --region $REGION

aws lambda add-permission \
  --function-name "flask-hello:dev" \
  --statement-id "apigw-invoke-dev" \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:${REGION}:${ACCOUNT_ID}:${API_ID}/*/GET/hello" \
  --region $REGION
```

> ⚠️ Es importante usar `/*` en el source ARN. El ARN específico por stage (ej: `/prod/GET/hello`) es demasiado restrictivo y falla en el test de la consola y en redeployments futuros.

### 3.8 Verificar Lambda desde CloudShell

```bash
curl https://$API_ID.execute-api.$REGION.amazonaws.com/prod/hello
# Esperado: {"mensaje": "Hola prod", "stage": "prod"}

curl https://$API_ID.execute-api.$REGION.amazonaws.com/dev/hello
# Esperado: {"mensaje": "Hola dev", "stage": "dev"}
```

Si ambos responden correctamente, Lambda y API Gateway están listos ✅

---

## PASO 4 — Crear la base de datos en RDS

RDS crea el servidor MySQL pero **no crea la base de datos automáticamente**. Debes crearla tú.

Primero obtén el endpoint de tu RDS: **Aurora and RDS → Databases → tu-base → Connectivity & security → Endpoint**.

Conéctate desde tu EC2 (o CloudShell):

```bash
mysql -u TU_USUARIO -p -h TU_ENDPOINT_RDS.rds.amazonaws.com
```

Dentro de MySQL:

```sql
CREATE DATABASE `TU_NOMBRE_DB`;
SHOW DATABASES;
EXIT;
```

> ⚠️ El nombre que uses aquí debe coincidir exactamente con `DB_NAME` en el archivo `.env` de la EC2.

---

## PASO 5 — Configurar la EC2

Conéctate a tu instancia:

```bash
ssh -i tu-key.pem ec2-user@<IP_PUBLICA_EC2>
```

### 5.1 Instalar dependencias del sistema

```bash
sudo dnf update -y
sudo dnf install -y nginx python3-pip git

# Cliente MySQL
sudo dnf clean packages
sudo dnf install -y https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm
sudo dnf install -y mysql-community-server --nogpgcheck

# Librerías Python
sudo pip3 install Flask gunicorn PyMySQL SQLAlchemy python-dotenv requests
```

### 5.2 Crear directorios con permisos correctos

```bash
sudo mkdir -p /opt/flask_app/nginx
sudo chown -R ec2-user:ec2-user /opt/flask_app
sudo mkdir -p /var/log/gunicorn
sudo chown -R ec2-user:ec2-user /var/log/gunicorn
```

> ⚠️ El `chown` en `/var/log/gunicorn` es obligatorio. Si el directorio pertenece a `root`, Gunicorn falla con `Permission denied` al intentar escribir los logs y no arranca.

### 5.3 Crear la aplicación Flask

```bash
nano /opt/flask_app/app.py
```

```python
import os
import requests
from flask import Flask, jsonify
from sqlalchemy import create_engine, text
from dotenv import load_dotenv

load_dotenv("/opt/flask_app/.env")

app = Flask(__name__)

DB_URL = (
    f"mysql+pymysql://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}"
    f"@{os.getenv('DB_HOST')}:{os.getenv('DB_PORT', 3306)}/{os.getenv('DB_NAME')}"
)
engine = create_engine(DB_URL, pool_pre_ping=True)


@app.route("/")
def home():
    return jsonify({"status": "ok", "mensaje": "Flask corriendo en EC2"})


@app.route("/db-check")
def db_check():
    try:
        with engine.connect() as conn:
            version = conn.execute(text("SELECT VERSION()")).scalar()
        return jsonify({"db_connected": True, "mysql_version": version})
    except Exception as e:
        return jsonify({"db_connected": False, "error": str(e)}), 500


@app.route("/lambda/<stage>")
def call_lambda(stage):
    if stage not in ("prod", "dev"):
        return jsonify({"error": "Usa prod o dev"}), 400
    url = os.getenv(f"LAMBDA_URL_{stage.upper()}")
    if not url:
        return jsonify({"error": f"LAMBDA_URL_{stage.upper()} no configurada"}), 500
    try:
        r = requests.get(url, timeout=5)
        return jsonify({"stage": stage, "lambda_response": r.json()})
    except Exception as e:
        return jsonify({"error": str(e)}), 500


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000)
```

`Ctrl+O` → Enter → `Ctrl+X`

### 5.4 Crear el archivo de variables de entorno

```bash
nano /opt/flask_app/.env
```

```ini
# RDS MySQL
DB_HOST=TU_ENDPOINT_RDS.rds.amazonaws.com
DB_PORT=3306
DB_NAME=TU_NOMBRE_DB
DB_USER=TU_USUARIO_RDS
DB_PASSWORD=TU_PASSWORD_RDS

# Lambda vía API Gateway
LAMBDA_URL_PROD=https://TU_API_ID.execute-api.us-east-2.amazonaws.com/prod/hello
LAMBDA_URL_DEV=https://TU_API_ID.execute-api.us-east-2.amazonaws.com/dev/hello
```

`Ctrl+O` → Enter → `Ctrl+X`

### 5.5 Crear el template de Nginx

```bash
nano /opt/flask_app/nginx/flask_app.conf.tpl
```

```nginx
server {
    listen 80;
    server_name __SERVER_IP__;

    access_log /var/log/nginx/flask_app.access.log;
    error_log  /var/log/nginx/flask_app.error.log;

    location / {
        proxy_pass         http://127.0.0.1:8000;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_read_timeout 60s;
    }
}
```

`Ctrl+O` → Enter → `Ctrl+X`

### 5.6 Crear el script de auto-detección de IP

```bash
sudo nano /usr/local/bin/update_ip.sh
```

```bash
#!/bin/bash
set -euo pipefail

LOG="/var/log/update_nginx_ip.log"
NGINX_CONF="/etc/nginx/conf.d/flask_app.conf"
TPL="/opt/flask_app/nginx/flask_app.conf.tpl"

echo "[$(date)] Detectando IP publica via IMDSv2..." >> "$LOG"

TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

PUBLIC_IP=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  "http://169.254.169.254/latest/meta-data/public-ipv4")

if [[ -z "$PUBLIC_IP" ]]; then
    echo "[$(date)] ERROR: no se pudo obtener la IP publica" >> "$LOG"
    exit 1
fi

echo "[$(date)] IP detectada: $PUBLIC_IP" >> "$LOG"
sed "s/__SERVER_IP__/${PUBLIC_IP}/g" "$TPL" > "$NGINX_CONF"
nginx -t 2>>"$LOG" && systemctl is-active --quiet nginx && systemctl reload nginx
echo "[$(date)] Nginx actualizado correctamente" >> "$LOG"
```

`Ctrl+O` → Enter → `Ctrl+X`

```bash
sudo chmod +x /usr/local/bin/update_ip.sh
```

### 5.7 Crear el servicio systemd para Gunicorn

```bash
sudo nano /etc/systemd/system/flask_app.service
```

```ini
[Unit]
Description=Gunicorn — Flask App
After=network.target update-nginx-ip.service

[Service]
User=ec2-user
Group=ec2-user
WorkingDirectory=/opt/flask_app
ExecStart=/usr/local/bin/gunicorn \
    --workers 3 \
    --bind 0.0.0.0:8000 \
    --access-logfile /var/log/gunicorn/access.log \
    --error-logfile /var/log/gunicorn/error.log \
    app:app
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

`Ctrl+O` → Enter → `Ctrl+X`

### 5.8 Crear el servicio systemd para auto-IP

```bash
sudo nano /etc/systemd/system/update-nginx-ip.service
```

```ini
[Unit]
Description=Detectar IP publica y actualizar configuracion de Nginx
After=network-online.target
Wants=network-online.target
Before=nginx.service flask_app.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/local/bin/update_ip.sh

[Install]
WantedBy=multi-user.target
```

`Ctrl+O` → Enter → `Ctrl+X`

### 5.9 Activar e iniciar todos los servicios

```bash
sudo systemctl daemon-reload
sudo systemctl enable update-nginx-ip nginx flask_app
sudo systemctl start update-nginx-ip
sudo systemctl start nginx
sudo systemctl start flask_app
```

### 5.10 Verificar el estado

```bash
sudo systemctl status update-nginx-ip --no-pager
sudo systemctl status nginx --no-pager
sudo systemctl status flask_app --no-pager
cat /var/log/update_nginx_ip.log
cat /etc/nginx/conf.d/flask_app.conf
```

Los tres servicios deben aparecer en verde como `active`.

---

## PASO 6 — Pruebas finales

```bash
IP_EC2="IP_PUBLICA_DE_TU_EC2"

curl http://$IP_EC2/
curl http://$IP_EC2/db-check
curl http://$IP_EC2/lambda/prod
curl http://$IP_EC2/lambda/dev
```

### Respuestas esperadas

```json
{ "status": "ok", "mensaje": "Flask corriendo en EC2" }

{ "db_connected": true, "mysql_version": "8.0.X" }

{ "stage": "prod", "lambda_response": { "mensaje": "Hola prod", "stage": "prod" } }

{ "stage": "dev", "lambda_response": { "mensaje": "Hola dev", "stage": "dev" } }
```

---

## 🔄 Comportamiento al reiniciar la EC2

Cuando la instancia obtiene una nueva IP pública, no hay que hacer nada manualmente.

```
Boot EC2
  │
  ├─[1]─► update-nginx-ip.service   → detecta IP → genera /etc/nginx/conf.d/flask_app.conf
  ├─[2]─► nginx.service             → carga la conf con la IP correcta
  └─[3]─► flask_app.service         → Gunicorn escucha en :8000
```

```bash
# Forzar actualización de IP sin reiniciar la instancia
sudo /usr/local/bin/update_ip.sh
```

---

## 🛠️ Comandos útiles

```bash
# Reiniciar servicios
sudo systemctl restart flask_app
sudo systemctl restart nginx

# Logs en tiempo real
sudo tail -f /var/log/gunicorn/error.log
sudo tail -f /var/log/gunicorn/access.log
sudo tail -f /var/log/nginx/flask_app.access.log
sudo tail -f /var/log/update_nginx_ip.log
sudo journalctl -u flask_app -f
sudo journalctl -u update-nginx-ip -f
```

---

## ❗ Errores comunes y soluciones

| Síntoma | Causa | Solución |
|---|---|---|
| SSH: Connection timed out | Puerto 22 no abierto en el SG de la EC2 | Agregar regla SSH en el SG de la EC2 |
| `/lambda/prod` → `Internal server error` | ARN sin `:${stageVariables.lambdaAlias}` | Editar Integration Request con el ARN completo y redesplegar |
| `Invalid permissions on Lambda function` | Source ARN demasiado restrictivo | Recrear permisos con `/*` en lugar de `/prod/GET/hello` |
| `/db-check` → `timed out` | SG de RDS no permite tráfico desde la EC2 | Agregar regla MySQL/3306 en el SG de la RDS con source = SG de la EC2 |
| `/db-check` → `Unknown database` | La base de datos no fue creada en MySQL | Conectarse a RDS y ejecutar `CREATE DATABASE nombre_db` |
| Gunicorn no arranca → `Permission denied` | `/var/log/gunicorn` pertenece a root | `sudo chown -R ec2-user:ec2-user /var/log/gunicorn` |
| Flask no arranca | Error en `.env` o `app.py` | `sudo journalctl -u flask_app -n 50` |
| IP no se detecta | IMDSv2 deshabilitado | EC2 → Actions → Modify instance metadata options → habilitar IMDSv2 |

---

## 🔁 Resetear permisos de Lambda

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
API_ID="TU_API_ID"
REGION="us-east-2"

aws lambda remove-permission --function-name "flask-hello:prod" --statement-id "apigw-invoke-prod" --region $REGION
aws lambda remove-permission --function-name "flask-hello:dev"  --statement-id "apigw-invoke-dev"  --region $REGION

aws lambda add-permission \
  --function-name "flask-hello:prod" \
  --statement-id "apigw-invoke-prod" \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:${REGION}:${ACCOUNT_ID}:${API_ID}/*/GET/hello" \
  --region $REGION

aws lambda add-permission \
  --function-name "flask-hello:dev" \
  --statement-id "apigw-invoke-dev" \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:${REGION}:${ACCOUNT_ID}:${API_ID}/*/GET/hello" \
  --region $REGION
```

## 🔁 Redesplegar API Gateway

```bash
API_ID="TU_API_ID"
REGION="us-east-2"

aws apigateway update-stage \
  --rest-api-id $API_ID --stage-name prod \
  --patch-operations op=replace,path=/variables/lambdaAlias,value=prod \
  --region $REGION
aws apigateway create-deployment --rest-api-id $API_ID --stage-name prod --region $REGION

aws apigateway update-stage \
  --rest-api-id $API_ID --stage-name dev \
  --patch-operations op=replace,path=/variables/lambdaAlias,value=dev \
  --region $REGION
aws apigateway create-deployment --rest-api-id $API_ID --stage-name dev --region $REGION
```

---

## 📋 Referencia rápida

### Endpoints

| Endpoint | Descripción |
|---|---|
| `GET /` | Health check de Flask |
| `GET /db-check` | Verifica conexión a RDS MySQL |
| `GET /lambda/prod` | Invoca Lambda stage prod → "Hola prod" |
| `GET /lambda/dev` | Invoca Lambda stage dev → "Hola dev" |

### Variables de entorno requeridas

| Variable | Ejemplo |
|---|---|
| `DB_HOST` | `db.xxxxxx.us-east-2.rds.amazonaws.com` |
| `DB_PORT` | `3306` |
| `DB_NAME` | `nombre_de_tu_db` |
| `DB_USER` | `admin` |
| `DB_PASSWORD` | `tu_password` |
| `LAMBDA_URL_PROD` | `https://TU_API_ID.execute-api.us-east-2.amazonaws.com/prod/hello` |
| `LAMBDA_URL_DEV` | `https://TU_API_ID.execute-api.us-east-2.amazonaws.com/dev/hello` |

### Puertos requeridos

| Recurso | Puerto | Origen |
|---|---|---|
| EC2 (SSH) | 22 | 0.0.0.0/0 |
| EC2 (HTTP) | 80 | 0.0.0.0/0 |
| RDS (MySQL) | 3306 | SG de la EC2 |