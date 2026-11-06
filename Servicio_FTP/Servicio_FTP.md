# Instalación y configuración del servicio FTP en una Raspberry Pi

## Índice

- [Introducción](#introducción)
- [Objetivo del informe](#objetivo-del-informe)
- [Desarrollo teórico](#desarrollo-teórico)
  - [Red LAN y arquitectura cliente-servidor](#red-lan-y-arquitectura-cliente-servidor)
  - [Protocolo FTP](#protocolo-ftp)
  - [Funcionamiento de FTP y conexiones TCP](#funcionamiento-de-ftp-y-conexiones-tcp)
  - [Modo activo y modo pasivo](#modo-activo-y-modo-pasivo)
  - [Usuarios, autenticación y permisos](#usuarios-autenticación-y-permisos)
  - [vsftpd y FileZilla](#vsftpd-y-filezilla)
  - [Seguridad: FTP, FTPS y SFTP](#seguridad-ftp-ftps-y-sftp)
  - [Diseño de la red](#diseño-de-la-red)
- [Instalación del servicio FTP](#instalación-del-servicio-ftp)
- [Configuración del servidor FTP](#configuración-del-servidor-ftp)
  - [Edición del archivo de configuración](#edición-del-archivo-de-configuración)
  - [Directivas configuradas](#directivas-configuradas)
  - [Reinicio y estado del servicio](#reinicio-y-estado-del-servicio)
- [Verificación del servicio y del puerto 21](#verificación-del-servicio-y-del-puerto-21)
- [Preparación de los archivos compartidos](#preparación-de-los-archivos-compartidos)
- [Configuración de la red LAN](#configuración-de-la-red-lan)
  - [Diagnóstico de la interfaz Ethernet](#diagnóstico-de-la-interfaz-ethernet)
  - [Asignación temporal de la IP del servidor](#asignación-temporal-de-la-ip-del-servidor)
  - [Configuración manual de la notebook](#configuración-manual-de-la-notebook)
- [Conexión desde FileZilla](#conexión-desde-filezilla)
- [Pruebas de funcionamiento](#pruebas-de-funcionamiento)
  - [Visualización y lectura de un archivo del servidor](#visualización-y-lectura-de-un-archivo-del-servidor)
  - [Subida de un archivo al servidor](#subida-de-un-archivo-al-servidor)
- [Dificultades encontradas y soluciones](#dificultades-encontradas-y-soluciones)
- [Consideraciones para futuras demostraciones](#consideraciones-para-futuras-demostraciones)
- [Conclusión](#conclusión)

---

## Introducción

En esta entrega se documenta la instalación, configuración y prueba de un **servidor FTP** en una **Raspberry Pi** con Raspberry Pi OS. El objetivo es que la Raspberry funcione como servidor de archivos dentro de la red local del **grupo 3**, permitiendo que una notebook conectada al mismo switch pueda acceder a sus directorios y transferir archivos.

Se utilizó **vsftpd** como software servidor y **FileZilla Client** como programa cliente en Windows. El servicio se incorporó a la red `192.168.3.0/24` desarrollada en la entrega anterior, dedicada al servidor DHCP. Para la prueba de esta entrega se mantuvo detenido el servidor DHCP y se asignaron manualmente las direcciones IP, evitando interferencias con la red del colegio.

La prueba final fue satisfactoria: se logró conectar desde FileZilla, visualizar un archivo alojado en la Raspberry, leer su contenido desde la notebook y subir un nuevo archivo al servidor.

---

## Objetivo del informe

El objetivo es documentar el proceso de:

- Comprender qué es FTP y cómo se realiza una transferencia de archivos entre un cliente y un servidor.
- Identificar la función de las conexiones TCP, los puertos, los usuarios y los permisos.
- Instalar el paquete `vsftpd` en Raspberry Pi OS.
- Configurar el servicio para permitir el acceso de usuarios locales y la transferencia de archivos.
- Verificar que el servicio esté activo y escuchando en el puerto TCP 21.
- Crear un directorio de prueba y un archivo inicial en la Raspberry.
- Preparar la red LAN del grupo mediante el switch, con IP `192.168.3.1` para la Raspberry y `192.168.3.2` para la notebook.
- Conectarse desde FileZilla y demostrar la visualización, lectura y subida de archivos.
- Registrar los problemas detectados y las soluciones aplicadas.

---

# Desarrollo teórico

## Red LAN y arquitectura cliente-servidor

Una **LAN** (*Local Area Network*, red de área local) es una red que conecta equipos dentro de un espacio reducido. En nuestro caso, la notebook y la Raspberry Pi se conectan por cable Ethernet a un **switch**, que permite intercambiar tramas entre los equipos de la misma red. No se requiere acceso a Internet para que los equipos de la LAN intercambien archivos.

En una arquitectura **cliente-servidor** se distinguen dos roles:

- **Servidor:** equipo que ejecuta un programa que permanece disponible para atender solicitudes. Aquí es la Raspberry Pi, que ejecuta `vsftpd`.
- **Cliente:** equipo o aplicación que solicita acceder al servicio. Aquí es la notebook con FileZilla.

Un mismo equipo puede ofrecer distintos servicios simultáneamente. Por ejemplo, la Raspberry puede tener instalados DHCP y FTP, aunque cada servicio cumple una función diferente: **DHCP entrega configuraciones de red; FTP transfiere archivos**.

---

## Protocolo FTP

**FTP** significa *File Transfer Protocol* (**Protocolo de Transferencia de Archivos**). Es un protocolo de aplicación diseñado para intercambiar archivos entre equipos conectados por una red TCP/IP.

Con FTP, un cliente autenticado puede, según los permisos que tenga, consultar carpetas, listar archivos, descargar documentos del servidor, subir nuevos archivos, cambiar nombres o eliminar archivos. FTP no se limita a Internet: también funciona en una red LAN aislada como la utilizada en este trabajo.

Una **descarga** consiste en copiar un archivo desde el servidor hacia el cliente. Una **subida** consiste en copiar un archivo desde el cliente hacia el servidor. En ambos casos, el archivo original no se elimina automáticamente: se transfiere su contenido para crear una copia en el destino.

---

## Funcionamiento de FTP y conexiones TCP

FTP utiliza **TCP** (*Transmission Control Protocol*), un protocolo de transporte orientado a conexión. TCP permite transmitir información en orden y detectar pérdidas para retransmitir segmentos cuando sea necesario.

FTP se caracteriza por separar dos tipos de comunicación:

| Conexión | Función | Puerto asociado |
|---|---|---|
| **Control** | Enviar comandos, credenciales y respuestas del servidor. | TCP **21** en el servidor, de forma habitual. |
| **Datos** | Transportar el listado de directorios y el contenido de los archivos. | Depende del modo de transferencia. |

Al establecer una sesión, el cliente se comunica con el puerto 21 del servidor. El servidor puede enviar un mensaje de bienvenida que comienza con el código `220`. Después se realiza la autenticación y se intercambian órdenes. Cuando el usuario solicita un listado o una transferencia, se utiliza además una conexión de datos.

Algunos comandos habituales del protocolo son `USER` (nombre de usuario), `PASS` (contraseña), `LIST` (listado), `RETR` (descarga) y `STOR` (subida). Normalmente FileZilla envía estos comandos automáticamente, sin que el usuario tenga que escribirlos.

> **Importante:** que el puerto TCP 21 esté en escucha demuestra que el servidor espera conexiones de control, pero no garantiza por sí solo que la autenticación o las transferencias de datos funcionen. Por eso se hicieron pruebas reales desde el cliente.

---

## Modo activo y modo pasivo

FTP admite dos formas principales de establecer la **conexión de datos**:

| Característica | Modo activo | Modo pasivo |
|---|---|---|
| Quién inicia la conexión de datos | El servidor hacia el cliente. | El cliente hacia el servidor. |
| Puertos de datos | Tradicionalmente se utiliza el puerto 20 del servidor como origen. | El servidor anuncia un puerto de datos en el que espera la conexión. |
| Uso habitual | Puede requerir permitir conexiones entrantes hacia el cliente. | Suele ser más sencillo con firewalls y NAT. |

En **modo pasivo**, el cliente primero mantiene la conexión de control al puerto 21 y después abre una segunda conexión hacia el puerto de datos anunciado por el servidor. Es el modo habitual de FileZilla.

Es importante no confundir **puerto 21** (control) con **puerto 20**: este último se asocia históricamente al modo activo, pero no es el puerto universal para transferir datos FTP.

---

## Usuarios, autenticación y permisos

Para controlar quién puede acceder al servidor, FTP puede utilizar credenciales. En esta práctica se configuró el acceso mediante **usuarios locales de Raspberry Pi OS**, es decir, cuentas ya existentes en el sistema operativo. En nuestro caso se utilizó el usuario `grupo3`.

El servidor verifica el usuario y su contraseña. Una vez autenticado, las operaciones permitidas dependen tanto de las opciones de `vsftpd` como de los **permisos de Linux** sobre los archivos y directorios.

En Linux, los permisos básicos son:

- **Lectura (`r`):** permite leer el contenido de un archivo; en directorios, consultar sus nombres.
- **Escritura (`w`):** permite modificar archivos; en directorios, crear o eliminar entradas si también se dispone de los permisos necesarios.
- **Ejecución (`x`):** permite ejecutar archivos; en directorios, acceder o atravesar sus rutas.

Por lo tanto, habilitar la escritura en `vsftpd` no significa que cualquier usuario pueda modificar cualquier archivo: siguen aplicándose los permisos del sistema.

---

## vsftpd y FileZilla

**vsftpd** significa *Very Secure FTP Daemon*. Es un programa servidor FTP utilizado en sistemas Linux. Un **daemon** es un proceso que se ejecuta en segundo plano para prestar un servicio y atender solicitudes sin necesidad de mantener una terminal abierta.

Su archivo de configuración principal es `/etc/vsftpd.conf`, y en Raspberry Pi OS el servicio se administra mediante `systemctl`.

**FileZilla Client** es una aplicación gráfica que permite conectarse a servidores de archivos. Presenta los archivos locales y remotos en paneles separados y permite transferirlos arrastrándolos de un panel al otro. FileZilla es el **cliente**; no reemplaza al servidor `vsftpd` que se ejecuta en la Raspberry.

---

## Seguridad: FTP, FTPS y SFTP

El **FTP tradicional no cifra** ni el contenido transferido ni las credenciales. Por ello, en esta actividad se utilizó únicamente dentro de una **LAN de laboratorio aislada y confiable**. No se recomienda habilitar FTP plano en Internet ni en redes compartidas donde terceros puedan interceptar el tráfico.

Conviene distinguir tres tecnologías:

| Tecnología | Descripción |
|---|---|
| **FTP** | Transferencia de archivos sin cifrado por defecto. |
| **FTPS** | FTP protegido con TLS; mantiene el modelo de conexiones de FTP. |
| **SFTP** | Protocolo de transferencia de archivos sobre SSH; es distinto de FTP y normalmente utiliza el puerto 22. |

Para un entorno de producción sería preferible FTPS o SFTP, limitar los usuarios autorizados y aplicar permisos restrictivos. La configuración de este informe responde a una práctica educativa en una red local controlada.

---

## Diseño de la red

Esta entrega reutiliza la red del **grupo 3**, `192.168.3.0/24`, desarrollada en la entrega de DHCP.

| Equipo | Función en esta entrega | Dirección IPv4 |
|---|---|---|
| Raspberry Pi | Servidor FTP (`vsftpd`) | `192.168.3.1/24` |
| Notebook Windows | Cliente FTP (FileZilla) | `192.168.3.2/24` |
| Switch | Interconectar físicamente los equipos | No necesita IP para conmutar tráfico |

La máscara de subred es `255.255.255.0` (`/24`). Ambos dispositivos pertenecen a la misma red, por lo que pueden comunicarse directamente sin un router. Durante la prueba no se configuró una puerta de enlace ni DNS, ya que no eran necesarios para acceder al servidor por su dirección IP.

En la entrega anterior se había configurado el servidor DHCP para entregar direcciones del rango `192.168.3.100` a `192.168.3.200`. **Durante esta prueba FTP el DHCP de la Raspberry permaneció detenido y se utilizaron direcciones manuales**. Por lo tanto, el funcionamiento de FTP quedó demostrado de manera independiente de la asignación automática de direcciones.

```text
 [Notebook Windows]          [Switch]          [Raspberry Pi]
  FileZilla (cliente)  ----  (LAN)  ----  vsftpd (servidor)
   192.168.3.2/24                          192.168.3.1/24
```

> **Esquema 1:** Red LAN del grupo 3 utilizada para la prueba de FTP.

---

# Instalación del servicio FTP

Para instalar el servidor, la Raspberry necesitó conectividad con los repositorios de paquetes. Primero se actualizó la lista de paquetes disponibles y luego se instaló `vsftpd`:

```bash
sudo apt update
sudo apt install vsftpd -y
```

### Explicación de los comandos

- `sudo`: ejecuta la orden con privilegios de administrador, necesarios para instalar software.
- `apt`: gestor de paquetes de sistemas derivados de Debian, como Raspberry Pi OS.
- `update`: consulta los repositorios configurados y descarga la información actualizada sobre los paquetes disponibles. **No actualiza automáticamente los programas ya instalados**. Requiere conectividad y resolución DNS funcional para consultar los repositorios por su nombre.
- `install`: solicita instalar el paquete indicado.
- `vsftpd`: paquete que contiene el servidor FTP utilizado en el trabajo.
- `-y`: confirma automáticamente la instalación cuando `apt` solicita autorización.

La instalación incorpora los archivos del programa, su configuración y la unidad del servicio que puede ser controlada por `systemctl`.

![Actualización de paquetes e instalación de vsftpd](imagenes/ftp_1.jpeg)

> **Figura 1:** Ejecución de `sudo apt update` y `sudo apt install vsftpd -y` en la terminal de la Raspberry.

---

# Configuración del servidor FTP

## Edición del archivo de configuración

Una vez instalado el servicio, se editó su archivo de configuración principal con el editor `nano`:

```bash
sudo nano /etc/vsftpd.conf
```

### Explicación del comando

- `nano`: editor de texto que funciona en la terminal.
- `sudo`: necesario para guardar cambios en archivos de configuración del sistema.
- `/etc/vsftpd.conf`: archivo de configuración principal de `vsftpd`.

Dentro del archivo se localizaron y ajustaron las siguientes directivas, retirando `#` cuando correspondía y evitando duplicarlas:

```ini
anonymous_enable=NO
local_enable=YES
write_enable=YES
local_umask=022
```

Para guardar los cambios en `nano` se utilizó `Ctrl + O` para guardar, `Enter` para confirmar el nombre del archivo y `Ctrl + X` para salir.

> **Recomendación:** antes de modificar un archivo de configuración conviene hacer una copia de respaldo, por ejemplo con `sudo cp /etc/vsftpd.conf /etc/vsftpd.conf.bak`, para poder recuperar la configuración anterior si se comete un error.

---

## Directivas configuradas

### `anonymous_enable=NO`

Deshabilita el acceso anónimo. Esto impide que una persona entre sin autenticarse mediante una cuenta autorizada. Es importante porque el servidor está pensado para los integrantes del grupo, no para acceso público.

### `local_enable=YES`

Permite el inicio de sesión mediante usuarios locales de Raspberry Pi OS. FileZilla utiliza el nombre de usuario y la contraseña correspondientes a una cuenta existente en la Raspberry.

### `write_enable=YES`

Habilita los comandos FTP que pueden modificar el contenido del servidor, como subir archivos. Esta directiva es necesaria para la prueba de escritura, pero no reemplaza los permisos propios del sistema Linux.

### `local_umask=022`

Define la máscara de permisos aplicada a archivos y directorios nuevos creados por usuarios locales mediante FTP. La **umask** restringe permisos respecto de los valores base. Con `022`, normalmente los archivos nuevos quedan con permisos `644` (`rw-r--r--`) y los directorios con `755` (`rwxr-xr-x`), salvo otras restricciones o configuraciones.

La notación `022` es **octal**: cada dígito representa bits de permisos para propietario, grupo y otros usuarios.

---

## Reinicio y estado del servicio

Para que el programa leyera la nueva configuración se reinició el servicio y se comprobó su estado:

```bash
sudo systemctl restart vsftpd
sudo systemctl status vsftpd
```

### Explicación de los comandos

- `systemctl`: herramienta de administración de servicios de `systemd`.
- `restart`: detiene y vuelve a iniciar el servicio para aplicar la configuración nueva.
- `status`: muestra si el servicio está cargado y en ejecución, junto con mensajes recientes.
- `vsftpd`: servicio sobre el que se actúa.

![Archivo de configuración editado y reinicio del servicio](imagenes/ftp_2.jpeg)

> **Figura 2:** En la parte superior se observa el editor `nano` con el archivo `/etc/vsftpd.conf` ya modificado con las directivas descriptas (el contenido no se aprecia en la captura). Debajo aparecen los comandos `sudo systemctl restart vsftpd` y `sudo systemctl status vsftpd`.

---

# Verificación del servicio y del puerto 21

Se comprobó nuevamente el estado del servicio y se verificó que estuviera esperando conexiones TCP en el puerto 21:

```bash
sudo systemctl status vsftpd
sudo ss -lntp | grep ':21'
```

### Explicación de los comandos

- `systemctl status vsftpd`: muestra el estado del servicio. Se espera ver **`Active: active (running)`**, que confirma que el servidor está funcionando. Para salir de la pantalla de estado se puede presionar `Q`.
- `ss`: herramienta que permite consultar sockets y conexiones de red.
- `-l`: muestra sockets en estado de escucha (*listening*).
- `-n`: muestra direcciones y puertos numéricamente, sin traducirlos a nombres.
- `-t`: limita la salida a conexiones TCP.
- `-p`: intenta mostrar el proceso asociado al socket.
- `|`: tubería que envía la salida del primer comando al siguiente.
- `grep ':21'`: filtra las líneas que contienen la secuencia `:21`.

La salida mostró una línea con estado **`LISTEN`** para el puerto 21, lo que confirmó que `vsftpd` estaba escuchando solicitudes de conexión FTP.

![Estado del servicio y puerto 21 en escucha](imagenes/ftp_3.jpeg)

> **Figura 3:** Resultado de `sudo systemctl status vsftpd` y de `sudo ss -lntp | grep ':21'`.

---

# Preparación de los archivos compartidos

Se creó un directorio llamado `ftp` dentro de la carpeta personal del usuario de la Raspberry y se generó un archivo de prueba:

```bash
mkdir -p ~/ftp
mkdir -p ~/home/grupo3/ftp
ls -ld ~/ftp
echo "Prueba del servidor FTP - Grupo 3" > ~/ftp/prueba.txt
cat ~/ftp/prueba.txt
```

### Explicación de los comandos

- `mkdir`: crea directorios.
- `-p`: crea también los directorios intermedios que falten y no produce error si el directorio ya existe.
- `~`: representa el directorio personal del usuario que ejecuta el comando. Para el usuario `grupo3`, `~/ftp` equivale a `/home/grupo3/ftp`, que es la carpeta utilizada en la práctica.
- `mkdir -p ~/home/grupo3/ftp`: este segundo comando se ejecutó intentando indicar la ruta completa, pero como `~` ya equivale a `/home/grupo3`, creó una estructura anidada innecesaria (`/home/grupo3/home/grupo3/ftp`). La carpeta que efectivamente se utilizó en la prueba fue `~/ftp`.
- `ls -ld ~/ftp`: `ls` lista archivos y directorios; `-l` muestra información detallada (permisos, propietario, fecha) y `-d` muestra los datos del directorio en sí, en vez de listar su contenido.
- `echo "..."`: escribe un texto en la salida estándar.
- `>`: redirige la salida hacia un archivo, creándolo si no existe o sobrescribiéndolo si ya existe.
- `~/ftp/prueba.txt`: ruta del archivo de prueba creado.
- `cat`: muestra el contenido de un archivo de texto sin modificarlo.

El resultado de `cat` fue:

```text
Prueba del servidor FTP - Grupo 3
```

Este archivo sirvió para verificar posteriormente que el cliente podía visualizar y descargar contenido del servidor.

![Creación de la carpeta de prueba y diagnóstico de red](imagenes/ftp_4.jpeg)

> **Figura 4:** Continuación de la captura de la Figura 2. En la parte inicial se observan los comandos de esta sección (`mkdir -p ~/ftp`, `mkdir -p ~/home/grupo3/ftp`, `ls -ld ~/ftp`, `echo ... > ~/ftp/prueba.txt` y `cat ~/ftp/prueba.txt`). A continuación aparecen los comandos de diagnóstico de red que se explican en la sección siguiente.

---

# Configuración de la red LAN

## Diagnóstico de la interfaz Ethernet

Tras crear el archivo de prueba se revisó el estado de red de la Raspberry mediante los siguientes comandos:

```bash
hostname -I
ip -4 addr show eth0
nmcli device status
ip link show eth0
```

### Explicación de los comandos

- `hostname -I`: muestra las direcciones IP asignadas al equipo. En este caso no mostró ninguna, lo que indicaba que la interfaz no tenía dirección.
- `ip -4 addr show eth0`: `ip addr show` muestra las direcciones de una interfaz; `-4` limita la salida a IPv4 y `eth0` es la interfaz Ethernet consultada.
- `nmcli device status`: `nmcli` es la herramienta de línea de comandos de NetworkManager; `device status` muestra el estado de las interfaces de red y si están conectadas o no.
- `ip link show eth0`: consulta el estado físico de la interfaz. La señal **`NO-CARRIER`** indica que el sistema no detecta enlace físico; **`LOWER_UP`** indica que el enlace con el switch sí fue detectado.

La ausencia de enlace físico o de dirección IP es distinta de un error de configuración de FTP: antes de intercambiar paquetes IP, el cable, el puerto del switch y la dirección de la interfaz deben estar en orden.

![Creación de la carpeta de prueba y diagnóstico de red](imagenes/ftp_4.jpeg)

> **Figura 4 (repetida):** Misma captura de la sección anterior. En esta parte interesan los comandos de diagnóstico de red: `hostname -I`, `ip -4 addr show eth0`, `nmcli device status` e `ip link show eth0`.

---

## Asignación temporal de la IP del servidor

Como la interfaz no tenía la dirección deseada, se la asignó manualmente:

```bash
sudo ip addr add 192.168.3.1/24 dev eth0
ip -4 addr show eth0
```

### Explicación de los comandos

- `sudo`: ejecuta la modificación con privilegios de administrador.
- `ip addr add`: agrega una dirección IP a una interfaz.
- `192.168.3.1/24`: IP de la Raspberry y máscara `/24`, equivalente a `255.255.255.0`.
- `dev eth0`: selecciona la interfaz Ethernet.
- `ip -4 addr show eth0`: permite comprobar que aparezca `inet 192.168.3.1/24`.

La dirección **desapareció en varias ocasiones** después de asignarla, por lo que el procedimiento se repitió: se volvió a aplicar `ip addr add` y se verificó con `ip -4 addr show eth0` hasta que la dirección quedó presente. Una vez estable, se comprobó otra vez el servicio y el puerto:

```bash
sudo systemctl status vsftpd
sudo ss -lntp | grep ':21'
ip -4 addr show eth0
```

Con esto se confirmó que, con la IP asignada, `vsftpd` seguía activo, el puerto 21 seguía en escucha y la interfaz mantenía la dirección `192.168.3.1/24`.

> **Nota:** `ip addr add` configura una dirección **temporal**, no persistente. Puede desaparecer al reiniciar o al cambiar el estado o perfil de la conexión, lo cual explica lo ocurrido durante la práctica. Para hacerla permanente se necesita un paso adicional en el administrador de red que no se realizó en esta entrega.

![Asignación repetida de la IP y comprobaciones finales](imagenes/ftp_5.jpeg)

> **Figura 5:** Se repite varias veces `sudo ip addr add 192.168.3.1/24 dev eth0` seguido de `ip -4 addr show eth0`, hasta ejecutar `sudo systemctl status vsftpd`, luego `sudo ss -lntp | grep ':21'` y, por último, `ip -4 addr show eth0`.

---

## Configuración manual de la notebook

Como el servidor DHCP propio estaba detenido, se configuró manualmente el adaptador Ethernet de Windows. Para abrir la lista de adaptadores se utilizó `Windows + R` y el comando:

```text
ncpa.cpl
```

Después se ingresó a **Ethernet → Propiedades → Protocolo de Internet versión 4 (TCP/IPv4) → Propiedades** y se seleccionó la opción de utilizar una dirección IP manual.

| Parámetro | Valor |
|---|---|
| Dirección IPv4 | `192.168.3.2` |
| Máscara de subred | `255.255.255.0` |
| Puerta de enlace | Sin configurar |
| DNS | Sin configurar |

Esta dirección pertenece a la misma red que la Raspberry (`192.168.3.0/24`) y es diferente de `192.168.3.1`, por lo que no genera un conflicto entre ambos equipos.

> **Importante:** al terminar la práctica, si la notebook debe conectarse a otra red que utiliza DHCP, conviene restaurar la opción **Obtener una dirección IP automáticamente**.

---

# Conexión desde FileZilla

Se utilizó **FileZilla Client** en la notebook Windows para acceder a la Raspberry. Los parámetros de conexión fueron:

| Campo | Valor utilizado |
|---|---|
| Protocolo | FTP |
| Servidor / Host | `192.168.3.1` |
| Puerto | `21` |
| Usuario | `grupo3` (usuario local de Raspberry Pi OS) |
| Contraseña | Contraseña de ese usuario |
| Modo de transferencia | Pasivo |
| Cifrado | FTP sin cifrado, únicamente para la LAN de laboratorio |

FileZilla permite introducir estos datos desde la barra de **Conexión rápida** o mediante **Archivo → Gestor de sitios**, donde se pueden guardar los parámetros de conexión.

Al principio FileZilla mostraba mensajes como:

```text
Conectando a 192.168.3.1:21...
La conexión superó el tiempo de espera después de 20 segundos de inactividad
No se pudo conectar al servidor
```

Se verificó que el servicio estaba activo, que la IP era correcta y que el puerto TCP 21 estaba en escucha. Finalmente se corrigió la configuración de conexión en FileZilla indicando explícitamente el puerto `21`, y **se logró ingresar y visualizar los archivos de la Raspberry**.

Una vez conectado, FileZilla muestra dos áreas principales:

- **Sitio local:** archivos y carpetas de la notebook Windows.
- **Sitio remoto:** archivos y carpetas accesibles en la Raspberry con la cuenta autenticada.

Para transferir un archivo se puede arrastrar desde uno de los paneles hacia el otro.

---

# Pruebas de funcionamiento

## Visualización y lectura de un archivo del servidor

Con FileZilla conectado, en el **sitio remoto** se ingresó a la carpeta `grupo3` y dentro de ella a la carpeta `ftp`, donde se encontró el archivo `prueba.txt` creado previamente desde la terminal de la Raspberry.

![FileZilla conectado: carpeta ftp con prueba.txt](imagenes/ftp_6.jpeg)

> **Figura 6:** Captura tomada en la notebook. FileZilla muestra las dos interfaces: el **sitio local** y el **sitio remoto**. En este último se navegó hasta `grupo3/ftp`, donde aparece el archivo `prueba.txt`.

Luego se abrió el archivo en la notebook para comprobar que su contenido podía leerse correctamente. Al ser un archivo de formato `.txt`, se muestra únicamente texto, y contiene la frase escrita desde la Raspberry:

![Contenido de prueba.txt abierto en la notebook](imagenes/ftp_7.jpeg)

> **Figura 7:** Contenido de `prueba.txt` visto desde la notebook: `Prueba del servidor FTP - Grupo 3`.

Esta prueba fue **exitosa**: el archivo alojado en el servidor pudo visualizarse desde el cliente y su contenido coincidió con el creado en la Raspberry. Esto confirmó que el cliente tenía acceso de lectura y que la conexión de datos FTP funcionaba.

---

## Subida de un archivo al servidor

Para comprobar la operación inversa, se trabajó en el **sitio local** ubicado en `C:\Users\Usuario\Downloads\`, donde se encontraba un archivo llamado `holarasp.txt`. Este archivo se transfirió hacia la carpeta `ftp` del sitio remoto.

![holarasp.txt en el sitio local y en el sitio remoto](imagenes/ftp_8.jpeg)

> **Figura 8:** Nuevamente en FileZilla. En el sitio local (`C:\Users\Usuario\Downloads\`) aparece `holarasp.txt`, y en el sitio remoto, dentro de la carpeta `ftp`, el archivo también aparece, lo que demuestra que la subida se realizó.

Para confirmar el resultado desde el lado del servidor, se listó el contenido del directorio directamente en la Raspberry:

```bash
ls -l ~/ftp
```

### Explicación del comando

- `ls`: lista el contenido de un directorio.
- `-l`: muestra detalles de cada archivo, incluidos tamaño, permisos, propietario y fecha de modificación.
- `~/ftp`: directorio utilizado para las transferencias.

![Contenido de la carpeta ftp desde la terminal de la Raspberry](imagenes/ftp_9.jpeg)

> **Figura 9:** Ejecución de `ls -l ~/ftp` en el monitor de la Raspberry, donde se ven los dos archivos presentes en el servidor: `prueba.txt` y `holarasp.txt`.

La subida fue **exitosa**: el archivo enviado desde Windows quedó almacenado en la Raspberry. Con ello se comprobó que la configuración del servidor permitía escribir en el directorio de destino con los permisos del usuario utilizado.

---

# Dificultades encontradas y soluciones

Durante la práctica se presentaron varios problemas que permitieron comprender mejor la relación entre la red y el servicio FTP.

| Dificultad | Causa o diagnóstico | Solución aplicada |
|---|---|---|
| `hostname -I` no mostraba ninguna IP. | La interfaz Ethernet no tenía una dirección asignada. | Se inspeccionó la interfaz y se asignó manualmente la IP de la LAN con `ip addr add`. |
| `eth0` aparecía como `NO-CARRIER`. | No se detectaba enlace físico por Ethernet. | Se revisó la conexión al switch hasta obtener `LOWER_UP`. |
| La dirección `192.168.3.1` desaparecía de `eth0`. | La IP se había configurado de forma temporal y se perdía al cambiar el estado de la conexión. | Se volvió a asignar con `ip addr add` y se verificó con `ip -4 addr show eth0` antes de cada prueba. |
| FileZilla mostraba tiempo de espera agotado. | La conexión del cliente no estaba correctamente completada. | Se revisó que el servicio estuviera activo y el puerto 21 en escucha, y se indicó el puerto `21` en FileZilla, logrando conectar. |
| Se creó una carpeta anidada innecesaria con `mkdir -p ~/home/grupo3/ftp`. | `~` ya equivale a `/home/grupo3`, por lo que la ruta quedó duplicada. | Se utilizó la carpeta `~/ftp`, que es la que se muestra en FileZilla dentro de `grupo3/ftp`. |

Estas dificultades demostraron que un servicio puede estar en estado **`active (running)`** y, sin embargo, no ser accesible si hay problemas de conectividad física, direccionamiento o configuración del cliente. Por eso es importante comprobar cada capa de forma ordenada: enlace Ethernet, IP, puerto TCP, autenticación y transferencia de archivos.

---

# Consideraciones para futuras demostraciones

La práctica quedó funcional, pero hay aspectos que conviene tener en cuenta antes de volver a encender los equipos:

1. **IP temporal:** la dirección `192.168.3.1/24` fue asignada con `ip addr add`, por lo que podría perderse al reiniciar o al activar otro perfil de red. Para una configuración definitiva se deberá establecer de forma persistente en el administrador de red correspondiente.
2. **Servidor DHCP:** durante la prueba FTP se mantuvo detenido `isc-dhcp-server`. Si se desea volver a utilizar la asignación automática en la LAN aislada del grupo, deberá recuperarse la IP del servidor, comprobar la configuración DHCP e iniciar el servicio únicamente cuando no esté conectado a la red institucional.
3. **Verificación tras reiniciar:** después de un reinicio conviene comprobar el estado con `sudo systemctl status vsftpd` y la IP con `ip -4 addr show eth0`.
4. **Seguridad:** el FTP sin cifrado solo es apropiado para esta práctica controlada. No se deben reutilizar contraseñas importantes ni exponer el servicio a Internet.
5. **Configuración de Windows:** si se estableció una IP manual para la prueba, deberá restaurarse DHCP automático al volver a otra red que lo requiera.

---

# Conclusión

En esta entrega se logró **instalar, configurar y poner en funcionamiento un servidor FTP mediante `vsftpd` en una Raspberry Pi**, integrándolo a la red LAN del grupo 3, `192.168.3.0/24`.

Primero se actualizaron los paquetes y se instaló `vsftpd`. Se modificó el archivo `/etc/vsftpd.conf` para impedir el acceso anónimo, permitir usuarios locales y habilitar las operaciones de escritura, y luego se reinició el servicio y se verificó que estuviera activo y escuchando en el puerto TCP 21. Además, se creó una carpeta `ftp` con un archivo de prueba para comprobar el intercambio de información.

Para realizar la demostración se conectaron la Raspberry y una notebook al switch del grupo, con la IP `192.168.3.1/24` en la Raspberry y `192.168.3.2/24` en Windows. Tras resolver los problemas de enlace físico y de asignación temporal de la dirección, se estableció la conexión desde **FileZilla Client** y se comprobó la **visualización y lectura** de `prueba.txt` y la **subida** de `holarasp.txt`, que luego se confirmó desde la terminal de la Raspberry con `ls -l ~/ftp`.

Como aprendizaje principal, se comprendió la diferencia entre el **servidor** que ofrece los archivos y el **cliente** que los solicita, el papel de TCP y del puerto 21, la importancia de la autenticación y los permisos, y la necesidad de contar con una red correctamente configurada antes de utilizar cualquier servicio. También se observó que FTP puede funcionar dentro de una LAN sin acceso a Internet y que la transferencia de archivos requiere tanto conectividad de red como una configuración adecuada del servidor y del cliente.
