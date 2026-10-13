# Práctica de Redes: VM con proxy 3proxy

*Configuración de la VM, el proxy 3proxy y el firewall ufw*

## 1. Objetivo

Montar una máquina virtual con Ubuntu que funcione como **servidor proxy** usando **3proxy**, capaz de bloquear sitios web específicos y permitir el resto. La VM se administra por SSH desde la terminal de la computadora real (host) y los clientes (el propio host u otras VMs) se configuran para salir a internet a través del proxy.

El documento cubre la VM, el proxy, el firewall con ufw, cómo probarlo y qué cambiar cuando la VM cambia de red.

## 2. Conceptos clave

| Concepto | Qué es |
| --- | --- |
| **Proxy** | Servidor intermediario: el cliente le pide una página al proxy y este la solicita por él. Permite filtrar, bloquear y registrar el tráfico. |
| **3proxy** | Software ligero de proxy para Linux/Windows. Soporta HTTP, SOCKS, listas de control de acceso (ACL) y registros. |
| **Adaptador puente (bridged)** | La VM usa la tarjeta de red real del host y obtiene una IP en la misma red local, como si fuera otro equipo físico. |
| **SSH** | Protocolo para administrar un equipo remoto por terminal de forma cifrada (puerto 22). |
| **Puerto 3128** | Puerto en el que escucha el proxy (es el puerto tradicional de proxies HTTP). |

## 3. Crear la máquina virtual

1. Instalar Ubuntu en VirtualBox. Puede ser cualquier versión, incluso **Ubuntu Server** (sin interfaz gráfica), que consume menos recursos.
2. Asignar **2 GB de RAM** y **1 CPU**. No se necesita más para esta práctica.
3. Configurar el adaptador de red como **Adaptador puente**: en VirtualBox, *Configuración de la VM > Red > Adaptador 1 > Conectado a: Adaptador puente*. Así la VM comparte la tarjeta de red del host y se conecta a la misma red.

## 4. Identificar la configuración de red de la VM

Abrir la terminal de Ubuntu y ejecutar:

```bash
hostname -I
ip -4 addr
ip -4 route
```

| Comando | Para qué sirve |
| --- | --- |
| `hostname -I` | Muestra todas las direcciones IP asignadas a la VM. |
| `ip -4 addr` | Muestra las interfaces con su IP y su máscara de subred (por ejemplo `192.168.100.25/24`; el `/24` equivale a la máscara 255.255.255.0). |
| `ip -4 route` | Muestra la tabla de rutas: la red a la que pertenece la VM y la puerta de enlace. También permite identificar el nombre de la interfaz usada (aquí `enp0s3`). |

Anotar la **IP de la VM**, porque se usará en todos los pasos siguientes (en este documento se escribe como `<IP_VM>`).

## 5. Instalar y comprobar SSH

Instalar SSH permite administrar la VM desde la terminal del host, que es mucho más cómodo que usar la ventana de VirtualBox (se puede copiar y pegar).

```bash
sudo apt update
sudo apt install openssh-server -y
sudo systemctl enable --now ssh
```

| Comando | Para qué sirve |
| --- | --- |
| `sudo apt update` | Actualiza la lista de paquetes disponibles. |
| `sudo apt install openssh-server -y` | Instala el servidor SSH. `-y` responde "sí" automáticamente a las confirmaciones. |
| `sudo systemctl enable --now ssh` | `enable` hace que el servicio inicie con el sistema y `--now` lo inicia en este momento. |

**Comprobar que SSH funciona:**

```bash
sudo systemctl status ssh
sudo ss -tlnp | grep ':22'
```

| Comando | Para qué sirve |
| --- | --- |
| `systemctl status ssh` | Muestra si el servicio está activo (`active (running)`). |
| `ss -tlnp` | Lista los puertos TCP en escucha (`-t` TCP, `-l` en escucha, `-n` números en vez de nombres, `-p` proceso). Con `grep ':22'` se filtra solo el puerto 22. |

### Conectarse desde el host (Windows)

Desde PowerShell o CMD en el equipo real:

```bash
ssh USUARIO_DE_LINUX@<IP_VM>
```

Una vez dentro, comprobar que se está en la VM con:

```bash
whoami
```

