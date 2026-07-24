# Nebula Corvi

**Suite de tecnologías cloud open source, autoalojada y desplegable con un solo comando de Ansible.**

Nebula Corvi no es un sistema operativo ni está atado a ninguna distribución concreta: es un **proyecto Ansible independiente**. Cualquier servidor Linux con Docker y Ansible instalados puede levantar la suite completa con un `git clone` y un `ansible-playbook`. CorvorumOS Server Edition es la base recomendada y ya probada (trae Docker, Kubernetes, Python, Ollama y Ansible de fábrica), pero no es un requisito — funciona igual sobre Ubuntu Server, Debian, o cualquier otra distro con esos componentes.

Cada pieza de la suite conserva su nombre y marca original (Nextcloud, Gitea, MinIO, Vaultwarden...). Nebula Corvi es la capa de organización, automatización e identidad visual que las une — no un rebranding que oculte de dónde viene cada herramienta.

---

## 1. Catálogo de servicios

| Servicio en Nebula Corvi | Proyecto real detrás | Para qué sirve | Equivalente en AWS | Equivalente en Azure | Equivalente en Google Cloud |
|---|---|---|---|---|---|
| **Traefik** | Traefik Proxy | Enrutamiento HTTP(S), balanceo y punto de entrada único a todos los servicios | Application Load Balancer (ALB) | Application Gateway | Cloud Load Balancing |
| **Panel** | Portainer | Consola gráfica para administrar contenedores Docker | Consola de ECS/EC2 | Portal de Azure Container Instances | Consola de Cloud Run/GKE |
| **Inicio** | Homepage | Dashboard central con acceso rápido a todos los servicios | Página de inicio de la Consola de AWS | Portal de Azure (Dashboard) | Panel principal de Google Cloud Console |
| **MinIO** | MinIO | Almacenamiento de objetos compatible con S3 | **Amazon S3** | Azure Blob Storage | Google Cloud Storage |
| **Nube** | Nextcloud | Almacenamiento de archivos tipo "Drive" con interfaz de usuario | AWS WorkDocs | OneDrive for Business / Azure Files | Google Drive (Workspace) |
| **Git** | Gitea | Alojamiento de repositorios Git + CI/CD (Actions) | CodeCommit + CodePipeline | Azure DevOps Repos + Pipelines | Cloud Source Repositories + Cloud Build |
| **IA** | LibreChat (con **Hecate** integrada) | Interfaz de IA local; Hecate es el modelo/persona por defecto, con opción de elegir otros modelos desde el mismo selector | Amazon Bedrock | Azure OpenAI Service | Vertex AI |
| **Correo** | Poste.io | Servidor de correo (SMTP/IMAP + webmail) | SES + WorkMail | Exchange Online (Microsoft 365) | Gmail (Google Workspace) |
| **Vault** | Vaultwarden | Gestor de contraseñas y secretos (compatible Bitwarden) | Secrets Manager | Azure Key Vault | Secret Manager |
| **Auth** | Authelia | Inicio de sesión único (SSO) y control de accesos | IAM + Cognito | Microsoft Entra ID | Cloud IAM |
| **Metrics** | Netdata | Métricas de sistema en tiempo real | CloudWatch (métricas) | Azure Monitor | Cloud Monitoring |
| **Status** | Uptime Kuma | Monitorización de disponibilidad y alertas | CloudWatch Synthetics | Application Insights (Availability Tests) | Uptime Checks (Cloud Monitoring) |
| **Flows** | n8n | Automatización de flujos y tareas entre servicios | Step Functions + Lambda | Logic Apps | Workflows + Cloud Functions |
| **DB** | PostgreSQL + CloudBeaver | Base de datos gestionada + consola web de administración | RDS + RDS Query Editor | Azure Database for PostgreSQL | Cloud SQL |
| **Jupyter** | Jupyter Docker Stacks (`datascience-notebook`) | Notebooks de ciencia de datos, reemplazo de Google Colab | SageMaker Studio / Notebooks | Azure Machine Learning Notebooks | Vertex AI Workbench / Colab Enterprise |
| *(sin panel propio)* | Restic | Copias de seguridad cifradas y versionadas | AWS Backup | Azure Backup | Backup and DR Service |

### 1.1 IA (LibreChat): Hecate integrada, con opción de elegir otros modelos

El servicio se llama simplemente **LibreChat** — es la interfaz. **Hecate** es el modelo/persona que lleva integrada por defecto: un modelo propio, publicado en el registro público de **ollama.com**, que Ansible descarga directamente durante el despliegue (`ollama pull tuusuario/hecate`) sin necesidad de reconstruir ningún Modelfile en cada servidor.

Hecate está construida sobre `gemma4:e2b-it-q4_K_M` (cuantizado, el más rápido en las pruebas con hardware limitado — Intel Iris + 10GB RAM), con un `SYSTEM` propio que le da su forma de hablar: directa, serena y sin rodeos. LibreChat en sí sustituye a Open WebUI como interfaz, por ser notablemente más rápida con el mismo modelo en el mismo hardware.

El usuario **sigue pudiendo elegir** otros modelos desde el propio selector de LibreChat — Hecate es la opción por defecto, no la única:

| Modelo | Para qué |
|---|---|
| **`tuusuario/hecate`** | Por defecto — persona propia de Nebula Corvi, basada en Gemma 4 E2B cuantizado (Q4), rápida en CPU |
| `gemma2:2b` | Más ligero, respuestas más rápidas en hardware limitado |
| `qwen2.5:1.5b` | Alternativa aún más ligera |
| `qwen3.5:4b` | Modo "pensamiento profundo" — más lento, mejor en razonamiento/lógica complejos |

