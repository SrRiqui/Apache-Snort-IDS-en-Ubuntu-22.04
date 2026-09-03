# 🛡️ Apache + Snort IDS en Ubuntu 22.04

> Guía paso a paso para instalar Apache, crear un servidor web personalizado y configurar Snort IDS para detectar tráfico ICMP y HTTP.

![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04-E95420?style=flat&logo=ubuntu&logoColor=white)
![Apache](https://img.shields.io/badge/Apache-2.x-D22128?style=flat&logo=apache&logoColor=white)
![Snort](https://img.shields.io/badge/Snort-IDS-CC0000?style=flat)

---

## 📋 Requisitos previos

| Requisito | Detalle |
|-----------|---------|
| Sistema operativo | Ubuntu 22.04 |
| Conectividad | Conexión a Internet |
| Permisos | Usuario con `sudo` |
| Software | Apache2, Snort IDS |

---

## Paso 1 — Actualizar el sistema

```bash
sudo apt update
sudo apt upgrade -y
sudo reboot
```

> [!NOTE]
> Después del reinicio, abre una nueva terminal antes de continuar.

---

## Paso 2 — Instalar Apache

```bash
sudo apt install -y apache2 apache2-utils
```

Iniciar y habilitar Apache para que arranque automáticamente:

```bash
sudo systemctl start apache2
sudo systemctl enable apache2
```

Verificar que está corriendo:

```bash
sudo systemctl status apache2
```

> [!TIP]
> Debes ver `● apache2.service` con el estado **active (running)** en verde.

Abre el navegador y visita `http://localhost`. Deberías ver la página por defecto de Apache **"It works!"**

---

## Paso 3 — Crear tu página web personalizada

### Conoce tu dirección IP

```bash
ip addr show
# o también:
ip route
```

Anota esa IP, la usarás más adelante.

### Crear la página HTML

```bash
sudo nano /var/www/html/index.html
```

Reemplaza el contenido existente con lo siguiente:

```html
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Servidor Monitoreado</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background: #1a1a2e;
            color: #eee;
            text-align: center;
            padding: 50px;
        }
        h1 { color: #e94560; }
        .badge {
            background: #16213e;
            padding: 15px 30px;
            border-radius: 8px;
            display: inline-block;
            margin-top: 20px;
        }
    </style>
</head>
<body>
    <h1>🛡️ Servidor Web Monitoreado con Snort</h1>
    <div class="badge">
        <p>Este servidor está bajo supervisión del IDS Snort</p>
        <p>Cualquier acceso queda registrado</p>
    </div>
</body>
</html>
```

Guarda con `Ctrl+O` → `Enter` → `Ctrl+X`.

### Verificar la página

```bash
curl http://localhost
```

También puedes acceder desde el navegador en `http://localhost` o `http://TU_IP`.

---

## Paso 4 — Instalar Snort

### Instalar dependencias

```bash
sudo apt install -y build-essential libpcap-dev libpcre3-dev \
    libdumbnet-dev zlib1g-dev bison flex
```

> [!WARNING]
> Si el comando anterior falla (error de mirror colombiano), ejecuta estos pasos primero:
>
> ```bash
> # 1. Reemplazar mirror colombiano por el principal
> sudo sed -i 's/co.archive.ubuntu.com/archive.ubuntu.com/g' /etc/apt/sources.list
>
> # 2. Actualizar repositorios
> sudo apt update
>
> # 3. Instalar dependencias
> sudo apt install -y build-essential libpcap-dev libpcre3-dev \
>     libdumbnet-dev zlib1g-dev bison flex
> ```

### Instalar Snort

Durante la instalación se pedirá el rango de red (usa `ip route` para consultarlo):

```bash
sudo apt install -y snort
```

Verificar instalación:

```bash
snort --version
```

Debe mostrar la versión de Snort instalada correctamente.

---

## Paso 5 — Configurar Snort

### Editar la configuración principal

```bash
sudo nano /etc/snort/snort.conf
```

Busca con `Ctrl+W` la línea `ipvar HOME_NET` y ajusta tu red:

```conf
ipvar HOME_NET 192.168.1.0/24    # ← cambia por tu red real
ipvar EXTERNAL_NET !$HOME_NET
```

Guarda con `Ctrl+O` → `Enter` → `Ctrl+X`.

### Crear el archivo de reglas personalizadas

```bash
sudo nano /etc/snort/rules/local.rules
```

Pega las siguientes reglas:

```snort
# ==========================================
# REGLAS PERSONALIZADAS - Actividad Snort
# ==========================================

# Detectar PING (ICMP Echo Request)
alert icmp any any -> $HOME_NET any (msg:"[PING] ICMP Echo Request detectado"; itype:8; icode:0; sid:1000001; rev:1;)

# Detectar respuesta de PING (ICMP Echo Reply)
alert icmp $HOME_NET any -> any any (msg:"[PING] ICMP Echo Reply enviado"; itype:0; icode:0; sid:1000002; rev:1;)

# Detectar conexión HTTP al Apache
alert tcp any any -> $HOME_NET 80 (msg:"[HTTP] Conexion al servidor web Apache"; flow:to_server,established; sid:1000003; rev:1;)

# Detectar petición GET al servidor web
alert tcp any any -> $HOME_NET 80 (msg:"[HTTP] Peticion GET detectada"; content:"GET"; http_method; sid:1000004; rev:1;)
```

Guarda con `Ctrl+O` → `Enter` → `Ctrl+X`.

### Escaneo de puertos, fuerza bruta SSH e inyección SQL básica sobre tráfico HTTP

```bash


# Detectar escaneo de puertos
alert tcp any any -> $HOME_NET any (msg:"Port Scan Detectado"; flags:S; threshold: type threshold, track by_src, count 20, seconds 3; sid:1000005; rev:1;)

# Detectar ataque de fuerza bruta SSH
alert tcp any any -> $HOME_NET 22 (msg:"Fuerza Bruta SSH"; threshold: type threshold, track by_src, count 5, seconds 60; sid:1000006; rev:1;)

# Detectar SQL Injection basico
alert tcp any any -> $HOME_NET 80 (msg:"Posible SQL Injection"; content:"SELECT"; nocase; sid:1000007; rev:1;)




```

### Verificar que `snort.conf` incluye las reglas locales

```bash
sudo nano /etc/snort/snort.conf
```

Busca con `Ctrl+W` la línea `include $RULE_PATH/local.rules`. Asegúrate de que **no tenga `#` al inicio**. Si no existe, agrégala al final del archivo:

```conf
include $RULE_PATH/local.rules
```

---

## Paso 6 — Validar la configuración

```bash
sudo snort -c /etc/snort/snort.conf -T
```

Al final debe aparecer:

```
Snort successfully validated the configuration!
```

---

## Paso 7 — Ejecutar Snort

Primero identifica tu interfaz de red exacta:

```bash
ip addr show
```

> [!NOTE]
> Los nombres de interfaz más comunes son: `ens33`, `eth0`, `enp0s3`.

Abre una terminal y ejecuta Snort en modo monitoreo activo:

```bash
sudo snort -c /etc/snort/snort.conf -i ens33 -A console -l /var/log/snort
```

La opción `-A console` muestra las alertas en tiempo real en pantalla.

---

## Paso 8 — Generar tráfico de prueba

Abre una **segunda terminal** y ejecuta las siguientes pruebas:

### Prueba 1 — Ping a la máquina

```bash
ping localhost
# o usa tu IP:
ping 192.168.1.XX
```

### Prueba 2 — Acceso al servidor web por consola

```bash
curl http://localhost
# o:
curl http://192.168.1.XX
```

### Prueba 3 — Acceso desde navegador

Abre el navegador y visita `http://localhost` o `http://TU_IP`.

---

## Paso 9 — Ver las alertas

En la terminal donde corre Snort verás alertas en tiempo real similares a estas:

```
08/24-10:15:32.123456  [**] [1:1000001:1] [PING] ICMP Echo Request detectado [**]
[Priority: 0] {ICMP} 127.0.0.1 -> 127.0.0.1

08/24-10:15:45.654321  [**] [1:1000003:1] [HTTP] Conexion al servidor web Apache [**]
[Priority: 0] {TCP} 127.0.0.1:55234 -> 127.0.0.1:80
```

### Ver el archivo de log guardado

```bash
sudo cat /var/log/snort/alert
```

### Monitorear en tiempo real desde otra terminal

```bash
sudo tail -f /var/log/snort/alert
```

---

## ✅ Resumen

| Paso | Acción |
|------|--------|
| 1 | Actualizar el sistema |
| 2 | Instalar y habilitar Apache |
| 3 | Crear página web personalizada |
| 4 | Instalar Snort y dependencias |
| 5 | Configurar reglas ICMP y HTTP |
| 6 | Validar configuración |
| 7 | Ejecutar Snort en modo consola |
| 8 | Generar tráfico de prueba |
| 9 | Verificar alertas en tiempo real |