Muestra el usuario con el que se inició sesión. Para salir de la sesión SSH **sin apagar la máquina**, usar `exit`.

## 6. Instalar 3proxy

Hay varias formas de instalarlo (ver la página oficial de 3proxy o su GitHub). Esta es la más sencilla: descargar el paquete `.deb` desde las *releases* e instalarlo.

```bash
sudo apt install -y wget ca-certificates
cd /tmp
wget https://github.com/3proxy/3proxy/releases/download/0.9.7/3proxy-0.9.7.x86_64.deb
sudo apt install -y ./3proxy-0.9.7.x86_64.deb
```

| Comando | Para qué sirve |
| --- | --- |
| `apt install -y wget ca-certificates` | `wget` descarga archivos de internet; `ca-certificates` permite validar conexiones HTTPS. |
| `cd /tmp` | Entra a la carpeta temporal para descargar ahí el instalador. |
| `wget <URL>` | Descarga el paquete `.deb` de 3proxy desde GitHub. |
| `apt install -y ./archivo.deb` | Instala el paquete local (el `./` indica que es un archivo, no un paquete del repositorio) y resuelve dependencias. |

**Verificar que está instalado:**

```bash
systemctl cat 3proxy
```

Muestra el archivo de servicio (*unit*) de 3proxy. Si lo muestra, el paquete se instaló y quedó registrado en systemd.

## 7. Configurar 3proxy

El archivo de configuración es `/etc/3proxy/3proxy.cfg`. Antes de modificarlo conviene **hacer una copia de respaldo**.

### Opción A: con comandos

```bash
sudo cp /etc/3proxy/3proxy.cfg /etc/3proxy/3proxy.cfg.bak
sudo tee /etc/3proxy/3proxy.cfg > /dev/null <<'EOF'
log /var/log/3proxy/3proxy.log D
logformat "- +_L%t.%. %N.%p %E %U %C:%c %R:%r %O %I %h %T"
rotate 30
auth iponly

# Bloqueo de sitios (los deny van primero)
deny * * *.facebook.com,facebook.com
deny * * *.tiktok.com,tiktok.com
#deny * * *.youtube.com,youtube.com

# Bloqueos por IP
#deny * * 203.0.113.25
#deny * * 198.51.100.0/24

# Permitir desde la propia VM
allow * 127.0.0.1

# Permitir clientes de la red local
allow * 192.168.100.0/24

# Denegar el resto
deny *

EOF
```

| Comando | Para qué sirve |
| --- | --- |
| `cp ... .bak` | Respaldo del archivo original, por si hay que volver atrás. |
| `tee /etc/3proxy/3proxy.cfg` | Escribe el texto recibido en el archivo (se usa `tee` con `sudo` porque la redirección `>` normal no heredaría los permisos de administrador). |
| `> /dev/null` | Evita que `tee` imprima el contenido en pantalla. |
| `<<'EOF' ... EOF` | *Here-document*: todo lo que está entre ambas marcas EOF se envía como texto al comando. Las comillas en `'EOF'` evitan que el shell interprete símbolos como `$`. |

### Opción B: con nano (más cómodo para editar)

```bash
sudo nano /etc/3proxy/3proxy.cfg
```

Editar lo necesario o pegar el contenido de la configuración mostrada arriba. Para guardar y salir:

1. `Ctrl + O` (guardar)
2. `Enter` (confirmar el nombre del archivo)
3. `Ctrl + X` (salir)

**Verificar que la configuración quedó escrita:**

```bash
sudo cat /etc/3proxy/3proxy.cfg
```

### Explicación de cada línea de la configuración

