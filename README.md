# 🚀 Instalación de Immich en Proxmox con LXC y ZFS

Este tutorial guía paso a paso la instalación de [Immich](https://github.com/immich-app/immich), un sistema de gestión de fotos/videos con reconocimiento automático, utilizando:

- 🌧️ Contenedor LXC no privilegiado (Debian 12)
- 🧠 Docker + Docker Compose
- 📦 Dataset ZFS personalizado (`DataDell/immich-data`)
- 🔐 Contraseñas personalizadas

---

## 🔧 Requisitos previos

- Proxmox con ZFS habilitado
- Plantilla Debian 12 descargada (`debian-12-standard_12.7-1_amd64.tar.zst`)
- Acceso root al nodo

---

## 📝 Detalles de configuración

- Nombre del contenedor: `immich`
- VMID del CT: `100`
- Dataset de datos: `DataDell/immich-data`
- Contraseña de root CT: `immserver`
- Contraseña de bases de datos: `immserver-db`
- IP (asignada por DHCP): p. ej. `192.168.1.92`
- Puerto de acceso Immich: `http://192.168.1.92:2283`

---

## 🧱 Paso 1: Crea un ZFS con el nombre DataDell con los siguientes datos y luego en la shell de root inserta este comando dataset ZFS

![image](https://github.com/user-attachments/assets/942aa373-c897-4f80-9956-2441e9910cc8)

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

## 🔗 Paso 3: Montar dataset al contenedor

```bash
pct set 100 -mp0 /DataDell/immich-data,mp=/root/immich-app
chown -R 100000:100000 /DataDell/immich-data
```

---

## 🚀 Paso 4: Iniciar contenedor e instalar Docker

```bash
pct start 100
pct enter 100
```

### Dentro del contenedor:

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

## 📅 Paso 5: Descargar y configurar Immich

```bash
cd /root/immich-app
wget -O docker-compose.yml https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
wget -O .env https://github.com/immich-app/immich/releases/latest/download/example.env

# Establece la ubicación de la carpeta de subida
sed -i 's|^UPLOAD_LOCATION=.*|UPLOAD_LOCATION=./library|' .env

# Crear carpeta de subida
mkdir -p library
chown -R 1000:1000 .
```

> Opcional: puedes editar el archivo `.env` manualmente con `nano .env` y cambiar los valores de contraseña por defecto:

```
DB_PASSWORD=immserver-db
POSTGRES_PASSWORD=immserver-db
```

---

## 📦 Paso 6: Iniciar Immich

```bash
docker compose up -d
```

---

## ✅ Verificar estado

```bash
docker compose ps
```

---

## 🔐 Contraseñas

| Elemento             | Usuario      | Contraseña        |
|----------------------|--------------|-------------------|
| Contenedor LXC       | root         | `immserver`       |
| PostgreSQL Database  | postgres     | `immserver-db`    |
| Immich Admin inicial | Se define al iniciar sesión en el navegador |

---

## 🌐 Acceso a Immich

Abre tu navegador y visita:

```
http://[IP_del_contenedor]:2283
```

Ejemplo:

```
http://192.168.1.92:2283
```

---