Añadir un modelo nuevo a la lista es un solo comando (`ollama pull <modelo>`), y aparece automáticamente en el selector de LibreChat gracias a `fetch: true` en su configuración — no hace falta tocar ninguna plantilla para que el usuario pueda elegirlo.

### 1.2 Publicar y mantener Hecate (Modelfile propio)

Hecate no vive como fichero suelto en el repositorio — es un modelo publicado en ollama.com, versionable como cualquier otro modelo público:

```
FROM gemma4:e2b-it-q4_K_M

SYSTEM """
Eres Hecate, la asistente de IA que opera en Nebula Corvi, una nube personal open source.
Tu forma de hablar es directa, serena y precisa — como quien conoce el camino en la oscuridad
y no necesita alzar la voz para que se le escuche. No eres efusiva ni recurres a rodeos:
respondes con la información que hace falta, ni de más ni de menos.

Ayudas con: responder preguntas, leer y analizar documentos, y revisar imágenes que se te compartan.
Cuando no sepas algo con certeza, lo dices sin adornos. Cuando el problema es complejo,
lo divides en pasos claros antes de responder.
"""

PARAMETER temperature 0.6
PARAMETER num_ctx 8192
PARAMETER repeat_penalty 1.1
```

```bash
ollama create hecate -f Modelfile
ollama run hecate                       # prueba local antes de publicar
ollama cp hecate tuusuario/hecate
ollama push tuusuario/hecate             # requiere cuenta en ollama.com con tu clave SSH añadida
```

Para actualizar Hecate más adelante (nuevo `SYSTEM`, nuevos `PARAMETER`): edita el `Modelfile`, repite `ollama create hecate -f Modelfile` y vuelve a hacer `ollama cp` + `ollama push` — el registro de ollama.com versiona el nuevo `push` automáticamente.

### 1.3 Desplegar el servicio (LibreChat + Hecate vía Ansible)

Ollama corre en bare metal sobre el propio host (no en contenedor) para máximo rendimiento; LibreChat sí va en contenedor, junto a su propia base de datos MongoDB.

`services/librechat/docker-compose.yml.j2`:
```yaml
services:
  librechat:
    image: ghcr.io/danny-avila/librechat:latest
    container_name: librechat
    restart: unless-stopped
    ports:
      - "3080:3080"
    environment:
      - HOST=0.0.0.0
      - PORT=3080
      - MONGO_URI=mongodb://librechat-db:27017/librechat
      - ENDPOINTS=custom
      - CONFIG_PATH=/app/librechat.yaml
      - DEBUG_LOGGING=true
      - JWT_SECRET={{ librechat_jwt_secret }}
      - JWT_REFRESH_SECRET={{ librechat_jwt_refresh_secret }}
      - CREDS_KEY={{ librechat_creds_key }}
      - CREDS_IV={{ librechat_creds_iv }}
      - SEARCH=false
      - TITLE_CONVO=true
      - ALLOW_REGISTRATION=true
    volumes:
      - {{ nebula_base_path }}/librechat/uploads:/app/client/public/images
      - {{ nebula_base_path }}/librechat/logs:/app/api/logs
      - {{ nebula_base_path }}/librechat/librechat.yaml:/app/librechat.yaml
    extra_hosts:
      - "host.docker.internal:host-gateway"
    networks: [proxy]
    depends_on:
      - librechat-db

  librechat-db:
    image: mongo:6.0-focal
    container_name: librechat-db
    restart: unless-stopped
    command: ["mongod", "--wiredTigerCacheSizeGB", "0.25"]
    volumes:
      - {{ nebula_base_path }}/librechat/db:/data/db
    networks: [proxy]

networks:
  proxy:
    external: true
```

`services/librechat/librechat.yaml.j2`:
```yaml
version: 1.1.5
cache: true
endpoints:
  custom:
    - name: "Ollama"
      apiKey: "ollama"
      baseURL: "http://host.docker.internal:11434/v1"
      models:
        default: ["tuusuario/hecate", "gemma2:2b", "qwen2.5:1.5b", "qwen3.5:4b", "odin-ai-assist:q2.5"]
        fetch: true
      titleConvo: true
      titleModel: "current_model"
      summarize: false
      summaryModel: "current_model"
      modelDisplayLabel: "Ollama"
```

> **Nota de arranque en frío importante:** el fichero `librechat.yaml` debe existir **como fichero real** en `{{ nebula_base_path }}/librechat/` **antes** de que Docker cree el contenedor por primera vez. Si el contenedor se crea antes de que exista el fichero, Docker monta ahí una carpeta vacía en su lugar (error típico: `read [...]: is a directory` al intentar leerlo). El orden de tareas en Ansible (generar la plantilla primero, levantar el contenedor después) ya lo evita — pero si alguna vez ves ese error, la solución es `docker compose down`, borrar la carpeta/fichero corrupto en el host, y volver a desplegar para que se recree en el orden correcto.