| Línea | Significado |
| --- | --- |
| `log /var/log/3proxy/3proxy.log D` | Define el archivo donde se guardan los registros de actividad. La opción final controla el manejo del archivo por fecha (ver documentación de 3proxy para detalles). |
| `logformat "..."` | Define qué datos se registran por cada conexión (fecha y hora, usuario, IP del cliente, destino, bytes enviados y recibidos, etc.). |
| `rotate 30` | Rotación de logs: conserva 30 archivos antiguos para que el disco no se llene. |
| `auth iponly` | Autenticación solo por dirección IP del cliente, sin usuario ni contraseña. El control de quién puede usar el proxy se hace con las reglas `allow`/`deny`. |
| `deny * * *.facebook.com,facebook.com` | Bloquea el sitio y todos sus subdominios. Formato de la regla: `deny <usuarios> <origen> <destino>`; los `*` significan "cualquiera". |
| `deny * * *.tiktok.com,tiktok.com` | Igual que la anterior, para TikTok. |
| `#deny * * *.youtube.com,youtube.com` | Regla **comentada** (`#`): no tiene efecto. Quitar el `#` para bloquear YouTube. |
| `#deny * * 203.0.113.25` y `#deny * * 198.51.100.0/24` | Ejemplos comentados de bloqueo por IP: una IP individual y un rango completo (notación CIDR). |
| `allow * 127.0.0.1` | Permite el uso del proxy desde la propia VM (dirección *loopback*). |
| `allow * 192.168.100.0/24` | Permite a todos los equipos de la red local `192.168.100.0/24` usar el proxy. **Ajustar a la red real que se obtuvo con `ip -4 route`.** |
| `deny *` | Regla final: niega todo lo que no coincidió con un `allow` anterior. |

> **Importante: el orden importa.** 3proxy evalúa las reglas de arriba hacia abajo y gana la primera que coincide. Por eso los `deny` de sitios específicos van **antes** de los `allow`, y el `deny *` general va al final.

> **Nota:** el servicio que escucha en el puerto 3128 se define con una línea `proxy -p3128`, que va **después** de las reglas de acceso. Si al comprobar el puerto (paso 8) no aparece nada escuchando en el 3128, revisar que el archivo tenga esa línea al final.

### Permitir solo ciertas IPs de la red

Con `allow * 192.168.100.0/24` puede usar el proxy **toda** la red. Para que lo usen solo equipos concretos, esa línea se reemplaza por una regla `allow` por cada equipo. Lo que no esté en la lista cae en el `deny *` final y se rechaza.

**Paso 1.** Identificar la IP de cada cliente permitido:

- Windows: `ipconfig` y buscar *Dirección IPv4*.
- Linux: `ip -4 addr`.

**Paso 2.** Editar la configuración con `sudo nano /etc/3proxy/3proxy.cfg` y dejar el final del archivo así:

```
# Permitir desde la propia VM
allow * 127.0.0.1

# Permitir solo equipos específicos (uno por línea)
allow * 192.168.100.10
allow * 192.168.100.25

# Denegar el resto
deny *
```

**Paso 3.** Reiniciar el servicio para que lea los cambios:

```bash
sudo systemctl restart 3proxy
```

**Otras formas de escribir la regla:**

| Quiero permitir | Regla |
| --- | --- |
| Un equipo | `allow * 192.168.100.10` |
| Varios equipos en una sola línea | `allow * 192.168.100.10,192.168.100.25,192.168.100.30` |
| Un bloque pequeño de IPs (CIDR) | `allow * 192.168.100.32/29` (8 direcciones: de la .32 a la .39) |
| Un rango de IPs | `allow * 192.168.100.30-192.168.100.40` |
| Toda la red **excepto** un equipo | `deny * 192.168.100.50` y **debajo** `allow * 192.168.100.0/24` |

**Cómo verificar:**

- Desde un equipo permitido, `curl -x http://<IP_VM>:3128 -I http://example.com` responde `200 OK`.
- Desde un equipo que no está en la lista, el proxy rechaza la conexión.
- Para ver quién entró y a dónde, revisar el registro: `sudo tail -n 20 /var/log/3proxy/3proxy.log`.

> **Nota 1: el orden decide.** 3proxy lee de arriba hacia abajo y gana la primera regla que coincide. Por eso, en la variante "toda la red excepto un equipo", el `deny` de esa IP debe ir **antes** del `allow` de la red; si va después, el `allow` ya lo habría dejado pasar.

> **Nota 2: lista blanca vs. lista negra.** Permitir solo los equipos que enumeras (lista blanca) es más seguro que permitir toda la red y bloquear algunos (lista negra), porque cualquier equipo nuevo queda bloqueado por defecto. La lista negra es más cómoda cuando son muchos clientes y pocos excluidos.

