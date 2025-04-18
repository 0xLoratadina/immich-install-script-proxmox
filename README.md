
# 🚀 Instalación de Immich en Proxmox con ZFS (modo espejo)

Este tutorial guía la instalación de [Immich](https://github.com/immich-app/immich), usando:

- 🌧️ Contenedor LXC no privilegiado (Debian 12)
- 🧠 Docker + Docker Compose
- 📦 Dataset ZFS `DataDell/immich-data` (en disco espejo)
- 🔐 Contraseñas personalizadas

---

## 🔧 Requisitos previos

- Proxmox con ZFS habilitado
- Dataset ZFS `DataDell/immich-data` (modo espejo)
- Plantilla: `debian-12-standard_12.7-1_amd64.tar.zst`
- Acceso root al nodo

---

## 📝 Configuración base

| Elemento              | Valor                    |
|-----------------------|--------------------------|
| Nombre del contenedor | `immich`                 |
| VMID del CT           | `100`                    |
| Dataset de datos      | `DataDell/immich-data`   |
| Contraseña root CT    | `immserver`              |
| Contraseña DB         | `immserver-db`           |
| IP (DHCP)             | `192.168.1.xx`           |
| Puerto de Immich      | `2283`                   |

---

## 📦 Paso 1: Crear dataset ZFS

```bash
zfs create DataDell/immich-data
```

---

## 📦 Paso 2: Crear el contenedor LXC no privilegiado

```bash
pct create 100 local:vztmpl/debian-12-standard_12.7-1_amd64.tar.zst \
  -hostname immich \
  -cores 2 \
  -memory 5120 \
  -swap 5120 \
  -rootfs local-lvm:10 \
  -features nesting=1 \
  -net0 name=eth0,bridge=vmbr0,ip=dhcp \
  -unprivileged 1 \
  -onboot 1 \
  -password immserver
```

---

## 🔗 Paso 3: Montar el dataset ZFS en el contenedor

```bash
pct set 100 -mp0 /DataDell/immich-data,mp=/root/immich-app
chown -R 100000:100000 /DataDell/immich-data
```

---

## 🚀 Paso 4: Iniciar el contenedor e instalar Docker

```bash
pct start 100
pct enter 100
```

Dentro del contenedor:

```bash
apt update && apt upgrade -y
apt install -y ca-certificates curl gnupg lsb-release gnupg2 software-properties-common

install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null

apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

---

## 📁 Paso 5: Descargar y configurar Immich

```bash
cd /root/immich-app
wget -O docker-compose.yml https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
wget -O .env https://github.com/immich-app/immich/releases/latest/download/example.env

# Ajustar la ruta de subida
sed -i 's|^UPLOAD_LOCATION=.*|UPLOAD_LOCATION=./library|' .env

# Crear carpeta de subida
mkdir -p library
chown -R 1000:1000 .
```

Modificar en `.env`:
```env
DB_PASSWORD=immserver-db
POSTGRES_PASSWORD=immserver-db
```

Establecer plantilla de almacenamiento recomendada en Immich (en UI o .env si aplica):

```env
UPLOAD_LOCATION={{y}}/{{y}}-{{MM}}-{{dd}}/{{filename}}
```

---

## ▶️ Paso 6: Iniciar Immich

```bash
docker compose up -d
docker compose ps
```

---

## 🌐 Acceder a Immich

```
http://[IP_del_contenedor]:2283
```

Ejemplo:

```
http://192.168.1.88:2283
```

---

## 🔐 Contraseñas

| Servicio       | Usuario      | Contraseña       |
|----------------|--------------|------------------|
| Contenedor     | root         | `immserver`      |
| PostgreSQL     | postgres     | `immserver-db`   |
| Immich admin   | definido al iniciar sesión      |

---

## SSH dentro del contenedor

```bash
apt update && apt install -y openssh-server
systemctl enable ssh
systemctl start ssh
systemctl status ssh
```

Editar configuración si es necesario:

```bash
nano /etc/ssh/sshd_config
```

Asegurarse que estén sin `#`:

```
PermitRootLogin yes
PasswordAuthentication yes
```

Reiniciar SSH:

```bash
systemctl restart ssh
```