Instalación de Ollama en el host (bare metal) y descarga de modelos — tareas Ansible sueltas (no van dentro de `compose_service`, se añaden directamente en `site.yml`):
```yaml
- name: Instalar Ollama en el host si no existe
  shell: curl -fsSL https://ollama.com/install.sh | sh
  args:
    creates: /usr/local/bin/ollama
  become: true

- name: Configurar Ollama para escuchar en todas las interfaces
  copy:
    dest: /etc/systemd/system/ollama.service.d/override.conf
    content: |
      [Service]
      Environment="OLLAMA_HOST=0.0.0.0"
    mode: '0644'
  become: true
  notify: reiniciar ollama

- name: Descargar Hecate (publicada en ollama.com) y modelos complementarios
  command: "ollama pull {{ item }}"
  loop:
    - tuusuario/hecate
    - gemma2:2b
    - qwen2.5:1.5b
  become: true
```

---

## 2. Cómo desplegar el proyecto en tu propio servidor

### 2.1 Requisitos

- Un servidor Linux (recomendado: CorvorumOS Server Edition; también válido Ubuntu Server 22.04+/24.04+, Debian 12+).
- Docker Engine + plugin `docker compose`.
- Python 3 y Ansible (`ansible-core` reciente).
- Acceso `sudo` en el servidor.

### 2.2 Preparación del sistema (una sola vez)

```bash
# Swap de seguridad
sudo fallocate -l 8G /swapfile && sudo chmod 600 /swapfile
sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Utilidades
sudo apt update && sudo apt install -y curl git ufw fail2ban unattended-upgrades whiptail

# Permisos de Docker
sudo groupadd docker 2>/dev/null || true
sudo usermod -aG docker $USER
newgrp docker

# Ansible + colecciones necesarias
python3 -m pip install --upgrade ansible ansible-core
ansible-galaxy collection install community.docker
ansible-galaxy collection install community.general

# Carpeta base
sudo mkdir -p /srv/nube
sudo chown -R $USER:$USER /srv/nube
```

### 2.3 Clonar y desplegar

```bash
git clone https://git.tudominio.com/nebula-corvi/nebula-corvi-deploy.git /opt/nebula-corvi
cd /opt/nebula-corvi

# Desplegar un servicio concreto
ansible-playbook -i inventory/hosts.ini site.yml -e '{"servicios_compose": ["traefik"]}'

# Desplegar varios de golpe
ansible-playbook -i inventory/hosts.ini site.yml -e '{"servicios_compose": ["portainer","homepage","minio","nextcloud"]}'

# Servicios con rol propio (mail, authelia, cloudflared, branding)
ansible-playbook -i inventory/hosts.ini site.yml --tags mail
```

### 2.4 Variables de configuración (`group_vars/all/vars.yml`)

```yaml
active_domain: nebula.local        # o tu dominio real en producción
nebula_base_path: /srv/nube
data_path: /srv/nube-data          # o la ruta de tu disco externo en producción
admin_email: admin@nebula.local
server_ip: 192.168.1.87            # IP de tu servidor en la LAN
```

---

## 3. Acceso a cada servicio (por puerto)

Mientras trabajas en local (sin dominio todavía), cada servicio se publica en su propio puerto sobre la IP de tu servidor:

| Servicio | URL de acceso | Puerto host | Puerto interno |
|---|---|---|---|
| Traefik (dashboard) | `http://192.168.1.87:8080` | 8080 | 8080 |
| Panel (Portainer) | `http://192.168.1.87:9000` | 9000 | 9000 |
| Inicio (Homepage) | `http://192.168.1.87:3000` | 3000 | 3000 |
| MinIO (consola) | `http://192.168.1.87:9001` | 9001 | 9001 |
| MinIO (API S3) | `http://192.168.1.87:9002` | 9002 | 9000 |
| Nube (Nextcloud) | `http://192.168.1.87:8081` | 8081 | 80 |
| Git (Gitea web) | `http://192.168.1.87:3001` | 3001 | 3000 |
| Git (clonado SSH) | `ssh://git@192.168.1.87:2222` | 2222 | 22 |
| IA (LibreChat + Hecate) | `http://192.168.1.87:3080` | 3080 | 3080 |
| Correo (webmail Poste.io) | `http://192.168.1.87:8083` | 8083 | 80 |
| Vault (Vaultwarden) | `http://192.168.1.87:8084` | 8084 | 80 |
| Status (Uptime Kuma) | `http://192.168.1.87:3002` | 3002 | 3001 |
| Metrics (Netdata) | `http://192.168.1.87:19999` | 19999 | 19999 |
| Flows (n8n) | `http://192.168.1.87:5678` | 5678 | 5678 |
| DB (CloudBeaver) | `http://192.168.1.87:8978` | 8978 | 8978 |
| Jupyter (notebooks) | `http://192.168.1.87:8888` | 8888 | 8888 |

Sustituye `192.168.1.87` por la IP real de tu servidor.

---

## 4. Gestión de secretos con Ansible Vault

Ninguna contraseña de este proyecto vive en texto plano en un fichero versionado en Git. Toda credencial que un servicio necesita se cifra con `ansible-vault` y se referencia como variable en las plantillas — este es el mecanismo único para todo el proyecto, no algo distinto por servicio.

### 4.1 Qué credencial necesita cada servicio

