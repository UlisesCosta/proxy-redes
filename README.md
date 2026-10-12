Proxy 3proxy en una VM Debian (VirtualBox) con UFW y clientes Windows 11
Guía para replicar la práctica de proxy y firewall con 3proxy instalado en una máquina virtual Debian (VirtualBox, adaptador puente), usada por clientes Windows 11 en otros dispositivos. Incluye el procedimiento completo para cuando cambias de red.

Qué cubre y qué no. Cubre: navegar a través del proxy, bloquear dominios, permitir solo una red o IPs concretas, revisar logs y controlar el acceso con firewall (UFW). No cubre la prueba de caché (TCP_HIT): 3proxy no tiene caché web. Además, los mensajes y el formato del log no son los de Squid (TCP_DENIED/403, TCP_MISS, etc.), así que las capturas y resultados esperados de las guías hay que adaptarlos.

Índice
Arquitectura y datos que necesitas anotar
Crear la VM en VirtualBox
Instalar Debian
Primer arranque, IP y actualización
Instalar 3proxy
Configurar 3proxy
Iniciar el servicio y comprobar
Firewall UFW
Configurar los clientes Windows 11
Pruebas y revisión del log
Cuando cambias de red
IP fija para la VM
Apagar, encender y reactivar el servicio
Problemas frecuentes
Referencia rápida de comandos
1. Arquitectura y datos que necesitas anotar
 [Win11 compañero 1]  ┐
 [Win11 compañero 2]  ├──►  Red real (Wi-Fi/LAN)  ◄──►  [VM Debian "debproxy"]  ──►  Internet
 [Tu laptop (pruebas)]┘          adaptador puente        3proxy en el puerto 3128

La VM usa adaptador puente: tiene su propia IP en la red real, igual que cualquier otro dispositivo.
Los clientes configuran como proxy la IP de la VM y el puerto 3128.
El control de quién puede usar el proxy lo hacen dos capas: las reglas allow/deny de 3proxy y las reglas de UFW.
Llena esta tabla en cuanto la tengas. Estos valores cambian cuando cambias de red:

Dato	Cómo obtenerlo	Tu valor
IP de la VM	ip -4 -br addr	_____________
Subred de la red	ip -4 route show scope link	_____________
IP de tu laptop	ipconfig en PowerShell	_____________
Usuario de la VM	el que creaste (en esta guía, proxyadmin)	_____________
Puerta de enlace (solo si fijas la IP dentro de Debian)	ip route | grep default	_____________
En el resto del documento se usan estos marcadores. Sustitúyelos por tus valores:

<IP_VM>: la IP de la VM, por ejemplo 192.168.1.50.
<SUBRED>: la subred con máscara, por ejemplo 192.168.1.0/24.
Requisitos
VirtualBox instalado y la ISO netinst amd64 de Debian estable (esta guía se hizo con Debian 13 «Trixie»): https://www.debian.org/distrib/netinst
Conexión a Internet en el equipo anfitrión.
Un cliente Windows 11 por compañero (o tu propia laptop para probar).
Sobre el Debian de WSL. Si ya tienes Debian en WSL, no hay conflicto: son sistemas independientes, con su disco, IP y configuración propios. Para ahorrar RAM, puedes apagar WSL mientras trabajas con wsl --shutdown en PowerShell. VirtualBox y WSL2 pueden convivir, pero la VM puede ir algo más lenta de lo normal.

2. Crear la VM en VirtualBox
Nueva → nombre debproxy, tipo Linux, versión Debian (64-bit), y selecciona la ISO.
Marca Omitir instalación desatendida (Skip Unattended Installation). Así controlas qué se instala (sin escritorio, con servidor SSH).
Si no la omites, no uses daemon como nombre de usuario: es una cuenta de sistema que ya existe en Debian.
Hardware:
Recurso	Mínimo recomendado	Si tienes holgura
RAM	1024 MB	2048 MB
CPU	1	2
Disco (VDI dinámico, sin «Reservar completamente»)	10 GB	20 GB
3proxy consume del orden de 1 a 2 MB de RAM; lo que ocupa recursos es el sistema operativo, por eso no se instala escritorio.

Red (con la VM apagada): Configuración → Red → Adaptador 1 → Conectado a: Adaptador puente, y en Nombre elige la tarjeta física que estás usando (Wi-Fi o Ethernet).
Si la VM no obtiene IP, abre Avanzadas y cambia Modo promiscuo a Permitir todo.
3. Instalar Debian
Inicia la VM y elige Install (modo texto). Respuestas recomendadas:

