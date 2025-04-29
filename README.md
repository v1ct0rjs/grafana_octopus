# Stack de Monitoreo con Grafana, InfluxDB 2 y Telegraf (Docker Compose)

## Descripción general 

Este proyecto despliega un **stack de monitoreo** completo utilizando **Grafana**, **InfluxDB 2** y **Telegraf** mediante **Docker Compose**. El objetivo es proporcionar una solución lista para usar para la monitorización de infraestructura: métricas de CPU, memoria RAM, uso de disco, tráfico de red, y métricas de contenedores Docker, tanto en entornos locales como en servidores remotos.

Grafana ofrece la interfaz web para visualizar las métricas en forma de *dashboards* interactivos. InfluxDB 2 actúa como base de datos de series temporales donde se almacenan los datos de monitoreo. Telegraf funciona como agente de recolección de métricas, obteniendo datos del sistema operativo (host) y de Docker, y enviándolos a InfluxDB. Todo el stack se orquesta con Docker Compose para facilitar su despliegue y gestión.

**Características principales:**

- Monitoreo de recursos del sistema (CPU, RAM, disco, red, procesos) y de contenedores Docker en tiempo real.
- Almacenamiento eficiente de métricas históricas en InfluxDB 2 (uso de *bucket* con retención configurable).
- Visualización mediante Grafana: paneles personalizables y posibilidad de alertas sobre las métricas.
- Despliegue sencillo con Docker Compose, apto para entornos de desarrollo (local) o producción (servidor remoto).
- Fácil extensión para monitorear múltiples servidores (instalando agentes Telegraf adicionales apuntando al mismo InfluxDB, si se desea).

## Estructura del proyecto 

El repositorio está organizado de la siguiente manera:

```
├── docker-compose.yml          # Definición de los servicios Docker (Telegraf, InfluxDB, Grafana)
├── .env.example                # Archivo de ejemplo de configuración de variables de entorno (.env)
├── telegraf/
│   └── telegraf.conf           # Configuración principal de Telegraf (inputs, outputs)
├── grafana/                    # (Opcional) Carpeta para datos/config de Grafana (p.ej. provisioning)
├── influxdb/                   # (Opcional) Carpeta para datos/config adicionales de InfluxDB
├── data/                       # (Opcional) Directorio para persistencia manual de datos (si no se usan volumes de Docker)
└── README.md                   # Documentación (este archivo)
```

> **Nota:** En el archivo `docker-compose.yml` se definen *volúmenes* Docker nombrados para persistir los datos:
>
> - `influxdb_data` – almacena los datos de InfluxDB (bases de datos, buckets, etc. en `/var/lib/influxdb2` dentro del contenedor).
> - `grafana_data` – almacena los datos de Grafana (configuración interna, dashboards, usuarios, etc. en `/var/lib/grafana` dentro del contenedor).

Mantener los datos en volúmenes asegura que no se pierdan las métricas ni la configuración al reiniciar o actualizar los contenedores.

## Requisitos previos 

Antes de desplegar el stack, asegúrate de contar con los siguientes requisitos en la máquina host:

- **Docker** instalado. Se recomienda Docker Engine 20.x o superior, funcionando en Linux (también compatible con Docker Desktop en Windows/Mac).
- **Docker Compose** instalado. Puedes usar Docker Compose v2 (plugin integrado de Docker) o Compose v1.29+ standalone. Asegúrate de poder ejecutar `docker compose` o `docker-compose`.
- **Recursos mínimos del sistema:** Para un monitoreo básico, se sugieren al menos 1 CPU y 1~2 GB de RAM libres. En entornos de producción o con muchas métricas/hosts monitorizados, reserva más recursos (ej. 2+ CPU, 4+ GB RAM) para un rendimiento fluido.
- **Puertos disponibles:** por defecto Grafana usará el puerto `3000` y InfluxDB el `8086` en el host. Verifica que estos puertos no estén en uso o modifica la configuración si es necesario.
- **Conexión de red:** en caso de despliegue remoto, se necesita acceso a la máquina (por SSH para configuración) y abrir en el firewall los puertos necesarios (p. ej. 3000/tcp para Grafana). Se recomienda **no** exponer InfluxDB (`8086`) abiertamente a Internet a menos que sea necesario y esté adecuadamente asegurado.

## Configuración del archivo `.env` ️

Todas las credenciales y parámetros principales del stack se gestionan mediante el archivo `.env`. Antes de levantar los servicios, copia el archivo de ejemplo proporcionado (`.env.example`) a un nuevo archivo `.env` y edítalo con los valores apropiados. A continuación se explican las variables de entorno utilizadas:

- **`INFLUXDB_ADMIN_USER`** – Nombre de usuario para la cuenta admin inicial de InfluxDB. *Ejemplo:* `admin` o un nombre de tu elección.
- **`INFLUXDB_ADMIN_PASSWORD`** – Contraseña para el usuario admin de InfluxDB. Debe ser fuerte especialmente si InfluxDB estará accesible remotamente.
- **`INFLUXDB_ORG`** – Nombre de la **organización** en InfluxDB 2. Las organizaciones en InfluxDB agrupan recursos; puedes poner el nombre de tu empresa, proyecto o simplemente `default-org`.
- **`INFLUXDB_BUCKET`** – Nombre del **bucket** de InfluxDB donde se almacenarán las métricas de Telegraf. Un *bucket* es similar a una base de datos o esquema de datos de series temporales. Ej: `telegraf_metrics` o `monitoring`.
- **`INFLUXDB_TOKEN`** – Token de autenticación de InfluxDB con privilegios de *admin*. Este token se usará para que Telegraf pueda escribir datos en InfluxDB y para que Grafana lea datos. Puedes generar uno manualmente, pero lo normal es definirlo aquí para la inicialización. **Mantén este token en secreto.**
- **`INFLUXDB_RETENTION`** – (Opcional) Período de retención de datos del bucket de InfluxDB, en tiempo. Por ejemplo `30d` para 30 días. Si no se especifica, el bucket tendrá retención infinita (los datos se conservan hasta que se borran manualmente).
- **`GRAFANA_ADMIN_USER`** – Usuario administrador inicial de Grafana. Por defecto suele ser `admin`, pero puedes cambiarlo.
- **`GRAFANA_ADMIN_PASSWORD`** – Contraseña para el usuario admin de Grafana. Si usas los valores por defecto (`admin/admin`), Grafana te pedirá cambiarla en el primer inicio de sesión. Es recomendable establecer aquí una contraseña robusta para automatizar ese paso.
- **`HOSTNAME`** – (Opcional) Nombre del host a usar para identificar las métricas del sistema. Si no se especifica, Telegraf usará el ID del contenedor Docker como hostname en InfluxDB, lo cual puede ser poco descriptivo. Puedes poner el hostname real del servidor (ej. `mi-servidor-prod`) para que las series de métricas lleven ese identificador.

Una vez configurado `.env`, Docker Compose lo utilizará para inyectar estos valores en los contenedores. Esto facilita cambiar credenciales u otros parámetros sin modificar los archivos de configuración directamente.

> **Consejo de seguridad:** No compartas tu archivo `.env` con credenciales sensibles en repositorios públicos. Si vas a versionar el proyecto, mantén `.env` fuera del control de versiones (ya se incluye típicamente en `.gitignore`). Puedes compartir el `.env.example` (con valores ficticios) para que otros sepan cómo configurarlo.

## Configuración de Telegraf