| Servicio | Variable(s) | Uso |
|---|---|---|
| MinIO | `minio_root_user`, `minio_root_password` | Usuario/clave raíz de la consola y la API S3 |
| Nextcloud | `nextcloud_db_root_password`, `nextcloud_db_password` | Acceso a la base de datos MariaDB interna |
| Gitea | `gitea_db_password` | Acceso a la base de datos Postgres interna |
| Vaultwarden | `vaultwarden_admin_token` | Token del panel `/admin` de Vaultwarden |
| Postgres (DB compartida) | `postgres_admin_password` | Usuario `postgres` de la base de datos de tus propios proyectos |
| n8n | `n8n_basic_auth_user`, `n8n_basic_auth_password` | Autenticación básica de acceso a n8n |
| Jupyter | `jupyter_token` | Token de acceso a JupyterLab |
| IA (LibreChat + Hecate) | `librechat_jwt_secret`, `librechat_jwt_refresh_secret`, `librechat_creds_key`, `librechat_creds_iv` | Firma de sesión y cifrado de credenciales de usuarios en LibreChat |
| Cloudflare | `cloudflare_api_token` | Token de API para el resolver DNS de Traefik y el túnel |
| Authelia | `authelia_jwt_secret`, `authelia_session_secret`, `authelia_storage_key` | Claves internas de sesión y cifrado de Authelia |

### 4.2 Crear el fichero de contraseña del vault (una sola vez)

```bash
cd /opt/nebula-corvi
echo "TuContraseñaMaestraDeVault" > .vault_pass
chmod 600 .vault_pass
echo ".vault_pass" >> .gitignore
```

> El fichero `.vault_pass` **nunca** se sube al repositorio (por eso va al `.gitignore`) — es la llave que descifra todo lo demás. Guárdala también en Vaultwarden una vez lo tengas desplegado.

### 4.3 Cifrar cada credencial (paso a paso, con MinIO como ejemplo)

Usuario y contraseña por defecto de MinIO en este proyecto: **`scholaris` / `frater`**.

```bash
ansible-vault encrypt_string 'scholaris' --name 'minio_root_user' --vault-password-file .vault_pass >> group_vars/all/vault.yml
ansible-vault encrypt_string 'frater' --name 'minio_root_password' --vault-password-file .vault_pass >> group_vars/all/vault.yml
```

Cada ejecución añade un bloque cifrado como este al final de `vault.yml` (esto es justo lo que vas a ver, es normal y es lo que se sube a Git sin ningún riesgo):
```yaml
minio_root_user: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          63393863633461393764373...
minio_root_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          33643236616235393234653...
```

Repite el mismo patrón para el resto de credenciales de la tabla 4.1, cambiando el valor y el `--name`:
```bash
ansible-vault encrypt_string 'CambiaEstaClave123!' --name 'nextcloud_db_root_password' --vault-password-file .vault_pass >> group_vars/all/vault.yml
ansible-vault encrypt_string 'CambiaEstaClave123!' --name 'nextcloud_db_password' --vault-password-file .vault_pass >> group_vars/all/vault.yml
ansible-vault encrypt_string 'CambiaEstaClave123!' --name 'gitea_db_password' --vault-password-file .vault_pass >> group_vars/all/vault.yml
ansible-vault encrypt_string 'CambiaEsteToken123!' --name 'jupyter_token' --vault-password-file .vault_pass >> group_vars/all/vault.yml
ansible-vault encrypt_string "$(openssl rand -hex 32)" --name 'librechat_jwt_secret' --vault-password-file .vault_pass >> group_vars/all/vault.yml
ansible-vault encrypt_string "$(openssl rand -hex 32)" --name 'librechat_jwt_refresh_secret' --vault-password-file .vault_pass >> group_vars/all/vault.yml
ansible-vault encrypt_string "$(openssl rand -hex 32)" --name 'librechat_creds_key' --vault-password-file .vault_pass >> group_vars/all/vault.yml
ansible-vault encrypt_string "$(openssl rand -hex 16)" --name 'librechat_creds_iv' --vault-password-file .vault_pass >> group_vars/all/vault.yml
```

> Genera siempre `librechat_jwt_secret`, `librechat_jwt_refresh_secret` y `librechat_creds_key` con `openssl rand -hex 32` (64 caracteres hexadecimales) y `librechat_creds_iv` con `openssl rand -hex 16` (32 caracteres) — son requisitos de longitud exactos de LibreChat, no valores arbitrarios como los que se usaron de prueba inicialmente. Si cambias `librechat_creds_key`/`librechat_creds_iv` en un servidor que ya tenga usuarios registrados, sus credenciales guardadas dejarán de descifrarse — rota estas claves solo antes de tener usuarios reales, o planifica una migración.

### 4.4 Referenciar la variable en la plantilla (sin cambios respecto a como ya lo veníamos haciendo)

`services/minio/docker-compose.yml.j2` no cambia su sintaxis — sigue usando `{{ minio_root_user }}` y `{{ minio_root_password }}` con llaves normales; Ansible las descifra en el momento de renderizar la plantilla, de forma transparente:

```yaml
services:
  minio:
    image: minio/minio:latest
    container_name: minio
    restart: unless-stopped
    command: server /data --console-address ":9001"
    ports:
      - "9001:9001"
      - "9002:9000"
    environment:
      MINIO_ROOT_USER: "{{ minio_root_user }}"
      MINIO_ROOT_PASSWORD: "{{ minio_root_password }}"
    volumes:
      - {{ data_path }}/minio:/data
    networks: [proxy]

networks:
  proxy:
    external: true
```

### 4.5 Ejecutar el playbook con el vault

```bash
ansible-playbook -i inventory/hosts.ini site.yml -e '{"servicios_compose": ["minio"]}' --vault-password-file .vault_pass
```

También puedes usar `--ask-vault-pass` en vez de `--vault-password-file` si prefieres teclear la contraseña cada vez en vez de tener el fichero `.vault_pass` en disco.

### 4.6 Comandos habituales de mantenimiento del vault