> **Nota 3: la IP del cliente debe ser estable.** En modo puente las IPs las reparte el DHCP del router y pueden cambiar. Si la IP de un cliente cambia, el proxy lo bloqueará aunque sea el mismo equipo. Para evitarlo, reservar la IP en el router (reserva DHCP) o poner una IP fija en el cliente.

> **Nota 4: los sitios bloqueados siguen bloqueados.** Los `deny` de Facebook y TikTok van antes de los `allow`, así que aplican incluso a las IPs permitidas. Las reglas de IP controlan *quién* usa el proxy; las de sitios controlan *a dónde* puede ir.

## 8. Iniciar el servicio y comprobar que funciona

```bash
sudo systemctl enable --now 3proxy
sudo systemctl restart 3proxy
systemctl status 3proxy --no-pager
sudo ss -tlnp | grep 3128
```

| Comando | Para qué sirve |
| --- | --- |
| `systemctl enable --now 3proxy` | Activa 3proxy para que inicie con el sistema y lo arranca ahora. |
| `systemctl restart 3proxy` | Reinicia el servicio para que lea la configuración nueva. **Hacerlo siempre después de editar el `.cfg`.** |
| `systemctl status 3proxy --no-pager` | Muestra el estado del servicio; `--no-pager` imprime todo de una vez sin abrir el paginador. Debe decir `active (running)`. |
| `ss -tlnp \| grep 3128` | Confirma que hay un proceso escuchando en el puerto 3128. |

## 9. Probar el proxy desde la propia VM

Instalar `curl` (cliente para hacer peticiones web desde la terminal; es más cómodo que `wget` para pruebas):

```bash
sudo apt install -y curl
```

Probar usando la IP de la VM:

```bash
curl -x http://<IP_VM>:3128 -I http://example.com
curl -x http://<IP_VM>:3128 -I http://www.facebook.com
```

O usando la dirección *loopback* (`127.0.0.1`, la VM hablando consigo misma):

```bash
curl -x http://127.0.0.1:3128 -I http://example.com
curl -x http://127.0.0.1:3128 -I http://www.facebook.com
```

| Opción | Significado |
| --- | --- |
| `-x http://IP:PUERTO` | Indica a curl que haga la petición **a través del proxy** indicado. |
| `-I` | Pide solo las cabeceras de la respuesta (rápido y suficiente para probar). |

**Resultado esperado:**

- `example.com` responde con `HTTP/1.1 200 OK` (el sitio está permitido).
- `facebook.com` es rechazado por el proxy (error o código distinto de 200), porque coincide con una regla `deny`.

Si ambos pasan o ambos fallan, revisar el orden de las reglas y la red permitida en el `allow`.

## 10. Configurar el proxy en un cliente (ejemplo: Windows 11)

1. Abrir **Configuración > Red e Internet > Proxy**.
2. Bajar a la sección **Configuración manual del proxy** y pulsar *Configurar*.
3. Activar **Usar un servidor proxy**.
4. En *Dirección IP del proxy*, escribir la **IP de la máquina virtual**.
5. En *Puerto*, escribir **3128**.
6. Guardar.

Para **quitar el proxy** en el cliente, basta con desactivar la opción o borrar la IP.

## 11. Comandos útiles

| Comando | Para qué sirve |
| --- | --- |
| `exit` | Cierra la sesión SSH sin apagar la máquina virtual. |
| `sudo systemctl enable --now 3proxy` | Hace que 3proxy inicie automáticamente cuando se enciende la VM y lo arranca. |
| `sudo systemctl restart 3proxy` | Reinicia el servicio (necesario tras cada cambio de configuración). |
| `systemctl status 3proxy --no-pager` | Verifica que el servicio esté activo. |
| `sudo cat /etc/3proxy/3proxy.cfg` | Muestra la configuración actual. |
| `sudo cp /etc/3proxy/3proxy.cfg.bak /etc/3proxy/3proxy.cfg` | Restaura la configuración original desde el respaldo. |

**Cuando la VM se apaga y se vuelve a encender**, si 3proxy no está corriendo:

```bash
sudo systemctl enable --now 3proxy
sudo systemctl restart 3proxy
systemctl status 3proxy --no-pager
```

## 12. Posibles problemas