Pantalla	Respuesta
Idioma, país, teclado	Español / México (o el tuyo)
Nombre de la máquina	debproxy (borra el texto largo que aparece por defecto)
Dominio	vacío
Contraseña de root	vacía (así se instala sudo y tu usuario recibe permisos)
Usuario	por ejemplo proxyadmin (no uses daemon) y su contraseña
Particionado	guiado, usar todo el disco, todos los archivos en una partición
Analizar medios adicionales	No (la ISO netinst descarga el resto de Internet)
País de la réplica	México (o el tuyo), servidor por defecto
Proxy HTTP del instalador	vacío
Encuesta de popularidad	No
Selección de software	Desmarcar «Entorno de escritorio Debian» y «GNOME». Marcar «Servidor SSH» y «Utilidades estándar del sistema» (barra espaciadora)
GRUB	instalar en el disco /dev/sda
Al terminar, el instalador pide reiniciar. Pulsa Continuar.

Si aparece el menú de GRUB con «Debian GNU/Linux», arrancó desde el disco: déjalo continuar.
Si aparece otra vez el menú azul del instalador, arrancó desde la ISO. No elijas nada: quita la ISO (Dispositivos → Unidades ópticas → Quitar disco de la unidad virtual) y reinicia (Máquina → Reiniciar).
Quitar la ISO no borra nada de la VM: la configuración vive en el disco virtual (.vdi).
4. Primer arranque, IP y actualización
Inicia sesión con tu usuario (al escribir la contraseña no se muestra nada; es normal). Luego:

ip -4 -br addr
ip -4 route show scope link

La primera línea muestra la IP de la VM (<IP_VM>), por ejemplo enp0s3 UP 192.168.1.50/24.
La segunda muestra la subred (<SUBRED>), por ejemplo 192.168.1.0/24 dev enp0s3 ....
Anótalas en la tabla de la sección 1. Después actualiza el sistema:

sudo apt update && sudo apt full-upgrade -y

Administrar la VM por SSH (opcional pero cómodo)
Desde PowerShell en tu laptop:

ssh proxyadmin@<IP_VM>

La primera vez pregunta si confías en el servidor: escribe yes. Así puedes copiar y pegar los comandos. Si tu laptop no alcanza la IP de la VM, usa la consola de VirtualBox.

¿Quieres que la IP no cambie? Por defecto la VM pide su IP por DHCP y puede recibir una distinta cada vez. Para fijarla, mira la sección 12.

5. Instalar 3proxy
El proyecto publica paquetes .deb para x86 de 64 bits en sus releases. La versión usada en esta guía es la 0.9.7.

sudo apt install -y wget ca-certificates
cd /tmp
wget https://github.com/3proxy/3proxy/releases/download/0.9.7/3proxy-0.9.7.x86_64.deb
sudo apt install -y ./3proxy-0.9.7.x86_64.deb

Si wget da 404, abre https://github.com/3proxy/3proxy/releases en el navegador, copia el enlace del archivo .x86_64.deb y úsalo.
Un aviso de apt sobre «descarga sin sandbox» es inofensivo.
No hace falta seguir en /tmp para los pasos siguientes: todos usan rutas absolutas. El .deb ya no se necesita (rm /tmp/3proxy-0.9.7.x86_64.deb).
Verificar cómo quedó el servicio
systemctl cat 3proxy

En la instalación documentada aquí aparece:

Línea	Valor	Qué significa
Environment=CONFIGFILE=	/etc/3proxy/3proxy.cfg	Esa es la ruta del archivo de configuración
Type=	simple	No agregues la línea daemon al .cfg
User= / Group=	root / proxy	El servicio puede escribir sus logs en /var/log/3proxy
Restart= / RestartSec=	on-failure / 60s	Si hay un error, reintenta cada 60 s
Si en tu equipo Type= fuera forking, agrega la línea daemon al principio del .cfg. Si la ruta del .cfg fuera otra, usa esa en la sección 6.

6. Configurar 3proxy
Respalda el archivo original y escribe el nuevo. Cambia 192.168.1.0/24 por tu <SUBRED>:

sudo cp /etc/3proxy/3proxy.cfg /etc/3proxy/3proxy.cfg.bak
sudo tee /etc/3proxy/3proxy.cfg > /dev/null <<'EOF'
log /var/log/3proxy/3proxy.log D
logformat "- +_L%t.%. %N.%p %E %U %C:%c %R:%r %O %I %h %T"
rotate 30
auth iponly

# Bloqueo de sitios (los deny van primero)
deny * * *.facebook.com,facebook.com
#deny * * *.youtube.com,youtube.com

# Solo la red de clase; el resto se niega
allow * 192.168.1.0/24
deny *

proxy -n -p3128
EOF

Para revisar el resultado (el archivo no es legible sin sudo, y eso es normal):

sudo cat /etc/3proxy/3proxy.cfg

Qué hace cada línea
Línea	Función
log ... D	Un archivo de log por día en /var/log/3proxy/ (la fecha se añade al nombre)
logformat ...	Formato del registro: hora, servicio, error, usuario, IP:puerto del cliente (%C:%c), destino (%R:%r), bytes, host y petición (%T)
rotate 30	Conserva 30 archivos de log
auth iponly	Identifica a los clientes por IP, sin usuario ni contraseña
deny * * *.facebook.com,facebook.com	Bloquea ese dominio y sus subdominios
allow * 192.168.1.0/24	Permite solo esa subred
deny *	Niega todo lo demás
proxy -n -p3128	Levanta el proxy HTTP/HTTPS en el puerto 3128 (-n desactiva NTLM)
El orden importa: las reglas se evalúan de arriba abajo y gana la primera que coincide. Por eso los deny específicos van antes del allow, y deny * va al final.

Facebook usa dominios adicionales (por ejemplo fbcdn.net), así que el bloqueo de facebook.com no es total. Para la práctica basta con ese dominio.

7. Iniciar el servicio y comprobar
sudo systemctl enable --now 3proxy
sudo systemctl restart 3proxy
systemctl status 3proxy --no-pager
sudo ss -tlnp | grep 3128

Resultado esperado:

status: Active: active (running) y enabled.
ss: una línea LISTEN 0.0.0.0:3128 con el proceso 3proxy.
Un aviso de «SysV script» al habilitar el servicio es normal.
Si falla, el motivo suele estar en el .cfg:

sudo journalctl -u 3proxy -n 30 --no-pager

Cada vez que edites el .cfg, ejecuta sudo systemctl restart 3proxy. No esperes al reinicio automático, que tarda 60 s.

Prueba local desde la propia VM
Instala curl (la instalación mínima de Debian no lo trae) y prueba con la IP de la VM, no con 127.0.0.1 (que está fuera de la subred permitida y tu deny * lo rechazaría):

sudo apt install -y curl
curl -x http://<IP_VM>:3128 -I http://example.com
curl -x http://<IP_VM>:3128 -I http://www.facebook.com

example.com → HTTP/1.1 200 o una redirección.
facebook.com → rechazo, normalmente 403 Forbidden. El texto exacto del mensaje de 3proxy no está confirmado: captura el que veas.
Alternativa sin instalar nada, con wget:

wget -e use_proxy=yes -e http_proxy=http://<IP_VM>:3128 -S --spider http://example.com

8. Firewall UFW
UFW limita quién puede llegar a la VM. Se aplica antes que las reglas de 3proxy: si UFW bloquea a un cliente, ni siquiera llega a 3proxy y no queda registrado en su log.

sudo apt install -y ufw
sudo ufw allow 22/tcp
sudo ufw allow from 192.168.1.0/24 to any port 3128 proto tcp
sudo ufw enable
sudo ufw status verbose

Cambia 192.168.1.0/24 por tu <SUBRED>.

Permite SSH (22/tcp) antes de ejecutar ufw enable. Si estás conectado por SSH y no lo haces, te quedas fuera de la VM. Si te pasa, entra por la consola de VirtualBox y ejecuta sudo ufw disable.

Notas:

Por defecto UFW niega lo entrante y permite lo saliente, así que la VM sigue teniendo Internet.
UFW responde al ping por defecto, lo que sirve para comprobar conectividad.
La regla de SSH 22/tcp sin restricción permite conectarte desde cualquier red. Si la limitaste a una subred, tendrás que actualizarla también al cambiar de red.
Si la VM es solo de pruebas, el riesgo de un error de firewall es bajo: lo peor que pasa es perder el acceso a la VM, no a tu equipo real. Se corrige desde la consola de VirtualBox.
Ver y gestionar las reglas
sudo ufw status numbered       # reglas con número
sudo ufw delete <N>            # borra la regla número N (uno a la vez; los números cambian)
sudo ufw disable               # apaga el firewall
sudo ufw enable                # lo vuelve a encender