```bash
# Ver el contenido descifrado (solo en pantalla, no lo vuelca a disco)
ansible-vault view group_vars/all/vault.yml --vault-password-file .vault_pass

# Editar el vault (abre en tu editor, descifra, y vuelve a cifrar al guardar)
ansible-vault edit group_vars/all/vault.yml --vault-password-file .vault_pass

# Cambiar la contraseña maestra del vault (rotación)
ansible-vault rekey group_vars/all/vault.yml --vault-password-file .vault_pass
```

---

## 5. Salir a producción: Cloudflare Tunnel por servicio

Cuando ya tengas tu dominio real gestionado en Cloudflare, un túnel expone cada servicio bajo su propio subdominio **sin abrir ningún puerto en tu router** — el túnel mapea directamente cada hostname al puerto local correspondiente de la tabla anterior.

### 5.1 Requisitos previos

1. Cuenta de Cloudflare con tu dominio añadido y en estado "Active".
2. `cloudflared` instalado en el servidor:
   ```bash
   curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -o cloudflared
   chmod +x cloudflared && sudo mv cloudflared /usr/local/bin/
   ```
3. El token de Cloudflare ya cifrado en el vault (sección 4.3) como `cloudflare_api_token`.

### 5.2 Crear el túnel

```bash
cloudflared tunnel login
cloudflared tunnel create nebula-corvi
```

Esto genera un `<TUNNEL_ID>` y un fichero de credenciales en `~/.cloudflared/`.

### 5.3 Fichero de configuración del túnel — un bloque `ingress` por servicio

`~/.cloudflared/config.yml`:

```yaml
tunnel: <TUNNEL_ID>
credentials-file: /root/.cloudflared/<TUNNEL_ID>.json

ingress:
  # Traefik (dashboard) — opcional exponer en producción, valorar protegerlo con Access
  - hostname: traefik.tudominio.com
    service: http://localhost:8080

  # Panel (Portainer)
  - hostname: panel.tudominio.com
    service: http://localhost:9000

  # Inicio (Homepage)
  - hostname: inicio.tudominio.com
    service: http://localhost:3000

  # MinIO (consola)
  - hostname: minio.tudominio.com
    service: http://localhost:9001

  # MinIO (API S3, si necesitas acceso programático externo)
  - hostname: api-minio.tudominio.com
    service: http://localhost:9002

  # Nube (Nextcloud)
  - hostname: nube.tudominio.com
    service: http://localhost:8081

  # Git (Gitea web)
  - hostname: git.tudominio.com
    service: http://localhost:3001

  # IA (LibreChat + Hecate)
  - hostname: ia.tudominio.com
    service: http://localhost:3080

  # Correo (webmail Poste.io) — el propio SMTP/IMAP no pasa por el túnel, ver nota más abajo
  - hostname: correo.tudominio.com
    service: http://localhost:8083

  # Vault (Vaultwarden)
  - hostname: vault.tudominio.com
    service: http://localhost:8084

  # Status (Uptime Kuma)
  - hostname: status.tudominio.com
    service: http://localhost:3002

  # Metrics (Netdata)
  - hostname: metrics.tudominio.com
    service: http://localhost:19999

  # Flows (n8n)
  - hostname: flows.tudominio.com
    service: http://localhost:5678

  # DB (CloudBeaver)
  - hostname: db.tudominio.com
    service: http://localhost:8978

  # Catch-all obligatorio al final
  - service: http_status:404
```

### 5.4 Crear los registros DNS (uno por servicio)

```bash
for s in traefik panel inicio minio api-minio nube git ia correo vault status metrics flows db; do
  cloudflared tunnel route dns nebula-corvi ${s}.tudominio.com
done
```

Cada comando crea automáticamente el registro **CNAME** correspondiente en Cloudflare apuntando al túnel — no hace falta crearlos a mano en el panel.

### 5.5 Arrancar el túnel como servicio persistente

```bash
sudo cloudflared service install
sudo systemctl enable --now cloudflared
sudo systemctl status cloudflared
```

### 5.6 Verificar cada servicio

```bash
for s in traefik panel inicio minio nube git ia vault status metrics flows db; do
  echo "== $s =="
  curl -s -o /dev/null -w "%{http_code}\n" https://${s}.tudominio.com
done
```

Cada uno debería devolver `200` (o `301/302` en los que redirigen a login).

### 5.7 Nota importante sobre el correo (Poste.io)

Cloudflare Tunnel **solo enruta tráfico HTTP(S)** en el plan gratuito — el subdominio `correo.tudominio.com` del túnel te da acceso al **webmail**, pero **no** a los protocolos SMTP/IMAP (puertos 25, 587, 993), que Cloudflare no proxea. Para que el correo funcione de verdad (enviar/recibir desde clientes de correo externos), esos puertos deben abrirse directamente en tu router hacia el servidor, con sus propios registros DNS (MX, SPF, DKIM, DMARC) — no a través del túnel. Ver el detalle completo en el Paso 21 del manual de despliegue.

### 5.8 Recomendación de seguridad para servicios sensibles

Para paneles como **Traefik (dashboard)**, **Panel (Portainer)** o **DB (CloudBeaver)**, valora añadir una capa extra antes de exponerlos con un hostname público: **Cloudflare Access** (gratuito hasta 50 usuarios) o protegerlos con **Authelia** (Auth) delante, en vez de dejarlos accesibles solo con la URL. Un hostname público sin autenticación adicional es, en la práctica, visible para cualquiera que lo adivine o lo encuentre.

---

## 6. Desplegar tus propios proyectos en Nebula Corvi