| Síntoma | Qué revisar |
| --- | --- |
| No conecta por SSH | Que la VM esté en adaptador puente, que la IP sea la correcta y que `ssh` esté activo (`systemctl status ssh`). |
| El cliente no navega con el proxy activado | Que el puerto 3128 esté en escucha (`ss -tlnp`), que la IP del cliente esté dentro de la red del `allow` y que 3proxy esté `active (running)`. |
| Un sitio bloqueado sí abre | Orden de las reglas (los `deny` antes de los `allow`), que se haya reiniciado el servicio y que el cliente realmente esté usando el proxy. |
| El servicio no inicia tras editar | Error de sintaxis en el `.cfg`: revisar el archivo o restaurar el respaldo `.bak`. |
| La IP de la VM cambió | En modo puente la IP puede cambiar por DHCP; volver a revisarla con `hostname -I` y actualizar el cliente. |

## 13. Firewall con ufw

**ufw** (*Uncomplicated Firewall*) es una herramienta sencilla para administrar el firewall de Linux (netfilter). Trabaja por encima de iptables/nftables, así que no hace falta escribir reglas complejas.

### Configuración básica

```bash
sudo apt install -y ufw
sudo ufw allow 22/tcp
sudo ufw allow from <RED_LOCAL>/24 to any port 3128 proto tcp
sudo ufw enable
sudo ufw status verbose
```

| Comando | Para qué sirve |
| --- | --- |
| `apt install -y ufw` | Instala ufw (en Ubuntu suele venir instalado pero desactivado). |
| `ufw allow 22/tcp` | Permite el puerto 22 (SSH) por TCP. Se hace **antes** de activar el firewall para que no se corte la conexión SSH. |
| `ufw allow from <RED_LOCAL>/24 to any port 3128 proto tcp` | Permite que los equipos de esa red lleguen al puerto 3128 (el proxy). `from` indica el origen, `to any` cualquier IP de la VM, `port 3128` el puerto y `proto tcp` el protocolo. Ejemplo de red: `192.168.100.0/24`. |
| `ufw enable` | Activa el firewall y hace que inicie con el sistema. |
| `ufw status verbose` | Muestra el estado, la política por defecto y todas las reglas. |

> **Nota 1: política por defecto.** Al activarse, ufw **bloquea todo lo que entra** (`deny incoming`) y **permite todo lo que sale** (`allow outgoing`). Por eso hay que abrir explícitamente el 22 (SSH) y el 3128 (proxy): lo que no está permitido, se descarta.

> **Nota 2: no perder la conexión SSH.** Al ejecutar `ufw enable`, ufw avisa que puede interrumpir conexiones SSH y pide confirmar con `y`. Como el puerto 22 ya está permitido, la sesión sigue viva. Si se activara sin esa regla, la sesión se cortaría y habría que entrar por la ventana de VirtualBox.

> **Nota 3: lo que sale no necesita regla.** El proxy sale a internet a través de la política `allow outgoing`, por eso no hay que abrir nada para eso. Las reglas de entrada solo controlan quién puede llegar a la VM.

### Ver y gestionar las reglas

```bash
sudo ufw status numbered
sudo ufw delete <N>
sudo ufw disable
sudo ufw enable
```

| Comando | Para qué sirve |
| --- | --- |
| `ufw status numbered` | Lista las reglas con un número a la izquierda. |
| `ufw delete <N>` | Borra la regla número `N`. Se borra una a la vez y **los números cambian** después de cada borrado, así que conviene volver a listar. |
| `ufw disable` | Apaga el firewall (las reglas se conservan). |
| `ufw enable` | Lo vuelve a encender con las reglas guardadas. |

### Limitar el proxy a IPs específicas también en ufw

Igual que en 3proxy, ufw puede aceptar solo equipos concretos. Así se tienen dos capas: el firewall corta lo que no es de confianza antes de que llegue al proxy, y 3proxy decide qué puede hacer cada equipo permitido.

```bash
sudo ufw allow from 192.168.100.10 to any port 3128 proto tcp
sudo ufw allow from 192.168.100.25 to any port 3128 proto tcp
```