9. Configurar los clientes Windows 11
En cada equipo: Configuración → Red e Internet → Proxy → Configuración manual del proxy → Configurar.

Usar un servidor proxy: Activado.
Dirección del proxy: <IP_VM>.
Puerto: 3128.
Guardar.
Antes de configurarlo, comprueba la conectividad desde PowerShell:

ping <IP_VM>

Si no responde, revisa la sección 12.

Probar solo con Firefox (opcional)
Si no quieres tocar el proxy de todo el sistema: Firefox → Configuración → General → Configuración de red → Configuración → Configuración manual del proxy (HTTP <IP_VM>, puerto 3128, y marca «Usar también este proxy para HTTPS»).

Activar o desactivar el proxy del sistema desde PowerShell (opcional)
# Desactivar (si te quedas sin navegación porque la VM está apagada)
Set-ItemProperty -Path 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings' -Name ProxyEnable -Value 0

# Activar con tu IP
Set-ItemProperty -Path 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings' -Name ProxyServer -Value '<IP_VM>:3128'
Set-ItemProperty -Path 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings' -Name ProxyEnable -Value 1

Cierra y vuelve a abrir el navegador para que lo tome.

Si apagas la VM con el proxy activado en Windows, tu navegador deja de cargar páginas. Desactiva el proxy al terminar.

10. Pruebas y revisión del log
Desde un Windows 11 (o tu laptop)
Usa curl.exe (con la extensión, para no llamar al alias de PowerShell):

curl.exe -x http://<IP_VM>:3128 -I http://example.com
curl.exe -x http://<IP_VM>:3128 -I http://www.facebook.com
curl.exe -x http://<IP_VM>:3128 -I https://www.facebook.com

Las dos primeras comprueban el bloqueo en HTTP; la tercera lo comprueba en HTTPS (túnel CONNECT) y debe fallar.
En el navegador, un sitio HTTPS bloqueado probablemente muestre un error de conexión o de túnel en lugar de una página de aviso.
Ver el log en vivo en la VM
sudo ls -l /var/log/3proxy
sudo sh -c 'tail -n 20 -f /var/log/3proxy/*'

(Con sudo sh -c el comodín * lo expande root, que sí puede leer la carpeta. Detén con Ctrl+C.)

En el log verás la IP del cliente (campo %C), la hora y la URL pedida. Sirve para tres cosas:

Comprobar quién navega y qué pide.
Evidenciar los sitios bloqueados.
Averiguar la IP de cada compañero para el ejercicio de «solo ciertas IPs».
Ejercicio: permitir solo ciertas IPs
Reemplaza la línea allow de la subred por las IPs permitidas, y reinicia:

sudo nano /etc/3proxy/3proxy.cfg
#   allow * 192.168.1.30,192.168.1.31
sudo systemctl restart 3proxy

Un tercer equipo que intente navegar quedará denegado. Si también quieres que UFW lo corte antes, ajusta su regla (sección 8).

Ejercicio: quitar o añadir bloqueos
Para bloquear YouTube, quita el # de la línea deny ... youtube.com y reinicia el servicio.
Para desbloquear Facebook, comenta su línea deny con # y reinicia.
Capturas sugeridas como evidencia
#	Qué capturar
1	systemctl status 3proxy en active (running)
2	ss -tlnp | grep 3128 mostrando el puerto en LISTEN
3	sudo cat /etc/3proxy/3proxy.cfg
4	sudo ufw status verbose
5	Configuración del proxy en el Windows 11 cliente
6	curl.exe a example.com (200) y a facebook.com (rechazo)
7	El log en vivo con la IP del cliente
8	Un cliente no permitido siendo denegado (si hiciste el ejercicio de IPs)
11. Cuando cambias de red
Al cambiar de red cambian tres cosas: la IP de la VM, la subred permitida y lo que configuran los clientes. Si no actualizas la subred en 3proxy y en UFW, tu propia laptop quedará bloqueada (por el deny * de 3proxy o por la política de UFW).