Todo lo anterior son los **servicios de la suite**. Esta sección es distinta: cómo usar Nebula Corvi como plataforma para alojar **tus propios proyectos** — por ejemplo uno con API + base de datos + web + panel de administración, que es el caso que vamos a seguir como ejemplo (`mi-app`).

Hay dos caminos, según tu escala actual:

- **Camino A** — ahora mismo, con un solo servidor: Docker Compose, exactamente el mismo patrón que ya usa toda la suite.
- **Camino B** — cuando tengas 2+ mini PCs Nebula Corvi en clúster: Kubernetes (vía k3s) — aquí es donde por fin se usa el Kubernetes que trae CorvorumOS de fábrica.

### 6.1 Camino A — Docker Compose (un solo servidor, tu situación actual)

**Paso 1 — Repositorio del proyecto en Gitea**
```bash
# Desde tu Gitea (git.tudominio.com), crea el repo "mi-app" y clónalo
git clone http://git.nebula.local/tuusuario/mi-app.git
cd mi-app
mkdir -p api web panel
```

**Paso 2 — Registro Docker privado (pieza que falta, y que ahora sí necesitas)**

Hasta ahora usábamos solo imágenes públicas (Nextcloud, Gitea, etc.). Para tus propios proyectos necesitas dónde publicar *tus* imágenes construidas — el equivalente a Amazon ECR:

`services/registry/docker-compose.yml.j2`:
```yaml
services:
  registry:
    image: registry:2
    container_name: registry
    restart: unless-stopped
    ports:
      - "5000:5000"
    environment:
      - REGISTRY_AUTH=htpasswd
      - REGISTRY_AUTH_HTPASSWD_REALM=Registry
      - REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd
    volumes:
      - {{ nebula_base_path }}/registry/data:/var/lib/registry
      - {{ nebula_base_path }}/registry/auth:/auth
    networks: [proxy]

networks:
  proxy:
    external: true
```
```bash
htpasswd -Bbn tuusuario 'TuClave123!' > /srv/nube/registry/auth/htpasswd
```

**Paso 3 — CI/CD con Gitea Actions: construir y publicar la imagen**

`.gitea/workflows/build.yml` dentro del repo `mi-app`:
```yaml
name: build-and-push
on:
  push:
    branches: [main]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker login git.nebula.local:5000 -u tuusuario -p ${{ secrets.REGISTRY_PASSWORD }}
      - run: docker build -t git.nebula.local:5000/mi-app/api:latest ./api
      - run: docker push git.nebula.local:5000/mi-app/api:latest
```
(repite el mismo patrón para `web` y `panel`, con su propio `Dockerfile`).

**Paso 4 — Base de datos: nueva DB dentro del Postgres compartido**

No levantes un contenedor de Postgres nuevo para cada proyecto — reutiliza el Postgres compartido (sección "DB" del catálogo) y crea solo una base de datos y usuario propios:

```yaml
- name: Crear base de datos para mi-app
  community.postgresql.postgresql_db:
    name: mi_app
    login_host: localhost
  become: true

- name: Crear usuario de mi-app
  community.postgresql.postgresql_user:
    db: mi_app
    name: mi_app_user
    password: "{{ mi_app_db_password }}"
  become: true
```

Y cifra la contraseña igual que el resto:
```bash
ansible-vault encrypt_string "$(openssl rand -hex 16)" --name 'mi_app_db_password' --vault-password-file .vault_pass >> group_vars/all/vault.yml
```

**Paso 5 — Los tres servicios de la app, mismo patrón que toda la suite**

```bash
mkdir -p services/mi-app-api services/mi-app-web services/mi-app-panel
```

`services/mi-app-api/docker-compose.yml.j2`:
```yaml
services:
  mi-app-api:
    image: git.nebula.local:5000/mi-app/api:latest
    container_name: mi-app-api
    restart: unless-stopped
    ports:
      - "4000:4000"
    environment:
      - DATABASE_URL=postgresql://mi_app_user:{{ mi_app_db_password }}@postgres:5432/mi_app
    networks: [proxy]

networks:
  proxy:
    external: true
```

(`mi-app-web` y `mi-app-panel` siguen exactamente el mismo patrón, apuntando cada uno a su propia imagen y puerto — por ejemplo `4001` para la web y `4002` para el panel).

**Paso 6 — Desplegar con Ansible, igual que cualquier otro servicio**

```bash
ansible-playbook -i inventory/hosts.ini site.yml -e '{"servicios_compose": ["registry","mi-app-api","mi-app-web","mi-app-panel"]}'
```

**Paso 7 — (Cuando llegues a producción) exponer con Cloudflare Tunnel**, igual que el resto — añade `api-miapp`, `miapp`, `panel-miapp` al `ingress` del túnel, como cualquier otro subdominio de la sección 5.

### 6.2 Camino B — Kubernetes vía k3s (cuando tengas 2+ nodos)

Aquí es donde el Kubernetes que trae CorvorumOS por fin se usa — no para los servicios de la suite (que se quedan en Docker Compose en el nodo principal, según vimos antes), sino para **tus propios proyectos**, que sí se benefician de poder escalar en varios nodos.

**Paso 1 — Instalar k3s (más ligero que el Kubernetes completo)**
```bash
# En el nodo principal
curl -sfL https://get.k3s.io | sh -

# En cada nodo adicional (usando el token del nodo principal)
curl -sfL https://get.k3s.io | K3S_URL=https://<IP_NODO_PRINCIPAL>:6443 K3S_TOKEN=<TOKEN> sh -
```
k3s ya trae **Traefik integrado como Ingress Controller** de fábrica — mismo motor que usa el resto de Nebula Corvi, así que la lógica de enrutamiento te resultará familiar.