> **Nota 4: capas distintas.** ufw trabaja a nivel de **red y puerto** (¿puede esta IP llegar a este puerto?). 3proxy trabaja a nivel de **aplicación** (¿puede esta IP navegar y a qué sitios?). Para que un equipo use el proxy debe pasar por las dos.

## 14. Probar el firewall

Probar el firewall es comprobar dos cosas: que **lo permitido pasa** (SSH y proxy) y que **lo no permitido se bloquea**. Las pruebas se hacen desde la VM y desde un cliente.

| # | Prueba | Dónde | Comando | Resultado esperado |
| --- | --- | --- | --- | --- |
| 1 | Estado del firewall | VM | `sudo ufw status verbose` | `Status: active`, política `deny (incoming), allow (outgoing)` y las reglas del 22 y del 3128. |
| 2 | Servicios en escucha | VM | `sudo ss -tlnp` | Puertos 22 (ssh) y 3128 (3proxy) en estado `LISTEN`. |
| 3 | Puerto del proxy abierto | Cliente Windows (PowerShell) | `Test-NetConnection <IP_VM> -Port 3128` | `TcpTestSucceeded : True` |
| 4 | Puerto no permitido cerrado | Cliente Windows (PowerShell) | `Test-NetConnection <IP_VM> -Port 8080` | `TcpTestSucceeded : False` (tarda unos segundos en fallar). |
| 5 | Proxy filtrando | Cliente permitido | `curl -x http://<IP_VM>:3128 -I http://example.com` y lo mismo con `www.facebook.com` | example.com responde `200 OK`; facebook es rechazado. |
| 6 | Cliente no permitido | Equipo cuya IP no está permitida | El mismo `curl` de la prueba 5 | Se queda esperando y termina con *timeout*: ufw descarta los paquetes. |
| 7 | Escaneo de puertos (opcional) | Otro equipo con nmap | `nmap -p 22,3128,8080 <IP_VM>` | 22 y 3128 `open`; 8080 `filtered`. |

En Linux, la prueba 3 y la 4 equivalen a `nc -zv <IP_VM> 3128` (conecta o expira).

### Prueba de cortar el acceso

Sirve para demostrar que la regla del firewall es la que deja pasar al proxy:

1. En la VM, listar las reglas: `sudo ufw status numbered`.
2. Borrar la regla del puerto 3128: `sudo ufw delete <N>`.
3. Desde el cliente repetir la prueba 3 o la 5: ahora **falla**.
4. Volver a agregar la regla: `sudo ufw allow from <RED_LOCAL>/24 to any port 3128 proto tcp`.
5. Repetir la prueba: vuelve a funcionar.

### Ver los bloqueos en el registro

```bash
sudo ufw logging on
sudo tail -f /var/log/ufw.log
```

Cada paquete bloqueado aparece como una línea `[UFW BLOCK]` con la IP de origen (`SRC`), el destino (`DST`) y el puerto (`DPT`). Mientras el comando corre, hacer una prueba desde el cliente y ver cómo aparece la línea. Si el archivo no existe, usar `sudo journalctl -k | grep UFW`. Salir con `Ctrl + C`.

> **Nota 1: tiempo agotado vs. rechazo.** Cuando **ufw** bloquea, descarta el paquete sin responder y el cliente espera hasta agotar el tiempo. Cuando **3proxy** bloquea, responde rápido con un error o cierra la conexión. Esa diferencia indica qué capa bloqueó. Si se prefiere que ufw responda de inmediato, se usa `reject` en lugar de `deny`.

> **Nota 2: el ping sí responde.** ufw permite por defecto las peticiones ICMP (ping), así que `ping <IP_VM>` funciona aunque el firewall esté activo. Que haga ping no significa que los puertos estén abiertos: para eso sirven las pruebas 3, 4 y 7.

> **Nota 3: con y sin firewall.** Si se apaga ufw (`sudo ufw disable`) y se prueba el puerto 8080, la conexión se rechaza al instante porque nadie escucha ahí; con ufw activo el mismo puerto aparece como `filtered` y tarda en fallar. Esa comparación muestra el efecto del firewall. Después hay que volver a activarlo con `sudo ufw enable`.