Si fijaste la IP dentro de Debian (opción B de la sección 12), esa IP solo es válida en la red donde la configuraste. En una red nueva, la VM quedaría sin Internet. Antes de cambiar de red, vuelve a DHCP (instrucciones en la sección 12) y sigue los pasos de abajo. Si usaste una reserva DHCP en el router (opción A), no hay que tocar nada en la VM, pero la reserva solo vale en ese router.

Paso 1: Apagar la VM y ajustar el adaptador
sudo poweroff

En VirtualBox: Configuración → Red → Adaptador 1 → Nombre:

Si sigues usando la misma tarjeta (otro Wi-Fi), no hay nada que cambiar.
Si pasas de Wi-Fi a cable (o al revés), elige la tarjeta activa.
Enciende la VM.

Paso 2: Obtener la IP y la subred nuevas
En la consola de VirtualBox (la IP anterior ya no sirve para SSH):

ip -4 -br addr
ip -4 route show scope link

Anota la nueva <IP_VM> y la nueva <SUBRED>. Comprueba que hay Internet:

ping -c 3 8.8.8.8

Si no responde, el problema es la red (aislamiento, portal cautivo), no el proxy.

Paso 3: Actualizar 3proxy
Cambia el allow por la subred nueva (ejemplo con 192.168.0.0/24):

sudo sed -i 's#^allow \* .*#allow * 192.168.0.0/24#' /etc/3proxy/3proxy.cfg
sudo grep -n '^allow' /etc/3proxy/3proxy.cfg
sudo systemctl restart 3proxy

El grep confirma que el cambio quedó bien. Si ves más de una línea allow (por ejemplo, una por IP), edita a mano con sudo nano /etc/3proxy/3proxy.cfg, porque el sed las sustituye todas por una sola.
Paso 4: Actualizar UFW (lo activaste, así que este paso es obligatorio)
sudo ufw status numbered

Localiza la regla del puerto 3128 con la subred vieja, bórrala por su número (una a la vez; después de borrar, vuelve a listar porque los números cambian) y agrega la nueva:

sudo ufw delete <N>
sudo ufw allow from 192.168.0.0/24 to any port 3128 proto tcp
sudo ufw status verbose

Revisa también la regla de SSH (22/tcp): si la limitaste a la subred anterior, desde la nueva red no podrás entrar por SSH. En ese caso, trabaja por la consola de VirtualBox y actualízala igual que la del 3128.

Paso 5: Actualizar los clientes
Cada Windows 11 cambia la dirección del proxy por la nueva <IP_VM> (sección 9). El puerto sigue siendo 3128.
Tú también, si pruebas desde tu laptop, y para conectarte por SSH usa la IP nueva.
Resumen: qué cambia y dónde
Elemento	¿Cambia?	Dónde se actualiza
IP de la VM	Sí (por DHCP)	Anotarla; se la pasas a los clientes
allow de 3proxy	Sí, si cambia la subred	/etc/3proxy/3proxy.cfg + systemctl restart 3proxy
Regla UFW del 3128	Sí, si cambia la subred	ufw delete <N> y ufw allow from <SUBRED_NUEVA> ...
Regla UFW de SSH	Solo si la restringiste	Igual que la anterior
Adaptador puente de VirtualBox	Solo si cambia la tarjeta física	Configuración → Red de la VM
IP fija (reserva DHCP o estática)	Sí, en cada red nueva	Reserva: panel del router de esa red. Estática: volver a DHCP o reconfigurar (sección 12)
Proxy en cada Windows 11	Sí (la IP)	Configuración → Red e Internet → Proxy
Puerto, reglas de bloqueo, log	No	—
Consejos para redes distintas
Redes de escuela, universidad o invitados: suelen activar el aislamiento de clientes (los dispositivos no se ven entre sí) o un portal cautivo (inicio de sesión web). Si tus compañeros no alcanzan la VM con ping, esa es la causa más probable. Usa el punto de acceso de un celular como red de la práctica.
Para no editar la subred cada vez, puedes permitir todos los rangos privados: allow * 192.168.0.0/16,10.0.0.0/8,172.16.0.0/12 (y lo mismo en UFW). Es más cómodo, pero menos restrictivo, y el ejercicio de «solo la red interna» se demuestra mejor con la subred exacta. Úsalo solo para pruebas.
IP fija: si controlas el router, reserva una IP para la VM por su MAC y no tendrás que avisar a los clientes de una IP nueva cada vez. Los pasos están en la sección 12. En redes que no administras (escuela, punto de acceso de un celular) no podrás reservarla: confirma la IP antes de cada práctica.
12. IP fija para la VM
3proxy y UFW dependen de la subred, no de la IP de la VM, así que una IP que cambie no rompe sus reglas. Pero obliga a avisar a los clientes (y a usar otra IP para SSH) cada vez que cambia. Fijarla evita ese paso.