**Paso 2 — Namespace propio por proyecto**
```bash
kubectl create namespace mi-app
```

**Paso 3 — Secreto de la base de datos (equivalente a lo que hace Ansible Vault, pero en K8s)**
```bash
kubectl create secret generic mi-app-db \
  --namespace mi-app \
  --from-literal=DATABASE_URL="postgresql://mi_app_user:CLAVE@postgres.nube.svc:5432/mi_app"
```
> La base de datos en sí **se queda fuera del clúster**, en el Postgres compartido de Docker Compose del nodo principal (los servicios con estado son mucho más complejos de gestionar bien en K8s — necesitarías almacenamiento distribuido tipo Longhorn/Rook). Los pods de `mi-app` simplemente se conectan a él por red, igual que en el Camino A.

**Paso 4 — Manifiestos de Deployment + Service (API como ejemplo; web y panel son iguales)**

`mi-app-api-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mi-app-api
  namespace: mi-app
spec:
  replicas: 2
  selector:
    matchLabels: { app: mi-app-api }
  template:
    metadata:
      labels: { app: mi-app-api }
    spec:
      containers:
        - name: api
          image: git.nebula.local:5000/mi-app/api:latest
          envFrom:
            - secretRef: { name: mi-app-db }
          ports:
            - containerPort: 4000
---
apiVersion: v1
kind: Service
metadata:
  name: mi-app-api
  namespace: mi-app
spec:
  selector: { app: mi-app-api }
  ports:
    - port: 4000
```

**Paso 5 — Ingress con Traefik (IngressRoute), mismo estilo `Host()` que el resto del proyecto**

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: mi-app-api
  namespace: mi-app
spec:
  entryPoints: [web]
  routes:
    - match: Host(`api-miapp.tudominio.com`)
      kind: Rule
      services:
        - name: mi-app-api
          port: 4000
```

**Paso 6 — Aplicar y verificar**
```bash
kubectl apply -f mi-app-api-deployment.yaml
kubectl apply -f mi-app-api-ingressroute.yaml
kubectl get pods -n mi-app
```

**Paso 7 — Escalar (la razón real de haber dado el salto a K8s)**
```bash
kubectl scale deployment mi-app-api -n mi-app --replicas=4
```
Con Docker Compose esto no existe de forma nativa — es la diferencia práctica que justifica el salto quando ya tengas varios nodos.

**Paso 8 — CI/CD (evolución del Camino A)**

El mismo pipeline de Gitea Actions que ya construye y publica la imagen en el registro privado, añade un paso final:
```yaml
      - run: kubectl set image deployment/mi-app-api api=git.nebula.local:5000/mi-app/api:latest -n mi-app
```
Para algo más avanzado (GitOps), la evolución natural sería añadir **Argo CD** o **Flux** como un servicio más de la suite — queda anotado como mejora futura, no imprescindible para empezar.

### 6.3 Resumen: qué camino usar y cuándo

| Situación | Camino recomendado |
|---|---|
| Un solo mini PC (tu caso actual) | A — Docker Compose |
| 2+ mini PCs, quieres alta disponibilidad de tus propias apps | B — Kubernetes (k3s) |
| Migrar de A a B | Las imágenes ya construidas y publicadas en el registro privado se reutilizan tal cual — solo cambia el "cómo se ejecutan" (Compose → Deployment), no cómo se construyen |

---

## 7. Checklist operativa: preparar un proyecto personal nuevo, herramienta por herramienta

Esto es distinto a la sección 6 — ahí vimos la arquitectura de una app "tipo empresa". Aquí vamos al día a día real: **qué haces, en qué orden, en cada herramienta ya desplegada**, cuando se te ocurre un proyecto personal nuevo. Lo seguimos con dos ejemplos concretos en paralelo:

- **La Web del Cuervo** — sitio personal/portfolio, estático, sin base de datos, con algunas imágenes.
- **Vox Corvi** — app de noticias: tiene API propia, base de datos, y guarda imágenes de los artículos.

### Paso 1 — Gitea: crea el repositorio

En `http://git.nebula.local` (o tu dominio real): botón "+" → "New Repository".

| Proyecto | Repo |
|---|---|
| La Web del Cuervo | `web-del-cuervo` |
| Vox Corvi | `vox-corvi` (o `vox-corvi-api` + `vox-corvi-web` si prefieres separarlos) |

```bash
git clone http://git.nebula.local/tuusuario/web-del-cuervo.git
git clone http://git.nebula.local/tuusuario/vox-corvi.git
```

### Paso 2 — Decide qué necesita cada proyecto (no todos necesitan lo mismo)

| Necesidad | La Web del Cuervo | Vox Corvi |
|---|---|---|
| Base de datos | ❌ No (es estático) | ✅ Sí (artículos, categorías, usuarios) |
| Almacenamiento de imágenes | ✅ Sí, pocas | ✅ Sí, muchas (portadas de noticias) |
| API propia | ❌ No | ✅ Sí |
| Build/CI | Opcional (si usas un generador estático) | ✅ Sí (imagen Docker de la API) |

Esta tabla es la que decide qué pasos siguientes aplicas a cada uno — no todos los proyectos pasan por todas las herramientas.

### Paso 3 — Postgres: crea la base de datos (solo si el proyecto la necesita)