Telegraf es el agente encargado de recopilar las métricas del sistema y enviarlas a InfluxDB. La configuración de Telegraf se encuentra en el archivo [`telegraf/telegraf.conf`](https://chatgpt.com/c/telegraf/telegraf.conf). En este proyecto, Telegraf está configurado para recopilar tanto **métricas del host** (sistema operativo) como de **contenedores Docker**:

- **Métricas del host:** Se usan los *input plugins* de Telegraf para CPU, memoria, disco, red, procesos, etc. Esto permite monitorear el uso de CPU por núcleo, porcentajes de RAM usada, espacio en discos y particiones, tráfico de red, carga del sistema, entre otros. Estos *plugins* (como `[[inputs.cpu]]`, `[[inputs.mem]]`, `[[inputs.disk]]`, `[[inputs.net]]`, etc.) están habilitados en el `telegraf.conf` incluido.
- **Métricas de Docker:** Se utiliza el plugin `[[inputs.docker]]` de Telegraf para recoger estadísticas de los contenedores Docker ejecutándose en el host (por ejemplo: uso de CPU y memoria por contenedor, I/O de disco, tráfico de red por contenedor, etc.). Gracias a esto, puedes visualizar el rendimiento de los propios contenedores de este stack u otros contenedores que se estén ejecutando en el mismo host.

Para permitir que el contenedor de Telegraf acceda a la información del host y de Docker, en el `docker-compose.yml` se han incluido las siguientes configuraciones especiales:

- **Montaje del socket de Docker:** El archivo `/var/run/docker.sock` del host se monta dentro del contenedor Telegraf (en la misma ruta). Esto permite que Telegraf se comunique con la API de Docker del host y lea métricas de los contenedores. *(El montaje se realiza en modo lectura `:ro` para mayor seguridad.)*
- **Montaje de sistemas de archivos del host:** Se montan directorios especiales del host dentro del contenedor (`/proc`, `/sys` e incluso `/etc` del host, bajo una ruta prefijada como `/rootfs`). De esta manera, Telegraf puede leer la información del sistema del host real. Además, se pasan variables de entorno `HOST_PROC`, `HOST_SYS` y `HOST_ETC` al contenedor Telegraf para indicar a los plugins de sistema de Telegraf que usen esas rutas montadas. Por ejemplo, el plugin de CPU leerá `HOST_PROC` (que apuntará a `/rootfs/proc`) en lugar de `/proc` dentro del contenedor, consiguiendo así las estadísticas del *host* y no solo del contenedor.

Ejemplo de configuración relevante en `docker-compose.yml` para Telegraf:

```yaml
services:
  telegraf:
    image: telegraf:latest               # Usamos la imagen oficial de Telegraf
    hostname: ${HOSTNAME}                # Asignar un nombre de host reconocible para las métricas
    environment:
      - HOST_PROC=/rootfs/proc
      - HOST_SYS=/rootfs/sys
      - HOST_ETC=/rootfs/etc
      - INFLUX_TOKEN=${INFLUXDB_TOKEN}
      - INFLUX_ORG=${INFLUXDB_ORG}
      - INFLUX_BUCKET=${INFLUXDB_BUCKET}
    volumes:
      - ./telegraf/telegraf.conf:/etc/telegraf/telegraf.conf:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /proc:/rootfs/proc:ro
      - /sys:/rootfs/sys:ro
      - /etc:/rootfs/etc:ro
    depends_on:
      - influxdb
    restart: always
```

En el fragmento anterior, además de los montajes, también observamos que se pasa a Telegraf el token, organización y bucket de InfluxDB mediante variables de entorno. Esto corresponde a la configuración de la salida (output) de Telegraf hacia InfluxDB 2:

- En el `telegraf.conf`, la sección `[[outputs.influxdb_v2]]` está configurada para enviar datos a la URL `http://influxdb:8086` (uso del nombre de servicio Docker de InfluxDB), utilizando el **token**, **org** y **bucket** definidos en las variables de entorno. De este modo, Telegraf autentica sus escrituras en InfluxDB usando el token de admin que configuramos en `.env`.

Telegraf está prácticamente listo desde el inicio con la configuración proporcionada. No obstante, puedes personalizar el archivo `telegraf.conf` para añadir/quitar métricas según tus necesidades:

- Para habilitar métricas adicionales (por ejemplo, *inputs* de sensores, SNMP, etc.), añade sus configuraciones correspondientes en el archivo.

- Si modificas la configuración, guarda los cambios y luego reinicia el contenedor Telegraf para aplicar la nueva configuración:

  ```bash
  docker compose restart telegraf
  ```

  (o `docker-compose restart telegraf` dependiendo de tu versión de Compose).

## Configuración de InfluxDB 2 

El servicio **InfluxDB 2** se configura automáticamente al iniciarse gracias a las variables de entorno proporcionadas. En el `docker-compose.yml` se utilizan las variables definidas en el archivo `.env` para inicializar InfluxDB en *modo setup*. Los principales parámetros de inicialización son:

- **Organización (`INFLUXDB_ORG`):** se creará una organización con este nombre dentro de InfluxDB.
- **Bucket (`INFLUXDB_BUCKET`):** se creará un bucket de datos con este nombre, dentro de la organización anterior. Telegraf enviará aquí todas las métricas. (Si está definida la variable `INFLUXDB_RETENTION`, se aplicará como política de retención de datos del bucket; de lo contrario será de retención ilimitada).
- **Usuario administrador (`INFLUXDB_ADMIN_USER` y `INFLUXDB_ADMIN_PASSWORD`):** se creará un usuario admin inicial con estas credenciales, que tendrá acceso completo a la instancia de InfluxDB.
- **Token de autenticación (`INFLUXDB_TOKEN`):** se generará (o fijará) un token de acceso con privilegios administrativos para la organización configurada. Este token es crucial ya que InfluxDB 2.x **requiere** autenticación token para cualquier acceso (lectura/escritura de datos, consultas, etc.). El token configurado se usará en Telegraf y luego en Grafana para conectarse a InfluxDB.

Dentro de Docker Compose, esto se logra mediante variables como `DOCKER_INFLUXDB_INIT_MODE=setup`, `DOCKER_INFLUXDB_INIT_USERNAME`, `DOCKER_INFLUXDB_INIT_PASSWORD`, `DOCKER_INFLUXDB_INIT_ORG`, `DOCKER_INFLUXDB_INIT_BUCKET` y `DOCKER_INFLUXDB_INIT_ADMIN_TOKEN`, que la imagen oficial de InfluxDB utiliza en su primer arranque para autoconfigurarse. Estas a su vez toman los valores de las variables que definimos en `.env`.

**Persistencia de datos:** El contenedor de InfluxDB monta un volumen llamado `influxdb_data` en la ruta de datos `/var/lib/influxdb2`. Esto significa que todos los datos (series temporales, metadata, etc.) residen en ese volumen Docker. Si detienes o eliminas el contenedor, los datos permanecerán en el volumen y se reutilizarán cuando lances un nuevo contenedor (p. ej. al actualizar la imagen de InfluxDB).

**Acceso a InfluxDB:** InfluxDB expone una interfaz HTTP (API y UI web) en el puerto `8086`. En este stack, ese puerto está mapeado al host, por lo que puedes acceder a la interfaz web de InfluxDB 2 si la necesitas, visitando `http://<host>:8086` en un navegador. Allí podrías usar la GUI de InfluxDB para explorar datos o administrar tokens/usuarios. Las credenciales de acceso serán el usuario y password definidos (o directamente el token). *Nota:* La interfaz web de InfluxDB no es necesaria para el funcionamiento normal del stack (ya que Grafana se encarga de la visualización), pero está disponible. Por seguridad, si este servicio se despliega en un servidor remoto, considera restringir el acceso a este puerto (por firewall o configurando Docker Compose para que solo escuche en localhost) si no planeas usar la UI de InfluxDB.

## Configuración de Grafana 🖥️

Grafana es el componente de visualización. En el stack se configura con la imagen oficial de Grafana (versión open source; opcionalmente podrías usar la Enterprise si lo necesitas). La configuración principal de Grafana en este contexto es sencilla:

- Se crean un usuario y contraseña de administrador inicial usando las variables `GRAFANA_ADMIN_USER` y `GRAFANA_ADMIN_PASSWORD` de `.env`. Por defecto, si no se cambian, Grafana crea el usuario `admin` con contraseña `admin` y obliga a cambiarla al primer acceso. Nuestra configuración permite establecer una contraseña desde el inicio para automatizar este paso y mejorar la seguridad.
- Grafana usa un volumen `grafana_data` montado en `/var/lib/grafana` para persistir su base de datos interna (que incluye configuraciones, dashboards creados, cuentas de usuarios, etc.). Así, cualquier dashboard que crees o ajuste que hagas en la interfaz se mantendrá aunque el contenedor se reinicie o actualice.
- El puerto web de Grafana (3000 dentro del contenedor) está mapeado al puerto `3000` del host. Por lo tanto, para acceder a Grafana deberás navegar a `http://<host>:3000` (por ejemplo `http://localhost:3000` si es local, o la IP/dominio de tu servidor remoto).

Una vez Grafana esté corriendo, podrás ingresar con las credenciales de admin configuradas. Veremos en las siguientes secciones cómo agregar la fuente de datos de InfluxDB y dashboards.

> **Nota:** La configuración por defecto de Grafana (a través de variables `GF_*`) se puede ampliar según necesites (por ejemplo, para permitir el registro de usuarios, deshabilitar registro anónimo, configurar un SMTP para alertas por correo, etc.). Este README cubre la configuración básica necesaria para ponerlo en marcha. Revisa la documentación oficial de Grafana para opciones avanzadas.

## Añadir InfluxDB 2 como fuente de datos en Grafana 

Una vez que Grafana está en funcionamiento, el siguiente paso es conectar Grafana con la base de datos de InfluxDB para poder consultar las métricas. Esto se realiza añadiendo **InfluxDB como una "fuente de datos" (Data Source)** en Grafana:

1. **Ingresar a Grafana:** Abre tu navegador en la URL de Grafana (por ejemplo, `http://localhost:3000`). Inicia sesión con tu usuario admin y contraseña.
2. **Ir a configuración de fuentes de datos:** En el menú lateral izquierdo de Grafana, haz clic en el icono de engranaje (**Configuration**) y selecciona **Data Sources**.
3. **Añadir nueva fuente de datos:** Haz clic en el botón **“Add data source” (Agregar fuente de datos)**.
4. **Seleccionar InfluxDB:** Grafana mostrará una lista de tipos de datos soportados. Busca y selecciona **InfluxDB**.
5. **Configurar la conexión a InfluxDB 2.x:** En la pantalla de configuración de la fuente de datos InfluxDB, completa los campos necesarios:
   - **URL:** `http://influxdb:8086`
      (Grafana está ejecutándose en Docker junto a InfluxDB, por lo que puede usar el nombre de host `influxdb` del contenedor. Si Grafana estuviera fuera del mismo entorno Docker, aquí usarías la IP o host real del servidor y el puerto 8086).
   - **Organization:** ingresa el nombre de la organización que configuraste (`INFLUXDB_ORG` en tu .env).
   - **Authentication > Token:** pega el token de acceso de InfluxDB. Usa el mismo valor que `INFLUXDB_TOKEN` de tu .env (Grafana usará este token para consultar datos).
   - **Default Bucket:** especifica el nombre del bucket por defecto, es decir, el bucket donde están las métricas (`INFLUXDB_BUCKET` de tu configuración).
   - **InfluxQL/Flux:** En la sección "Query Language", selecciona **Flux** (InfluxDB 2.x utiliza Flux como lenguaje de consultas por defecto). Algunas versiones recientes de Grafana detectan automáticamente que es 2.x al usar token.
   - Los demás campos (como "Database", "User" o "Password") **no** se usan para InfluxDB 2 con Flux, por lo que puedes dejarlos vacíos.
6. **Guardar y probar:** Haz clic en **Save & Test** (Guardar y probar). Grafana intentará conectar con InfluxDB. Si la configuración es correcta, deberías ver un mensaje de éxito como *"Data source is working"*.

Tras agregar la fuente de datos, Grafana ya está listo para usar InfluxDB como origen de datos de métricas. En caso de error:

- Verifica que el contenedor de InfluxDB esté corriendo (`docker compose ps`) y que el token, org y bucket sean correctos.
- Asegúrate de que Grafana puede resolver el host `influxdb` (dentro del docker-compose ya están en la misma red, debería funcionar). Si Grafana estuviera fuera del entorno Docker Compose, usa la dirección IP o hostname correcto donde esté InfluxDB.

## Importar dashboards de Telegraf en Grafana 

Con la fuente de datos configurada, el siguiente paso es crear o importar *dashboards* en Grafana para visualizar las métricas recolectadas por Telegraf. Grafana permite diseñar paneles desde cero, pero para facilidad podemos **importar dashboards prediseñados** específicamente hechos para Telegraf + InfluxDB:

1. En Grafana, ve al menú lateral y selecciona **Dashboards -> Manage (Administrar)**. Luego pulsa el botón **“Import” (Importar)**.
2. Grafana te pedirá un dashboard a importar. Tienes varias opciones:
   - **Importar mediante ID desde Grafana.com:** Grafana tiene un repositorio de dashboards comunitarios. Puedes pegar el ID de un dashboard público y Grafana lo descargará. Por ejemplo:
     - *Dashboard de métricas de Docker (Telegraf + InfluxDB 2.x):* ID `19889`. Este dashboard muestra el uso de CPU, memoria, red y I/O de contenedores Docker recopilados por Telegraf.
     - *Dashboard de métricas de sistema (Telegraf System Metrics):* ID `5955`. Muestra gráficos de uso de CPU, carga, memoria, swap, etc. de un sistema Linux usando datos de Telegraf.
   - **Importar desde archivo JSON:** Si en el repositorio u otro lugar tienes un archivo `.json` exportado de un dashboard de Grafana, puedes subirlo aquí (o copiar/pegar el JSON).
3. Después de proporcionar el ID o subir el JSON, haz clic en **Load (Cargar)**. Se mostrará una vista previa de la información del dashboard.
4. Selecciona la **fuente de datos** correcta: Grafana preguntará a qué fuente de datos deben asociarse las gráficas del dashboard importado. Selecciona la fuente de datos de InfluxDB que creaste anteriormente.
5. Haz clic en **Import** para finalizar.

Grafana creará el nuevo dashboard y te dirigirá a él. Deberías ver ya las visualizaciones llenándose con datos de tu sistema y/o contenedores Docker, en función de las métricas que Telegraf esté enviando.

Puedes repetir la importación con distintos dashboards de la comunidad o crear los tuyos propios:

- Para crear un dashboard personalizado, ve a **Dashboards -> New**. Agrega paneles (*Panels*) con las visualizaciones que desees y construye las consultas usando Flux hacia InfluxDB.
- Algunos dashboards útiles para este stack:
  - "InfluxDB 2.x Telegraf Docker Dashboard" (ID 19889, mencionado arriba) para vista de contenedores Docker.
  - "Telegraf System Metrics" (ID 5955) para vista de métricas de sistema (CPU, RAM, etc.).
  - "Docker Host & Container Overview" (ID 14280) u otros similares, que combinan host y contenedores.
- Si importas múltiples dashboards, organiza y renómbralos como te convenga. Los dashboards importados se pueden editar y adaptar a tus necesidades (por ejemplo, quitar secciones no relevantes, ajustar alertas, etc.).

> **Nota:** Los IDs de dashboards pueden cambiar o ser reemplazados con el tiempo en Grafana.com. Si alguno no funciona, busca en la página oficial de Grafana Labs (https://grafana.com/dashboards) por palabras clave como *Telegraf*, *Docker*, *InfluxDB 2* para encontrar dashboards actualizados creados por la comunidad.

## Instrucciones para levantar el stack 

A continuación se describen los pasos paso a paso para desplegar el stack de monitoreo usando Docker Compose. Suponiendo que ya has clonando o descargado este proyecto en tu máquina:

1. **Clonar el repositorio (si no lo has hecho):**

   ```bash
   git clone https://github.com/tu-usuario/tu-repo-monitoreo.git
   cd tu-repo-monitoreo
   ```

   *Alternativamente, descarga el archivo ZIP del proyecto y extráelo.*

2. **Configurar variables de entorno:**
    Copia el archivo `.env.example` como `.env`:

   ```bash
   cp .env.example .env
   ```

   Abre el archivo `.env` en un editor de texto y edita los valores según lo explicado en la sección de configuración (.env). Especialmente, establece contraseñas seguras y un token válido.

3. **Revisar puertos (opcional):**
    Por defecto Grafana usará el puerto 3000 y InfluxDB el 8086 en tu máquina host. Si esos puertos estuvieran ocupados o quieres cambiarlos, edita el `docker-compose.yml` y modifica las secciones de `ports` para grafana o influxdb (formato `host:contenedor`). Por ejemplo, para acceder Grafana en el puerto 80 podrías poner `80:3000`.

4. **Levantar los servicios con Docker Compose:**
    Ejecuta el comando en la raíz del proyecto:

   ```bash
   docker compose up -d
   ```

   *(Si tu versión de Docker Compose es antigua, usa `docker-compose up -d`.)*
    Esto descargará las imágenes de Grafana, InfluxDB y Telegraf (si no las tienes ya) y creará los contenedores definidos. La opción `-d` los ejecuta en segundo plano (modo *detached*).

5. **Verificar que los contenedores están corriendo:**

   ```bash
   docker compose ps
   ```

   Deberías ver listados `telegraf`, `influxdb` y `grafana` con estado "Up". Si alguno salió con error (`Exit`), ejecuta `docker compose logs --tail=100 <servicio>` para ver los últimos logs y determinar la causa.

6. **Esperar la inicialización de InfluxDB:**
    El primer arranque de InfluxDB 2 puede tardar unos segundos extra ya que realiza el proceso de *setup* inicial usando las variables de entorno. Puedes monitorear los logs con:

   ```bash
   docker compose logs -f influxdb
   ```

   Espera a ver un mensaje como "Welcome to InfluxDB 2.0" o confirmación de que el usuario/org se crearon. Una vez inicializado, debería quedarse ejecutando esperando conexiones.

7. **Acceder a la interfaz de Grafana:**
    Abre tu navegador web e ingresa la dirección `http://localhost:3000` (o la URL/IP de tu servidor remoto seguido de `:3000`). Debería cargar la pantalla de login de Grafana. Ingresa el usuario y contraseña que configuraste (`GRAFANA_ADMIN_USER` / `GRAFANA_ADMIN_PASSWORD`). Si usaste los valores por defecto (admin/admin), Grafana te pedirá inmediatamente cambiar la contraseña por una nueva.

8. **Configurar la fuente de datos InfluxDB en Grafana:**
    Sigue los pasos de la sección anterior ("Añadir InfluxDB como fuente de datos") para conectar Grafana a InfluxDB usando el token y bucket configurados.

9. **Importar o crear dashboards:**
    Importa algún dashboard de ejemplo o crea uno nuevo para verificar que los datos están llegando. En pocos minutos deberías ver las métricas poblándose. Asegúrate de seleccionar el rango de tiempo adecuado en la esquina superior derecha de Grafana (por ejemplo "Last 5 minutes" o "Last 1 hour") y que la actualización esté en automático (por ejemplo cada 5s o 10s) para ver datos en tiempo real.

Si todos los pasos son correctos, habrás desplegado exitosamente la plataforma de monitoreo.  Desde ahora, cada vez que inicies Docker Compose, Telegraf comenzará a enviar métricas a InfluxDB y podrás visualizarlas en Grafana.

> **Sugerencia:** Puedes agregar más agentes Telegraf (en otras máquinas) apuntando al mismo servidor InfluxDB para centralizar el monitoreo de múltiples hosts. Solo asegúrate de utilizar el token y URL adecuados de tu servidor InfluxDB en la configuración de esos agentes adicionales. Las métricas de diferentes hosts se diferenciarán por el `hostname` que reporte cada Telegraf.

## Ejemplo: Despliegue en un servidor remoto 

El stack está diseñado para funcionar igualmente en un servidor remoto (por ejemplo un VPS o instancia en la nube) que en local. Supongamos que quieres desplegarlo en un servidor Linux remoto para monitorear ese servidor. A continuación algunas pautas y consideraciones:

**1. Preparar el entorno remoto:**
 Asegúrate de instalar Docker y Docker Compose en el servidor tal como lo harías localmente. Puedes usar herramientas como *ssh* o una herramienta CI/CD para copiar el contenido del proyecto al servidor. Por ejemplo, desde tu máquina local podrías hacer:

```bash
scp -r ./tu-repo-monitoreo usuario@mi.servidor.com:/home/usuario/
```

Luego conectar por SSH y navegar al directorio.

**2. Configurar `.env` con valores seguros:**
 En entornos remotos es **crítico** usar contraseñas fuertes y un token difícil de adivinar. Edita `.env` en el servidor y asegúrate de que `INFLUXDB_ADMIN_PASSWORD` y `GRAFANA_ADMIN_PASSWORD` sean robustos. Evita valores por defecto. Esto protegerá la base de datos y la interfaz de Grafana de accesos no deseados.

**3. Iniciar los contenedores en el servidor:**
 En el servidor remoto, ejecuta `docker compose up -d` igual que en local. Los contenedores iniciarán en segundo plano. Verifica con `docker compose ps` que están "Up".

**4. Configurar acceso a Grafana desde tu máquina local:**
 Por razones de seguridad, podrías decidir **no** exponer Grafana públicamente con una IP abierta. Una práctica recomendable es restringir Grafana al propio servidor y usar un túnel SSH o VPN para acceder de forma segura:

- Si decides exponer Grafana directamente, asegúrate de que el puerto 3000 esté permitido en el firewall del servidor (ej. usando `ufw allow 3000/tcp` en Ubuntu, o la consola de tu proveedor cloud). Luego podrás acceder via `http://<IP-del-servidor>:3000` desde tu navegador. *Considera habilitar HTTPS mediante un proxy inverso (como Nginx o Caddy) si va a estar expuesto a Internet.*
- Si **no** quieres exponerlo, puedes hacer un túnel SSH: por ejemplo, desde tu PC local ejecutar `ssh -L 3000:localhost:3000 usuario@mi.servidor.com` de modo que `http://localhost:3000` en tu navegador se reenvíe al servidor remoto de forma segura.
- Para InfluxDB (puerto 8086), normalmente **no es necesario** abrirlo al público. Grafana está corriendo en el mismo servidor y puede acceder a InfluxDB internamente. Si necesitas acceder al UI/API de InfluxDB remotamente, puedes aplicar técnicas similares (túnel SSH para 8086 o exponerlo temporalmente con protección).

**5. Revisar seguridad de Docker socket (importante):**
 Observa que estamos montando el socket Docker del host en Telegraf para leer métricas de contenedores. Esto implica que el contenedor Telegraf tiene acceso de lectura a Docker (lo cual en teoría podría explotarse si no se tiene cuidado). Asegúrate de confiar en la configuración y, si es un servidor multiusuario, considera limitar quién puede editar el stack. Alternativamente, podrías optar por no montar el socket en entornos donde la seguridad sea crítica y prescindir de métricas de contenedores (o usar métodos alternativos como la API remota de Docker con TLS).

**6. Operación remota:**
 Una vez funcionando, el uso es idéntico a local: ingresar a Grafana (vía web o túnel), agregar la fuente de datos InfluxDB (usando `http://influxdb:8086` ya que Grafana corre en el mismo docker-compose), importar dashboards, etc. El servidor remoto comenzará a reportar sus métricas.

**7. Actualizaciones y mantenimiento:**
 En un server remoto, para actualizar el stack puedes conectarte por SSH y hacer `git pull` (si clonaste el repo) para obtener cambios, luego `docker compose pull` para obtener nuevas versiones de las imágenes Docker, y reiniciar los servicios. Verifica periódicamente que los contenedores siguen corriendo (`docker compose ps`) y configura algún mecanismo de reinicio automático del servicio Docker en caso de reinicio de la máquina (muchas distros ya lo hacen; Docker Compose con restart:always manejará que los contenedores arranquen con el daemon de Docker).

**Resumen:** Este ejemplo muestra que el stack es portátil. Solo recuerda endurecer la seguridad en producción: usar credenciales fuertes, limitar el acceso a puertos, y mantener los servicios actualizados.

## Buenas prácticas de operación 

Para asegurar la continuidad del monitoreo y la integridad de los datos, se recomiendan las siguientes buenas prácticas durante la operación diaria del stack:

###  Backups periódicos de datos

- **InfluxDB:** Realiza copias de seguridad regulares de la base de datos de InfluxDB, especialmente si almacenas datos históricos valiosos. Puedes usar la herramienta de línea de comandos de InfluxDB 2 dentro del contenedor. Por ejemplo:

  ```bash
  # Crear un backup completo en /backups (monta un volumen/carpeta local previamente)
  docker exec influxdb influx backup /backups
  ```

  Este comando generará archivos con el contenido del bucket, que luego podrías restaurar en otra instancia si es necesario. Alternativamente, puedes simplemente conservar copias del volumen `influxdb_data` (aunque es menos práctico para restores puntuales).

- **Grafana:** Los dashboards y ajustes de Grafana se almacenan en el volumen `grafana_data` (por defecto en un archivo SQLite). Puedes respaldar este volumen copiándolo o usando comandos Docker. Por ejemplo:

  ```bash
  docker run --rm -v tu-proyecto_grafana_data:/grafana_data -v $(pwd):/backup alpine \
      tar czf /backup/grafana_backup.tar.gz -C /grafana_data . 
  ```

  (Este comando archivará el contenido del volumen Grafana en un tar.gz en tu directorio actual).

- **Configuraciones:** Mantén a salvo los archivos de configuración (`.env`, `telegraf.conf`, `docker-compose.yml`). Idealmente versionados (excepto las credenciales reales) para recuperar rápidamente el entorno ante cualquier problema.

###  Reinicios controlados

- Los contenedores están configurados con `restart: always`, por lo que Docker intentará mantenerlos corriendo. Aun así, puede haber ocasiones donde debas reiniciar manualmente alguno (por ejemplo, tras cambiar `telegraf.conf` o actualizar la imagen):
  - Para reiniciar un servicio individual: `docker compose restart grafana` (por ejemplo).
  - Para detener todo el stack temporalmente: `docker compose down` (ten en cuenta que esto detiene contenedores pero **no borra** volúmenes, por lo que los datos quedan intactos).
  - Para volver a levantarlo luego: `docker compose up -d` nuevamente.
- Si reinicias la máquina host, Docker normalmente arrancará los contenedores automáticamente al iniciar (debido al `restart: always`). Aun así, comprueba tras un reinicio que Grafana e InfluxDB estén operativos.
- En caso de cortes o errores de red, Telegraf tiene buffers internos y reintentos; cuando InfluxDB vuelva a estar disponible, tratará de enviar cualquier métrica pendiente. Sin embargo, para evitar pérdida prolongada de datos, intenta minimizar el tiempo que InfluxDB esté caído.

###  Actualizaciones de versiones

- **Actualizar Grafana/InfluxDB/Telegraf:** Regularmente salen nuevas versiones con mejoras y parches de seguridad. Para actualizar, puedes cambiar las etiquetas de versión en el `docker-compose.yml` (o usar `latest` si prefieres siempre la última) y luego ejecutar:

  ```bash
  docker compose pull
  docker compose up -d
  ```

  Esto descargará las nuevas imágenes y recreará los contenedores. Los datos y configuraciones persistentes se mantendrán gracias a los volúmenes. **Recomendación:** prueba las actualizaciones en un entorno controlado antes de aplicarlas a producción, especialmente en el caso de InfluxDB 2.x donde saltos de versión mayor podrían requerir pasos de migración.

- **Migraciones de InfluxDB:** Si en algún momento quisieras migrar de InfluxDB 2 a otra instancia o a InfluxDB Cloud, podrías usar las funciones de exportación (`influx export`) o replicación de InfluxDB. Considera estas opciones si tu stack crece y necesitas alta disponibilidad o escalado.

- **Monitorizar las actualizaciones:** Suscríbete a boletines o revisa changelogs de estos productos para enterarte de cambios relevantes (por ejemplo, Grafana v10, InfluxDB 3.x si aparece en el futuro, etc.).

###  Alertas y monitoreo proactivo

- **Alertas en Grafana:** Grafana permite crear **alertas** basadas en las métricas, que pueden notificarte por correo, Slack u otros canales cuando una métrica cruza un umbral. Aprovecha esta funcionalidad para configurar alarmas, por ejemplo:
  - Uso de CPU > 90% durante 5 minutos.
  - Espacio en disco quedando por debajo de 10% libre.
  - Contenedor crítico caído (podrías usar la métrica de *uptime* o similar). Estas alertas requieren configurar un *Notification Channel* (en Grafana v9+ se hacen mediante *Contact points* y *Alertmanager* interno). Puedes encontrar documentación en Grafana para habilitarlas. Esto eleva tu stack de monitoreo de solo visualización a monitoreo activo.
- **Alertas en InfluxDB:** InfluxDB 2 tiene su propio sistema de tareas, checks y notifications. Sin embargo, dado que Grafana ya está en uso, suele ser más sencillo centralizar las alertas allí. Si no, podrías explorar InfluxDB tasks para, por ejemplo, enviar un webhook o email cuando una condición se cumple.
- **Logs y métricas del stack:** No olvides monitorizar la salud del propio sistema de monitoreo:
  - Revisa ocasionalmente los *logs* de InfluxDB (`docker compose logs influxdb`) para asegurarte de que no haya errores (por ejemplo problemas de escritura, falta de espacio en disco, etc.).
  - Revisa logs de Telegraf por si hay plugins fallando o métricas que no se puedan recoger (Telegraf loguea si, por ejemplo, no puede leer del socket Docker).
  - Grafana usualmente es silencioso, pero sus logs pueden indicar si hubo intentos fallidos de login, etc.
  - Considera incluir a este mismo stack alguna métrica de *autocontrol*, por ejemplo, usando Telegraf inputs para Docker que monitoreen también el uso de recursos de InfluxDB y Grafana (ya que son contenedores Docker, Telegraf ya está capturando CPU/Mem de ellos). Podrías hacer un dashboard meta-monitoreo donde Grafana muestre si Grafana/Influx están usando mucha memoria, etc., cerrando el círculo de observabilidad.

Resumiendo, tratar el sistema de monitoreo con el mismo cuidado que a los sistemas productivos: backup, actualización y supervisión continua. Así te aseguras de que esté disponible cuando más lo necesites.

## Comandos útiles 

A continuación se enumeran algunos comandos útiles para gestionar y operar el stack de monitoreo. Ejecuta estos comandos en el directorio raíz del proyecto (donde está el `docker-compose.yml`):

- **Levantar el stack en segundo plano:**

  ```bash
  docker compose up -d
  ```

- **Detener todos los servicios:**

  ```bash
  docker compose down
  ```

  *(Esto no borra los volúmenes; los datos quedan guardados para el próximo arranque.)*

- **Reiniciar un servicio (contenedor) específico:**

  ```bash
  docker compose restart <servicio>
  ```

  Reemplaza `<servicio>` por `telegraf`, `influxdb` o `grafana` según necesites.

- **Ver el estado de los contenedores:**

  ```bash
  docker compose ps
  ```

  Lista los contenedores con sus puertos, estado y comandos.

- **Ver logs en tiempo real:**

  ```bash
  docker compose logs -f <servicio>
  ```

  Útil para depurar. Por ejemplo, `docker compose logs -f telegraf` te mostrará qué está haciendo Telegraf (incluyendo errores de conexión, etc.). Sin `<servicio>`, mostrará logs de todos.

- **Ejecutar una shell dentro de un contenedor:**

  - InfluxDB: `docker compose exec influxdb /bin/sh`
     (Luego puedes usar el CLI `influx` dentro para, por ejemplo, listar buckets `influx bucket list` o verificar datos).
  - Grafana: `docker compose exec grafana /bin/bash`
     (Podrías usar herramientas como `grafana-cli` para instalar plugins, aunque es preferible hacerlo vía variables de entorno/ Dockerfile personalizado).
  - Telegraf: `docker compose exec telegraf /bin/sh`
     (No suele ser necesario, pero podrías comprobar archivos dentro o el estado del sistema visto desde el contenedor).

- **Actualizar imágenes a las últimas versiones disponibles:**

  ```bash
  docker compose pull
  docker compose up -d
  ```

  Esto descargará las últimas versiones de las imágenes (si no estás usando etiquetas fijas) y recreará los contenedores con ellas.

- **Comprobar versiones de cada componente:**

  - InfluxDB: `docker compose exec influxdb influx version`
  - Telegraf: `docker compose exec telegraf telegraf --version`
  - Grafana: `docker compose exec grafana grafana-server -v`
     Útil para registrar qué versión exacta está corriendo en caso de debug.

- **Gestión de volúmenes:**
   Si necesitas acceder a los datos en los volúmenes desde el host:

  ```bash
  docker volume ls        # listar volúmenes (deberían aparecer tuproyecto_influxdb_data, etc.)
  docker volume inspect tuproyecto_influxdb_data   # ver ruta en el host del volumen
  ```

  También puedes montar temporalmente el volumen en un contenedor auxiliar como se mostró en la sección de backups para inspeccionar su contenido.

- **Eliminar datos ( con precaución):**
   Si por alguna razón necesitas empezar desde cero (borrar datos y config):

  ```bash
  docker compose down -v
  ```

  La opción `-v` *eliminará* los volúmenes named asociados al compose (o sea, `influxdb_data` y `grafana_data`). **Advertencia:** esto borra irreversiblemente la base de datos de InfluxDB y la config de Grafana. Úsalo solo si realmente deseas resetear todo.

Estos comandos te ayudarán en la administración cotidiana. Recuerda que también puedes referirte a la documentación de Docker Compose para más opciones, y a la de cada componente (InfluxDB, Grafana, Telegraf) para comandos específicos dentro de ellos.