Hay dos formas:

A. Reserva DHCP en el router	B. IP estática en Debian
Requiere acceso al router	Sí	No
Cambia la configuración de la VM	No	Sí
Al cambiar de red	La VM sigue en DHCP y toma la IP que le den en esa red	Queda mal configurada hasta volver a DHCP
Recomendada	Sí, si controlas el router	Solo para una red fija sin acceso al router
Opción A (recomendada): reserva DHCP en el router
La VM sigue pidiendo IP por DHCP, pero el router le da siempre la misma.

Obtén la MAC de la VM (en la VM):

ip link show enp0s3

Es el valor que aparece después de link/ether. También la ves en VirtualBox: Configuración → Red → Avanzadas → Dirección MAC.

Entra al panel del router. Su dirección suele ser la puerta de enlace (ip route | grep default en la VM), a menudo 192.168.1.1 o 192.168.0.1. El usuario y la contraseña suelen estar en una etiqueta del router.

Busca la sección de reservas. El nombre cambia según la marca: Reserva DHCP, Static lease, Address reservation, Asignación estática o Vincular IP-MAC, normalmente dentro de «DHCP» o «LAN».

Agrega la reserva: la MAC de la VM y la IP que quieras. Debe estar dentro de tu subred y no ser la de otro dispositivo. Muchos routers piden elegir el equipo de una lista de clientes conectados, por eso conviene tener la VM encendida.

Guarda y reinicia la VM (sudo reboot). Comprueba con ip -4 -br addr que tomó la IP reservada.

Con Wi-Fi, a veces el router ve la MAC de tu laptop en lugar de la de la VM, por cómo funciona el puente de VirtualBox. Si la MAC de la VM no aparece en la lista de clientes, busca el cliente por su IP actual y reserva la MAC que muestre el router para esa IP. Por cable no suele pasar.

Limitación: la reserva vale solo para ese router. Si usas varias redes hay que hacerla en cada una, y en redes que no administras (escuela, punto de acceso de un celular) no se puede.

Opción B: IP estática dentro de Debian
Sirve sin acceso al router, con dos inconvenientes:

Solo funciona en esa red. En otra red, la IP y la puerta de enlace serían incorrectas y la VM se quedaría sin Internet hasta que vuelvas a DHCP.
Riesgo de choque de IP. Elige una IP fuera del rango que reparte el DHCP del router (lo ves en su panel), o puede coincidir con otro dispositivo.
1. Reúne los datos actuales (con la VM ya conectada y en DHCP):

ip -4 -br addr
ip route | grep default
cat /etc/resolv.conf

Necesitas la IP que usarás (address), la puerta de enlace (la IP que sigue a default via) y el servidor DNS (nameserver).

2. Comprueba cómo gestiona Debian la red:

cat /etc/network/interfaces
systemctl is-active NetworkManager systemd-networkd

Si interfaces contiene iface enp0s3 inet dhcp (lo más probable en una instalación sin escritorio), sigue con el paso 3.
Si NetworkManager aparece como active, ve a la variante con nmcli más abajo.
3. Edita /etc/network/interfaces (respalda antes). Cambia solo la línea iface enp0s3 inet dhcp y deja intactas las líneas auto o allow-hotplug:

sudo cp /etc/network/interfaces /etc/network/interfaces.dhcp.bak
sudo nano /etc/network/interfaces

Sustitúyela por este bloque, con tus valores:

iface enp0s3 inet static
    address 192.168.1.50/24
    gateway 192.168.1.1

4. Reinicia y comprueba (reiniciar es más seguro que reiniciar la red si estás conectado por SSH):

sudo reboot
ip -4 -br addr
ping -c 3 8.8.8.8
ping -c 3 deb.debian.org

Si responde 8.8.8.8 pero falla deb.debian.org, falta el DNS. Revisa cat /etc/resolv.conf y, si el archivo es normal y está vacío o tiene un servidor incorrecto, escribe en él nameserver <tu puerta de enlace>. El DNS es necesario, porque 3proxy resuelve por el cliente los nombres de los sitios que visita.