> **Nota 4: no cerrar la sesión SSH hasta probar.** Después de cambiar reglas del puerto 22, abrir una segunda terminal y conectarse por SSH antes de cerrar la primera. Si algo salió mal, la sesión original sigue abierta para corregirlo.

## 15. Qué cambiar al cambiar de red

Con el adaptador puente, la VM recibe su IP del DHCP de la red a la que esté conectado el host. Si la red cambia, cambian la IP de la VM y la subred, y hay que ajustar el proxy, el firewall y los clientes.

| Elemento | Dónde está | ¿Cambia? | Qué hacer |
| --- | --- | --- | --- |
| Adaptador puente de VirtualBox | Configuración de la VM > Red > Adaptador 1 > Nombre | Si cambias de Wi-Fi a cable (o al revés) | Elegir la tarjeta de red que el host esté usando ahora. |
| IP de la VM | `hostname -I` | Sí | Anotar la nueva IP: se usa en SSH y en el proxy de los clientes. |
| Subred permitida en 3proxy | `/etc/3proxy/3proxy.cfg`, línea `allow * <red>/24` | Sí | Cambiarla por la subred nueva y reiniciar 3proxy. |
| IPs específicas en 3proxy | Mismo archivo | Sí | Poner las IPs que tengan los clientes en la red nueva. |
| Regla de ufw del puerto 3128 | `sudo ufw status numbered` | Sí | Borrar la regla de la red anterior y crear la nueva. |
| Regla de ufw del puerto 22 (SSH) | `sudo ufw status numbered` | Solo si la limitaste a una IP o red | Si es `allow 22/tcp` general, no cambia. |
| Proxy en los clientes (Windows) | Configuración > Red e Internet > Proxy | Sí | Escribir la nueva IP de la VM. |
| Conexión SSH | Terminal del host | Sí | `ssh USUARIO@IP_NUEVA` |
| `allow * 127.0.0.1`, puerto 3128 y los `deny` de sitios | `3proxy.cfg` | No | Dejarlos igual. |

### Pasos

**1. Ver la nueva configuración de red.** Como la IP cambió, no se puede entrar por SSH todavía: abrir la terminal desde la ventana de VirtualBox y ejecutar:

```bash
hostname -I
ip -4 addr
ip -4 route
```

**2. Ajustar 3proxy.** Ejemplo si la nueva red es `10.0.5.0/24`:

```bash
sudo nano /etc/3proxy/3proxy.cfg
# cambiar: allow * 192.168.100.0/24  ->  allow * 10.0.5.0/24
sudo systemctl restart 3proxy
systemctl status 3proxy --no-pager
```

**3. Ajustar ufw.** Borrar la regla vieja del 3128 (no la del 22) y crear la nueva:

```bash
sudo ufw status numbered
sudo ufw delete <N>
sudo ufw allow from 10.0.5.0/24 to any port 3128 proto tcp
sudo ufw status verbose
```

**4. Ajustar los clientes.** Poner la nueva IP de la VM en la configuración del proxy y volver a conectar por SSH con la IP nueva.

**5. Repetir las pruebas** 1, 3 y 5 de la sección 14 para confirmar que todo quedó funcionando.

> **Nota 1: la máscara puede ser distinta.** No todas las redes usan `/24`. Si `ip -4 addr` muestra, por ejemplo, `/16` o `/23`, esa es la máscara que va en el `allow` de 3proxy y en la regla de ufw. Con una máscara demasiado chica, algunos clientes quedan fuera de la regla y se bloquean sin avisar.

> **Nota 2: menos cambios con IP estable.** Si el router lo permite, una reserva DHCP para la VM evita que su IP cambie dentro de la misma red. Al cambiar de red igual habrá que repetir los pasos.

> **Nota 3: algunas redes aíslan los equipos.** En redes de escuela, cafeterías o públicas, a veces los dispositivos no pueden verse entre sí o no se asigna IP a las VMs en modo puente. Si los clientes no alcanzan la VM aunque todo esté bien configurado, la alternativa es una red **solo-anfitrión** o **interna** de VirtualBox, que no depende de la red física.

> **Nota 4: si limitaste el SSH por IP.** Agregar primero la regla nueva del puerto 22 y probar la conexión, y después borrar la vieja. Si se hace al revés, la sesión SSH se corta.