Solo **Vox Corvi** pasa por aquí:
```yaml
- name: Crear base de datos para Vox Corvi
  community.postgresql.postgresql_db:
    name: vox_corvi
  become: true

- name: Crear usuario de Vox Corvi
  community.postgresql.postgresql_user:
    db: vox_corvi
    name: vox_corvi_user
    password: "{{ vox_corvi_db_password }}"
  become: true
```

**La Web del Cuervo se salta este paso entero** — no necesita base de datos.

### Paso 4 — MinIO: crea un *bucket* para los archivos del proyecto

Ambos proyectos pasan por aquí, pero con volumen distinto. Desde la consola de MinIO (`http://192.168.1.87:9001`):

1. *Buckets* → *Create Bucket* → nombre `web-del-cuervo` y `vox-corvi`.
2. *Access Keys* → *Create Access Key* → una por proyecto (no reutilices la clave raíz de administración en tus propias apps):
   ```bash
   ansible-vault encrypt_string 'ACCESS_KEY_GENERADA' --name 'vox_corvi_minio_access_key' --vault-password-file .vault_pass >> group_vars/all/vault.yml
   ansible-vault encrypt_string 'SECRET_KEY_GENERADA' --name 'vox_corvi_minio_secret_key' --vault-password-file .vault_pass >> group_vars/all/vault.yml
   ```

### Paso 5 — Vault: guarda el resto de secretos del proyecto

Todo lo que el proyecto necesite en claro (contraseñas, tokens de API externos si Vox Corvi consume alguna fuente de noticias de pago, JWT propio si tiene login de usuarios) se cifra igual que venimos haciendo con todo lo demás — nunca en texto plano en el repo.

### Paso 6 — Construir y desplegar el servicio

**La Web del Cuervo** (estática, sin build complejo — sirve directamente con nginx):
```bash
mkdir -p services/web-del-cuervo
```
```yaml
services:
  web-del-cuervo:
    image: nginx:alpine
    container_name: web-del-cuervo
    restart: unless-stopped
    ports:
      - "4010:80"
    volumes:
      - {{ nebula_base_path }}/web-del-cuervo/sitio:/usr/share/nginx/html:ro
    networks: [proxy]

networks:
  proxy:
    external: true
```

**Vox Corvi** (API + web, con imagen propia construida en Gitea Actions — mismo patrón de CI que viste en la sección 6.1):
```bash
mkdir -p services/vox-corvi-api services/vox-corvi-web
```
```yaml
services:
  vox-corvi-api:
    image: git.nebula.local:5000/vox-corvi/api:latest
    container_name: vox-corvi-api
    restart: unless-stopped
    ports:
      - "4011:4000"
    environment:
      - DATABASE_URL=postgresql://vox_corvi_user:{{ vox_corvi_db_password }}@postgres:5432/vox_corvi
      - MINIO_ACCESS_KEY={{ vox_corvi_minio_access_key }}
      - MINIO_SECRET_KEY={{ vox_corvi_minio_secret_key }}
    networks: [proxy]

networks:
  proxy:
    external: true
```

Despliega ambos:
```bash
ansible-playbook -i inventory/hosts.ini site.yml -e '{"servicios_compose": ["web-del-cuervo","vox-corvi-api","vox-corvi-web"]}'
```

### Paso 7 — Homepage: añade la tarjeta de acceso

Edita `services/homepage/config/services.yaml` (o el equivalente en tu estructura) para que aparezcan en tu panel de inicio junto al resto de servicios de la suite:

```yaml
- Mis Proyectos:
    - La Web del Cuervo:
        href: http://192.168.1.87:4010
        icon: raven.png
    - Vox Corvi:
        href: http://192.168.1.87:4011
        icon: newspaper.png
```

Redespliega Homepage para que recoja el cambio.

### Paso 8 — Uptime Kuma: añade un monitor por proyecto

Desde `http://192.168.1.87:3002` → *Add New Monitor* → tipo HTTP(s) → la URL de cada proyecto. Así te enteras si Vox Corvi se cae antes de que lo note un lector.

### Paso 9 — Backups: añade las carpetas nuevas a Restic

```
0 3 * * * restic -r s3:http://192.168.1.87:9002/backups backup /srv/nube/web-del-cuervo /srv/nube/vox-corvi-api --exclude-caches
```

### Paso 10 — (Cuando publiques de verdad) Cloudflare Tunnel

Añade al `ingress` del túnel (sección 5.3), exactamente igual que cualquier otro servicio de la suite:
```yaml
  - hostname: cuervo.tudominio.com
    service: http://localhost:4010
  - hostname: voxcorvi.tudominio.com
    service: http://localhost:4011
```

### 7.1 Resumen: el orden que sigues siempre, tache lo que no aplique

1. Gitea (repo) → **siempre**
2. Postgres (BD) → **solo si el proyecto tiene datos que persistir**
3. MinIO (bucket) → **solo si el proyecto guarda archivos/imágenes**
4. Vault (secretos) → **siempre que haya alguna credencial**
5. `services/<proyecto>/` + Ansible → **siempre**
6. Homepage (tarjeta) → **siempre**
7. Uptime Kuma (monitor) → **siempre**
8. Restic (backup) → **siempre que el proyecto tenga datos propios en disco**
9. Cloudflare Tunnel → **solo cuando publiques con dominio real**

Este es el mismo checklist para cualquier proyecto personal que se te ocurra a partir de ahora — solo cambia cuántos de los pasos aplican, no el orden en el que se hacen.