Volver a DHCP (por ejemplo, antes de cambiar de red), desde la consola de VirtualBox:

sudo cp /etc/network/interfaces.dhcp.bak /etc/network/interfaces
sudo reboot

Variante con NetworkManager (solo si aparece active):

nmcli con show
sudo nmcli con mod "<nombre_de_la_conexion>" ipv4.method manual ipv4.addresses 192.168.1.50/24 ipv4.gateway 192.168.1.1 ipv4.dns 192.168.1.1
sudo nmcli con up "<nombre_de_la_conexion>"

Para volver a DHCP: sudo nmcli con mod "<nombre_de_la_conexion>" ipv4.method auto ipv4.addresses "" ipv4.gateway "" ipv4.dns "" y luego sudo nmcli con up "<nombre_de_la_conexion>".

Qué no cambia con una IP fija
3proxy y UFW no necesitan cambios: sus reglas dependen de la subred. Lo único que sigue pendiente al cambiar de red es la sección 11.

13. Apagar, encender y reactivar el servicio
Apagar la VM correctamente
sudo poweroff

Usa un apagado real y no «Guardar el estado de la máquina»: al restaurar un estado guardado en otra red, la VM retoma el estado anterior, incluida la IP, y puede quedarse con una dirección de la red anterior.

¿Hay que detener y deshabilitar 3proxy cada vez?
Situación	Qué hacer
Siempre apagas la VM al terminar (lo habitual)	Nada. Al apagar, el servicio se detiene solo, y al estar enabled, arranca solo la próxima vez que enciendas la VM.
Dejas la VM encendida entre prácticas	sudo systemctl stop 3proxy y sudo systemctl disable 3proxy para que no quede un proxy abierto.
Vas a encender la VM en una red que no controlas	Deja el servicio deshabilitado y arráncalo a mano solo cuando lo necesites, o restringe el allow a IPs concretas (ver «Cuidado con subredes iguales»).
Al encender la VM: lista de comprobación
IP y subred actuales:

ip -4 -br addr
ip -4 route show scope link

Servicio y puerto:

systemctl status 3proxy --no-pager
sudo ss -tlnp | grep 3128

Que las reglas coinciden con la red actual:

sudo grep -n '^allow' /etc/3proxy/3proxy.cfg
sudo ufw status numbered

La subred del allow y la de la regla UFW del 3128 deben coincidir con la que te mostró ip -4 route show scope link.

Avisar a los clientes si la IP cambió.

Probar con curl.exe desde tu laptop (sección 10).

Según lo que encuentres:

Qué pasó	Qué hacer
Misma red, misma IP	Nada.
Misma red, IP distinta	No cambies 3proxy ni UFW. Avisa a los clientes de la nueva IP. Para SSH, usa también la nueva.
Subred distinta	Sección 11: actualiza el allow de 3proxy y la regla de UFW del 3128.
Reactivar el servicio manualmente
# Si lo detuviste Y lo deshabilitaste (stop + disable):
sudo systemctl enable --now 3proxy

# Si solo lo detuviste (stop):
sudo systemctl start 3proxy

Si no recuerdas cuál fue, usa el primero: sirve en ambos casos. Después comprueba con systemctl status 3proxy --no-pager y sudo ss -tlnp | grep 3128.

Si también apagaste UFW (sudo ufw disable), reactívalo:

sudo ufw enable
sudo ufw status verbose

Cuidado con subredes iguales
Muchas redes domésticas usan 192.168.1.0/24. Si llevas la VM a otra red con el mismo rango, tu allow y tu regla de UFW le abrirían el proxy a cualquier dispositivo de esa red, porque la subred coincide. En una subred distinta, el deny * y UFW lo rechazan todo, que es el comportamiento seguro.

Para redes que no controlas:

Restringe el allow a las IPs concretas de tus compañeros (allow * 192.168.1.30,192.168.1.31) y haz lo mismo en UFW, o
deja 3proxy deshabilitado y arráncalo solo cuando lo necesites.
Al terminar la práctica
Pide a tus compañeros que desactiven el proxy en Windows 11 (sección 9).
Toma una instantánea de la VM con la práctica funcionando (Máquina → Tomar instantánea) para repetirla sin reinstalar.
El log guarda los sitios visitados por tus compañeros: avísales antes de la práctica y elimínalo o consérvalo según corresponda (sudo rm /var/log/3proxy/* lo borra).
Revisa que la red de tu institución permita este tipo de pruebas.
Si dejas la VM encendida, detén el servicio y, si quieres, apaga UFW (ver la tabla de arriba).
14. Problemas frecuentes
Síntoma	Causa probable	Solución
cat: /etc/3proxy/3proxy.cfg: Permiso denegado	El archivo no es legible sin privilegios (es normal)	Usa sudo cat o sudo nano
curl: orden no encontrada	Debian mínimo no trae curl	sudo apt install -y curl
El servicio no arranca	Error de sintaxis en el .cfg	sudo journalctl -u 3proxy -n 30 --no-pager, corrige y restart
El servicio tarda en volver tras un error	RestartSec=60s	Usa siempre sudo systemctl restart 3proxy
Todo da rechazo, incluso desde tu laptop	La subred del allow (o de UFW) no corresponde a tu red actual	Sección 11, pasos 3 y 4
Los clientes no alcanzan la VM (ping sin respuesta)	Aislamiento de clientes en la red	Usa el punto de acceso de un celular
El ping funciona pero el proxy no responde	Regla de UFW del 3128 incorrecta, o proxy mal configurado en el cliente	sudo ufw status verbose y revisa la IP y el puerto en el cliente
La VM no obtiene IP en puente sobre Wi-Fi	Limitación del modo puente con algunas tarjetas	Modo promiscuo «Permitir todo»; si persiste, usa cable
Te quedaste fuera por SSH tras activar UFW	No se permitió el puerto 22 antes de enable	Entra por la consola de VirtualBox: sudo ufw allow 22/tcp o sudo ufw disable
El log no muestra a un cliente	UFW lo bloqueó antes de llegar a 3proxy	Revisa sudo ufw status verbose
El navegador del cliente no carga nada	El proxy del sistema sigue activo y la VM está apagada	Desactiva el proxy (sección 9)
La VM va lenta	VirtualBox y WSL2 comparten el hipervisor de Windows	wsl --shutdown o reduce los recursos asignados
La VM no tiene Internet tras fijar una IP estática	Puerta de enlace o DNS incorrectos, o red distinta a la configurada	ip route | grep default, cat /etc/resolv.conf; si cambiaste de red, vuelve a DHCP (sección 12)
La IP de la VM sigue cambiando pese a la reserva	La MAC reservada no es la que ve el router, o la IP está fuera del rango	Revisa la MAC en la lista de clientes del router y reinicia la VM
La VM conserva una IP de la red anterior	Se restauró un estado guardado en vez de un apagado real	sudo reboot y apaga siempre con sudo poweroff
15. Referencia rápida de comandos
Servicio 3proxy

sudo systemctl status 3proxy --no-pager
sudo systemctl restart 3proxy
sudo systemctl stop 3proxy
sudo systemctl start 3proxy
sudo systemctl enable --now 3proxy
sudo systemctl disable 3proxy
sudo journalctl -u 3proxy -n 30 --no-pager
sudo nano /etc/3proxy/3proxy.cfg
sudo cat /etc/3proxy/3proxy.cfg

Red

ip -4 -br addr
ip -4 route show scope link
ip route | grep default
ip link show enp0s3
cat /etc/resolv.conf
sudo ss -tlnp | grep 3128
ping -c 3 8.8.8.8
sudo reboot
sudo poweroff

Log

sudo ls -l /var/log/3proxy
sudo sh -c 'tail -n 20 -f /var/log/3proxy/*'

UFW

sudo ufw status verbose
sudo ufw status numbered
sudo ufw delete <N>
sudo ufw allow from <SUBRED> to any port 3128 proto tcp
sudo ufw allow 22/tcp
sudo ufw disable
sudo ufw enable

Pruebas (Windows 11, PowerShell)

ping <IP_VM>
curl.exe -x http://<IP_VM>:3128 -I http://example.com
curl.exe -x http://<IP_VM>:3128 -I http://www.facebook.com
ssh proxyadmin@<IP_VM>

Datos de la instalación documentada: Debian 13 «Trixie» (netinst amd64), 3proxy 0.9.7 (paquete .deb), VirtualBox con adaptador puente, UFW activo y clientes Windows 11. El texto exacto de los mensajes de rechazo de 3proxy no se verificó en esta práctica; captura los que veas en tu equipo.
