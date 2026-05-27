# Automatización de despliegues con WLST

## Metadatos

| Campo            | Valor                                      |
|------------------|--------------------------------------------|
| **Duración**     | 75 minutos                                 |
| **Complejidad**  | Media                                      |
| **Nivel Bloom**  | Crear (Create)                             |
| **Herramienta**  | WLST (WebLogic Scripting Tool) / Jython 2.7 |
| **Prerequisito** | Lab 03-00-01 completado                    |

---

## Descripción General

En este laboratorio el alumno utilizará WLST como herramienta de scripting imperativo para automatizar tareas de administración en WebLogic 15. La práctica se divide en dos bloques: primero, el uso de WLST en **modo offline** para generar y modificar configuraciones de dominio sin necesidad de un servidor activo; segundo, el uso de WLST en **modo online** para conectarse al Administration Server del dominio `wdt_domain`, navegar el árbol de MBeans, desplegar una aplicación, crear un JDBC Data Source, gestionar el ciclo de vida de servidores y capturar métricas de runtime. Al finalizar, el alumno habrá construido un script de despliegue completo con validaciones, rollback y log de auditoría, y comprenderá cuándo elegir WLST frente a WDT o la REST API.

---

## Objetivos de Aprendizaje

- [ ] Ejecutar WLST en modo offline para crear y modificar configuraciones de dominio sin servidor activo.
- [ ] Conectarse al Administration Server usando WLST en modo online y navegar el árbol de MBeans con `cd()`, `ls()` y `get()`.
- [ ] Escribir scripts Jython para automatizar despliegues, creación de recursos JDBC y gestión del ciclo de vida de servidores.
- [ ] Implementar manejo de errores, rollback y logging de auditoría en scripts WLST listos para entornos de producción.
- [ ] Comparar WLST, WDT y REST API identificando el caso de uso óptimo para cada herramienta.

---

## Prerrequisitos

### Conocimientos
- Lab 03-00-01 completado: dominio `wdt_domain` operativo con Administration Server activo en el puerto `7001`.
- Conocimiento básico de Python/Jython: variables, funciones, bucles y manejo de excepciones (`try/except`).
- Comprensión de la arquitectura de MBeans de WebLogic: Administration MBeans (config) vs Runtime MBeans (runtime/domainRuntime).

### Acceso y Software
- Usuario `oracle` con acceso al sistema de ficheros del dominio.
- `$WL_HOME/common/bin/wlst.sh` accesible y ejecutable.
- Aplicación de muestra `sample-app.war` proporcionada por el instructor (ubicada en `/home/oracle/labs/lab04/`).
- Directorio de trabajo del laboratorio: `/home/oracle/labs/lab04/`.

---

## Entorno de Laboratorio

### Hardware Mínimo Recomendado

| Recurso      | Mínimo          | Recomendado     |
|--------------|-----------------|-----------------|
| RAM          | 8 GB            | 16 GB           |
| CPU          | 4 núcleos       | 8 núcleos       |
| Disco libre  | 20 GB           | 40 GB           |

### Software y Rutas Clave

| Componente            | Ruta / Valor de Referencia                        |
|-----------------------|---------------------------------------------------|
| JDK 21                | `/u01/oracle/jdk21`                               |
| Middleware Home       | `/u01/oracle/middleware`                          |
| WebLogic Home         | `/u01/oracle/middleware/wlserver`                 |
| Dominio base          | `/u01/domains/wdt_domain`                         |
| Dominio offline (lab) | `/u01/domains/lab04_domain`                       |
| WLST script           | `$WL_HOME/common/bin/wlst.sh`                     |
| Directorio del lab    | `/home/oracle/labs/lab04`                         |
| WAR de muestra        | `/home/oracle/labs/lab04/sample-app.war`          |
| Admin URL             | `t3://localhost:7001`                             |
| Credenciales          | `weblogic / Welcome1` *(solo entorno de lab)*     |

### Preparación del Entorno

Abre una terminal como usuario `oracle` y ejecuta los siguientes comandos de preparación:

```bash
# 1. Crear directorio de trabajo del laboratorio
mkdir -p /home/oracle/labs/lab04/scripts
mkdir -p /home/oracle/labs/lab04/logs
cd /home/oracle/labs/lab04

# 2. Verificar que el Administration Server de wdt_domain está activo
curl -s -o /dev/null -w "%{http_code}" \
  http://localhost:7001/weblogic/ready
# Resultado esperado: 200

# 3. Configurar variables de entorno
export JAVA_HOME=/u01/oracle/jdk21
export MW_HOME=/u01/oracle/middleware
export WL_HOME=$MW_HOME/wlserver
export PATH=$JAVA_HOME/bin:$WL_HOME/common/bin:$PATH

# 4. Verificar que wlst.sh es accesible
which wlst.sh || ls -la $WL_HOME/common/bin/wlst.sh

# 5. Crear archivo de propiedades de conexión (evitar contraseñas en scripts)
cat > /home/oracle/labs/lab04/env.properties << 'EOF'
admin.url=t3://localhost:7001
admin.user=weblogic
admin.pass=Welcome1
EOF
chmod 600 /home/oracle/labs/lab04/env.properties
```

> **Nota de seguridad:** El archivo `env.properties` contiene credenciales de laboratorio. En producción, usa `storeUserConfig`/`userKeyFile` o un gestor de secretos corporativo. **Nunca** uses estas credenciales fuera del entorno de práctica.

---

## Pasos del Laboratorio

---

### Parte 1 — WLST en Modo Offline

---

### Paso 1: Exploración Interactiva del Modo Offline

**Objetivo:** Familiarizarse con el prompt de WLST offline, los comandos de navegación básicos y la lectura de una plantilla de dominio.

**Instrucciones:**

1. Lanza WLST en modo interactivo:

```bash
wlst.sh
```

2. Observa el prompt `wls:/offline>`. Ejecuta los siguientes comandos de exploración:

```python
# Mostrar ayuda general
help('all')

# Listar comandos disponibles en modo offline
help('offline')

# Ver la versión de WLST / WebLogic
wlsVersion()
```

3. Carga la plantilla estándar de WebLogic y explora su estructura:

```python
# Cargar la plantilla base de WebLogic Server
import os
wl_home = os.environ.get('WL_HOME', '/u01/oracle/middleware/wlserver')
template_path = wl_home + '/common/templates/wls/wls.jar'
readTemplate(template_path)

# Navegar al nodo raíz y listar contenido
cd('/')
ls()

# Explorar el nodo Servers
cd('Servers')
ls()

# Ver propiedades del AdminServer
cd('AdminServer')
ls()
```

4. Cierra la plantilla sin escribir cambios y sal de WLST:

```python
closeTemplate()
exit()
```

**Salida Esperada:**

```
wls:/offline> ls()
dr--   AppDeployments
dr--   Clusters
dr--   JDBCSystemResources
dr--   JMSSystemResources
dr--   Servers
dr--   Security
...

wls:/offline/Servers/AdminServer> ls()
-rw-   ListenAddress
-rw-   ListenPort                           7001
-rw-   Name                                 AdminServer
...
```

**Verificación:** El prompt muestra `wls:/offline>` en todo momento. Si aparece `wls:/...>` con una URL, estás en modo online — sal con `exit()` y vuelve a lanzar `wlst.sh` sin argumentos.

---

### Paso 2: Crear un Dominio Offline con Script

**Objetivo:** Escribir y ejecutar un script Jython que cree el dominio `lab04_domain` a partir de la plantilla estándar, configure el AdminServer en un puerto alternativo y establezca credenciales de administrador.

**Instrucciones:**

1. Crea el script `/home/oracle/labs/lab04/scripts/01_create_domain_offline.py`:

```bash
cat > /home/oracle/labs/lab04/scripts/01_create_domain_offline.py << 'PYEOF'
# =============================================================
# Script: 01_create_domain_offline.py
# Descripcion: Crea lab04_domain en modo offline desde plantilla
# Uso: wlst.sh 01_create_domain_offline.py
# =============================================================
import os
import sys

# --- Parametros configurables ---
DOMAIN_NAME    = 'lab04_domain'
DOMAIN_PATH    = '/u01/domains/lab04_domain'
ADMIN_PORT     = 7101          # Puerto alternativo para no colisionar con wdt_domain
ADMIN_ADDRESS  = 'localhost'
ADMIN_USER     = 'weblogic'
ADMIN_PASSWORD = 'Welcome1'    # SOLO laboratorio - usar vault en produccion
WL_HOME        = os.environ.get('WL_HOME', '/u01/oracle/middleware/wlserver')
TEMPLATE_PATH  = WL_HOME + '/common/templates/wls/wls.jar'

print('=== [INICIO] Creacion de dominio offline: ' + DOMAIN_NAME + ' ===')

# --- Paso 1: Cargar plantilla ---
print('[1/5] Cargando plantilla: ' + TEMPLATE_PATH)
readTemplate(TEMPLATE_PATH)

# --- Paso 2: Configurar AdminServer ---
print('[2/5] Configurando AdminServer en ' + ADMIN_ADDRESS + ':' + str(ADMIN_PORT))
cd('/Servers/AdminServer')
set('ListenAddress', ADMIN_ADDRESS)
set('ListenPort',    ADMIN_PORT)
set('Name',          'AdminServer')

# --- Paso 3: Establecer credenciales del administrador ---
print('[3/5] Configurando usuario administrador: ' + ADMIN_USER)
cd('/Security/' + DOMAIN_NAME + '/User/weblogic')
cmo.setPassword(ADMIN_PASSWORD)

# --- Paso 4: Configurar opciones del dominio ---
print('[4/5] Estableciendo modo de inicio: prod')
setOption('ServerStartMode', 'prod')
setOption('OverwriteDomain',  'true')

# --- Paso 5: Escribir dominio ---
print('[5/5] Escribiendo dominio en: ' + DOMAIN_PATH)
writeDomain(DOMAIN_PATH)
closeTemplate()

print('=== [FIN] Dominio creado exitosamente en: ' + DOMAIN_PATH + ' ===')
PYEOF
```

2. Ejecuta el script:

```bash
wlst.sh /home/oracle/labs/lab04/scripts/01_create_domain_offline.py
```

3. Verifica que el dominio fue creado correctamente:

```bash
ls -la /u01/domains/lab04_domain/
cat /u01/domains/lab04_domain/config/config.xml | grep -E 'listen-port|name'
```

**Salida Esperada:**

```
=== [INICIO] Creacion de dominio offline: lab04_domain ===
[1/5] Cargando plantilla: /u01/oracle/middleware/wlserver/common/templates/wls/wls.jar
[2/5] Configurando AdminServer en localhost:7101
[3/5] Configurando usuario administrador: weblogic
[4/5] Estableciendo modo de inicio: prod
[5/5] Escribiendo dominio en: /u01/domains/lab04_domain
=== [FIN] Dominio creado exitosamente en: /u01/domains/lab04_domain ===
```

```bash
# Verificación config.xml
<listen-port>7101</listen-port>
<name>AdminServer</name>
```

**Verificación:** El directorio `/u01/domains/lab04_domain/config/config.xml` existe y contiene `<listen-port>7101</listen-port>`.

---

### Paso 3: Añadir un JDBC Data Source en Modo Offline

**Objetivo:** Usar `readDomain`/`updateDomain` para añadir un JDBC Data Source al dominio `lab04_domain` sin arrancar ningún servidor, comprendiendo la diferencia con el enfoque declarativo de WDT.

**Instrucciones:**

1. Crea el script `/home/oracle/labs/lab04/scripts/02_add_datasource_offline.py`:

```bash
cat > /home/oracle/labs/lab04/scripts/02_add_datasource_offline.py << 'PYEOF'
# =============================================================
# Script: 02_add_datasource_offline.py
# Descripcion: Agrega un JDBC DataSource a lab04_domain (offline)
# Uso: wlst.sh 02_add_datasource_offline.py
# =============================================================

DOMAIN_PATH = '/u01/domains/lab04_domain'
DS_NAME     = 'LabDataSource'
DS_JNDI     = 'jdbc/LabDS'

print('=== [INICIO] Agregando DataSource offline: ' + DS_NAME + ' ===')

# --- Abrir dominio existente (detenido) ---
readDomain(DOMAIN_PATH)
cd('/')

# --- Verificar si ya existe (idempotencia) ---
existing = None
try:
    existing = getMBean('/JDBCSystemResources/' + DS_NAME)
except:
    pass

if existing is not None:
    print('[INFO] DataSource ' + DS_NAME + ' ya existe. Se omite creacion.')
else:
    print('[1/4] Creando JDBCSystemResource: ' + DS_NAME)
    create(DS_NAME, 'JDBCSystemResource')

    print('[2/4] Configurando parametros del driver JDBC')
    cd('/JDBCSystemResources/' + DS_NAME + '/JdbcResource/' + DS_NAME)
    set('Name', DS_NAME)

    cd('JDBCDriverParams/' + DS_NAME)
    set('DriverName', 'oracle.jdbc.OracleDriver')
    set('URL', 'jdbc:oracle:thin:@//localhost:1521/XEPDB1')
    set('PasswordEncrypted', 'DBLabPassword1')

    print('[3/4] Configurando parametros JNDI')
    cd('/JDBCSystemResources/' + DS_NAME + '/JdbcResource/' + DS_NAME + '/JDBCDataSourceParams/' + DS_NAME)
    set('JNDINames', jarray.array([DS_JNDI], java.lang.String))

    print('[4/4] Configurando pool de conexiones')
    cd('/JDBCSystemResources/' + DS_NAME + '/JdbcResource/' + DS_NAME + '/JDBCConnectionPoolParams/' + DS_NAME)
    set('InitialCapacity',          1)
    set('MaxCapacity',              10)
    set('TestConnectionsOnReserve', True)
    set('TestTableName',            'SQL SELECT 1 FROM DUAL')

# --- Persistir cambios ---
print('[OK] Actualizando dominio...')
updateDomain()
closeDomain()

print('=== [FIN] DataSource configurado en: ' + DOMAIN_PATH + ' ===')
PYEOF
```

2. Ejecuta el script:

```bash
wlst.sh /home/oracle/labs/lab04/scripts/02_add_datasource_offline.py
```

3. Verifica la presencia del DataSource en `config.xml`:

```bash
grep -A 5 'LabDataSource' /u01/domains/lab04_domain/config/config.xml
```

**Salida Esperada:**

```
=== [INICIO] Agregando DataSource offline: LabDataSource ===
[1/4] Creando JDBCSystemResource: LabDataSource
[2/4] Configurando parametros del driver JDBC
[3/4] Configurando parametros JNDI
[4/4] Configurando pool de conexiones
[OK] Actualizando dominio...
=== [FIN] DataSource configurado en: /u01/domains/lab04_domain ===
```

**Verificación:** `grep 'LabDataSource' /u01/domains/lab04_domain/config/config.xml` devuelve al menos una línea con el nombre del recurso.

> **Reflexión (Offline vs WDT):** El enfoque offline de WLST es imperativo: describes *cómo* crear el recurso paso a paso. WDT es declarativo: describes el *estado deseado* en YAML y el tooling calcula los pasos. Para aprovisionamiento reproducible a gran escala, WDT resulta más mantenible; WLST offline es preferible para modificaciones quirúrgicas o lógica condicional compleja.

---

### Parte 2 — WLST en Modo Online

---

### Paso 4: Conexión Online y Navegación del Árbol de MBeans

**Objetivo:** Conectarse al Administration Server de `wdt_domain` (puerto 7001) y explorar los tres árboles de MBeans: `config`, `runtime` y `domainRuntime`.

**Instrucciones:**

1. Asegúrate de que el Administration Server de `wdt_domain` está activo:

```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:7001/weblogic/ready
# Debe devolver: 200
```

2. Lanza WLST en modo interactivo y conéctate:

```bash
wlst.sh
```

```python
# Conectar al Administration Server
connect('weblogic', 'Welcome1', 't3://localhost:7001')
# Prompt esperado: wls:/wdt_domain/serverConfig>
```

3. Explora el árbol de **configuración** (config):

```python
# Estamos en serverConfig por defecto
ls()

# Listar servidores configurados
cd('Servers')
ls()

# Ver propiedades del AdminServer
cd('AdminServer')
ls()
get('ListenPort')
get('ListenAddress')

# Volver a la raiz
cd('/')
```

4. Cambia al árbol de **runtime** del servidor actual:

```python
serverRuntime()
# Prompt: wls:/wdt_domain/serverRuntime>
ls()

# Explorar JVM
cd('JVMRuntime')
ls()
get('HeapFreeCurrent')
get('HeapSizeMax')

# Explorar ThreadPool
cd('../ThreadPoolRuntime')
get('ExecuteThreadTotalCount')
get('ExecuteThreadIdleCount')
get('PendingUserRequestCount')
```

5. Cambia al árbol de **domainRuntime** (vista global del dominio):

```python
domainRuntime()
# Prompt: wls:/wdt_domain/domainRuntime>
ls()

# Estado de ciclo de vida de todos los servidores
cd('ServerLifeCycleRuntimes')
ls()

cd('AdminServer')
get('State')
get('Health')
```

6. Desconéctate:

```python
disconnect()
exit()
```

**Salida Esperada (extracto):**

```
wls:/wdt_domain/serverConfig> ls()
dr--   AppDeployments
dr--   Clusters
dr--   JDBCSystemResources
dr--   Servers
...

wls:/wdt_domain/serverRuntime/JVMRuntime> get('HeapSizeMax')
536870912

wls:/wdt_domain/domainRuntime/ServerLifeCycleRuntimes/AdminServer> get('State')
'RUNNING'
```

**Verificación:** El prompt cambia correctamente entre `serverConfig`, `serverRuntime` y `domainRuntime`. El estado del AdminServer devuelve `RUNNING`.

---

### Paso 5: Desplegar una Aplicación con WLST Online

**Objetivo:** Escribir un script de despliegue completo que valide la existencia de la aplicación antes de desplegarla, implemente lógica de rollback y genere un log de auditoría.

**Instrucciones:**

1. Verifica que el WAR de muestra existe:

```bash
ls -lh /home/oracle/labs/lab04/sample-app.war
# Si no existe, crea un WAR mínimo de prueba:
mkdir -p /tmp/webapp/WEB-INF
echo '<web-app xmlns="https://jakarta.ee/xml/ns/jakartaee" version="5.0"/>' \
  > /tmp/webapp/WEB-INF/web.xml
echo '<html><body><h1>Lab04 Sample App</h1></body></html>' \
  > /tmp/webapp/index.html
cd /tmp/webapp && jar cvf /home/oracle/labs/lab04/sample-app.war .
cd /home/oracle/labs/lab04
```

2. Crea el script de despliegue `/home/oracle/labs/lab04/scripts/03_deploy_app.py`:

```bash
cat > /home/oracle/labs/lab04/scripts/03_deploy_app.py << 'PYEOF'
# =============================================================
# Script: 03_deploy_app.py
# Descripcion: Despliega sample-app.war con validacion y rollback
# Uso: wlst.sh -loadProperties env.properties 03_deploy_app.py
# Propiedades requeridas: admin.url, admin.user, admin.pass
# =============================================================
import sys
import os
import datetime

# --- Parametros ---
APP_NAME    = 'sample-app'
APP_PATH    = '/home/oracle/labs/lab04/sample-app.war'
TARGET      = 'AdminServer'
LOG_FILE    = '/home/oracle/labs/lab04/logs/deploy_audit.log'
ADMIN_URL   = admin_url    # Inyectado por -loadProperties
ADMIN_USER  = admin_user
ADMIN_PASS  = admin_pass

# --- Funcion de logging ---
def log(level, message):
    ts = str(datetime.datetime.now())
    line = '[' + ts + '] [' + level + '] ' + message
    print(line)
    try:
        f = open(LOG_FILE, 'a')
        f.write(line + '\n')
        f.close()
    except Exception as e:
        print('[WARN] No se pudo escribir en log: ' + str(e))

# --- Funcion: verificar si la app ya esta desplegada ---
def app_exists(app_name):
    try:
        app_mbean = getMBean('/AppDeployments/' + app_name)
        return app_mbean is not None
    except:
        return False

# --- Funcion principal ---
def deploy_application():
    log('INFO', '=== INICIO despliegue: ' + APP_NAME + ' ===')

    # Verificar que el WAR existe en disco
    if not os.path.exists(APP_PATH):
        log('ERROR', 'Archivo WAR no encontrado: ' + APP_PATH)
        sys.exit(1)

    # Conectar al Administration Server
    log('INFO', 'Conectando a: ' + ADMIN_URL)
    try:
        connect(ADMIN_USER, ADMIN_PASS, ADMIN_URL)
        log('INFO', 'Conexion establecida correctamente')
    except Exception as e:
        log('ERROR', 'Fallo al conectar: ' + str(e))
        sys.exit(1)

    # Verificar si la app ya existe (decision: deploy vs redeploy)
    if app_exists(APP_NAME):
        log('INFO', 'Aplicacion existente detectada. Ejecutando REDEPLOY.')
        action = 'redeploy'
    else:
        log('INFO', 'Aplicacion nueva. Ejecutando DEPLOY inicial.')
        action = 'deploy'

    # Ejecutar despliegue con manejo de errores y rollback
    try:
        if action == 'deploy':
            log('INFO', 'Desplegando ' + APP_NAME + ' en target: ' + TARGET)
            deploy(APP_NAME,
                   APP_PATH,
                   targets=TARGET,
                   block='true',
                   upload='true')
            log('INFO', 'Despliegue completado exitosamente')

        else:  # redeploy
            log('INFO', 'Redesplegando ' + APP_NAME + ' en target: ' + TARGET)
            redeploy(APP_NAME,
                     APP_PATH,
                     block='true')
            log('INFO', 'Redeployment completado exitosamente')

    except Exception as e:
        log('ERROR', 'Fallo durante ' + action + ': ' + str(e))
        log('WARN',  'Iniciando rollback: undeploy de ' + APP_NAME)
        try:
            undeploy(APP_NAME, targets=TARGET, block='true')
            log('INFO', 'Rollback completado: aplicacion eliminada')
        except Exception as re:
            log('ERROR', 'Rollback fallido: ' + str(re))
        disconnect()
        sys.exit(2)

    # Verificar estado post-despliegue
    log('INFO', 'Verificando estado de la aplicacion...')
    try:
        domainRuntime()
        cd('AppRuntimeStateRuntime/AppRuntimeStateRuntime')
        state = getAppState(APP_NAME, TARGET)
        log('INFO', 'Estado de ' + APP_NAME + ' en ' + TARGET + ': ' + str(state))
    except Exception as e:
        log('WARN', 'No se pudo verificar estado runtime: ' + str(e))

    disconnect()
    log('INFO', '=== FIN despliegue: ' + APP_NAME + ' ===')

# --- Entry point ---
deploy_application()
PYEOF
```

3. Ejecuta el script usando `-loadProperties`:

```bash
wlst.sh -loadProperties /home/oracle/labs/lab04/env.properties \
        /home/oracle/labs/lab04/scripts/03_deploy_app.py
```

4. Revisa el log de auditoría:

```bash
cat /home/oracle/labs/lab04/logs/deploy_audit.log
```

5. Ejecuta el script **una segunda vez** para verificar que el flujo de redeploy funciona correctamente:

```bash
wlst.sh -loadProperties /home/oracle/labs/lab04/env.properties \
        /home/oracle/labs/lab04/scripts/03_deploy_app.py
```

**Salida Esperada (primera ejecución):**

```
[2025-...] [INFO] === INICIO despliegue: sample-app ===
[2025-...] [INFO] Conectando a: t3://localhost:7001
[2025-...] [INFO] Conexion establecida correctamente
[2025-...] [INFO] Aplicacion nueva. Ejecutando DEPLOY inicial.
[2025-...] [INFO] Desplegando sample-app en target: AdminServer
[2025-...] [INFO] Despliegue completado exitosamente
[2025-...] [INFO] === FIN despliegue: sample-app ===
```

**Verificación:**

```bash
# Verificar que la aplicacion responde
curl -s -o /dev/null -w "%{http_code}" http://localhost:7001/sample-app/
# Resultado esperado: 200
```

---

### Paso 6: Crear un JDBC Data Source en Modo Online con Sesión de Edición

**Objetivo:** Usar el flujo transaccional `edit()` → `startEdit()` → cambios → `save()` → `activate()` para crear un JDBC Data Source en `wdt_domain` con targeting explícito al AdminServer.

**Instrucciones:**

1. Crea el script `/home/oracle/labs/lab04/scripts/04_create_jdbc_online.py`:

```bash
cat > /home/oracle/labs/lab04/scripts/04_create_jdbc_online.py << 'PYEOF'
# =============================================================
# Script: 04_create_jdbc_online.py
# Descripcion: Crea JDBC DataSource en wdt_domain (modo online)
#              con sesion de edicion transaccional
# Uso: wlst.sh -loadProperties env.properties 04_create_jdbc_online.py
# =============================================================
import sys
import datetime

DS_NAME    = 'WdtLabDataSource'
DS_JNDI    = 'jdbc/WdtLabDS'
DS_TARGET  = 'AdminServer'
ADMIN_URL  = admin_url
ADMIN_USER = admin_user
ADMIN_PASS = admin_pass

def log(level, msg):
    print('[' + str(datetime.datetime.now()) + '] [' + level + '] ' + msg)

def ds_exists(name):
    try:
        return getMBean('/JDBCSystemResources/' + name) is not None
    except:
        return False

log('INFO', '=== Inicio: crear DataSource online ===')

# Conectar
connect(ADMIN_USER, ADMIN_PASS, ADMIN_URL)
log('INFO', 'Conectado a: ' + ADMIN_URL)

# Iniciar sesion de edicion
edit()
startEdit()
log('INFO', 'Sesion de edicion iniciada')

try:
    cd('/')

    if ds_exists(DS_NAME):
        log('INFO', DS_NAME + ' ya existe. Actualizando MaxCapacity.')
        cd('/JDBCSystemResources/' + DS_NAME +
           '/JdbcResource/' + DS_NAME +
           '/JDBCConnectionPoolParams/' + DS_NAME)
        set('MaxCapacity', 20)
        log('INFO', 'MaxCapacity actualizado a 20')
    else:
        log('INFO', 'Creando JDBCSystemResource: ' + DS_NAME)
        create(DS_NAME, 'JDBCSystemResource')

        cd('/JDBCSystemResources/' + DS_NAME +
           '/JdbcResource/' + DS_NAME)
        set('Name', DS_NAME)

        cd('JDBCDriverParams/' + DS_NAME)
        set('DriverName', 'oracle.jdbc.OracleDriver')
        set('URL', 'jdbc:oracle:thin:@//localhost:1521/XEPDB1')
        set('PasswordEncrypted', 'DBLabPassword1')

        cd('/JDBCSystemResources/' + DS_NAME +
           '/JdbcResource/' + DS_NAME +
           '/JDBCDataSourceParams/' + DS_NAME)
        set('JNDINames', jarray.array([DS_JNDI], java.lang.String))

        cd('/JDBCSystemResources/' + DS_NAME +
           '/JdbcResource/' + DS_NAME +
           '/JDBCConnectionPoolParams/' + DS_NAME)
        set('InitialCapacity',          1)
        set('MaxCapacity',              15)
        set('TestConnectionsOnReserve', True)
        set('TestTableName',            'SQL SELECT 1 FROM DUAL')

        log('INFO', 'Asignando DataSource al target: ' + DS_TARGET)
        assign('JDBCSystemResource', DS_NAME, 'Target', DS_TARGET)

    # Guardar y activar
    log('INFO', 'Guardando cambios...')
    save()
    log('INFO', 'Activando cambios (block=true)...')
    activate(block='true')
    log('INFO', 'Cambios activados exitosamente')

except Exception as e:
    log('ERROR', 'Fallo durante configuracion: ' + str(e))
    log('WARN',  'Ejecutando undoEdit para revertir cambios...')
    undoEdit(defaultAnswer='y')
    disconnect()
    sys.exit(1)

disconnect()
log('INFO', '=== Fin: DataSource ' + DS_NAME + ' configurado ===')
PYEOF
```

2. Ejecuta el script:

```bash
wlst.sh -loadProperties /home/oracle/labs/lab04/env.properties \
        /home/oracle/labs/lab04/scripts/04_create_jdbc_online.py
```

3. Verifica el DataSource en la consola o mediante WLST interactivo:

```bash
wlst.sh << 'EOF'
connect('weblogic','Welcome1','t3://localhost:7001')
cd('/JDBCSystemResources/WdtLabDataSource')
ls()
disconnect()
exit()
EOF
```

**Salida Esperada:**

```
[...] [INFO] === Inicio: crear DataSource online ===
[...] [INFO] Conectado a: t3://localhost:7001
[...] [INFO] Sesion de edicion iniciada
[...] [INFO] Creando JDBCSystemResource: WdtLabDataSource
[...] [INFO] Asignando DataSource al target: AdminServer
[...] [INFO] Guardando cambios...
[...] [INFO] Activando cambios (block=true)...
[...] [INFO] Cambios activados exitosamente
[...] [INFO] === Fin: DataSource WdtLabDataSource configurado ===
```

**Verificación:** El MBean `/JDBCSystemResources/WdtLabDataSource` existe y `ls()` muestra sus propiedades correctamente.

---

### Paso 7: Capturar Métricas de Runtime y Gestionar el Ciclo de Vida

**Objetivo:** Escribir un script que capture métricas de JVM, ThreadPool y estado de servidores, e implemente funciones de arranque/parada usando `domainRuntime`.

**Instrucciones:**

1. Crea el script `/home/oracle/labs/lab04/scripts/05_runtime_metrics.py`:

```bash
cat > /home/oracle/labs/lab04/scripts/05_runtime_metrics.py << 'PYEOF'
# =============================================================
# Script: 05_runtime_metrics.py
# Descripcion: Captura metricas de JVM, ThreadPool y estado
#              de servidores del dominio wdt_domain
# Uso: wlst.sh -loadProperties env.properties 05_runtime_metrics.py
# =============================================================
import sys
import datetime

ADMIN_URL  = admin_url
ADMIN_USER = admin_user
ADMIN_PASS = admin_pass

def log(level, msg):
    print('[' + str(datetime.datetime.now()) + '] [' + level + '] ' + msg)

def bytes_to_mb(b):
    return str(round(b / 1024.0 / 1024.0, 2)) + ' MB'

log('INFO', '=== Inicio: Captura de metricas runtime ===')
connect(ADMIN_USER, ADMIN_PASS, ADMIN_URL)

# -------------------------------------------------------
# 1. Metricas de JVM (serverRuntime del AdminServer)
# -------------------------------------------------------
log('INFO', '--- JVM Runtime ---')
serverRuntime()
cd('JVMRuntime')
heap_free    = get('HeapFreeCurrent')
heap_max     = get('HeapSizeMax')
heap_used    = heap_max - heap_free
heap_pct     = round((heap_used * 100.0) / heap_max, 1)
log('INFO', 'Heap Max:  ' + bytes_to_mb(heap_max))
log('INFO', 'Heap Used: ' + bytes_to_mb(heap_used) + ' (' + str(heap_pct) + '%)')
log('INFO', 'Heap Free: ' + bytes_to_mb(heap_free))

# -------------------------------------------------------
# 2. Metricas de ThreadPool
# -------------------------------------------------------
log('INFO', '--- ThreadPool Runtime ---')
cd('../ThreadPoolRuntime')
total   = get('ExecuteThreadTotalCount')
idle    = get('ExecuteThreadIdleCount')
pending = get('PendingUserRequestCount')
log('INFO', 'Hilos Total:    ' + str(total))
log('INFO', 'Hilos Ociosos:  ' + str(idle))
log('INFO', 'Peticiones Pendientes: ' + str(pending))

if pending > 10:
    log('WARN', 'Cola de peticiones alta: ' + str(pending) + ' pendientes')

# -------------------------------------------------------
# 3. Estado de servidores (domainRuntime)
# -------------------------------------------------------
log('INFO', '--- Estado de Servidores (domainRuntime) ---')
domainRuntime()
cd('ServerLifeCycleRuntimes')
servers = ls(returnMap='true')
for server_name in servers:
    cd('ServerLifeCycleRuntimes/' + server_name)
    state  = get('State')
    health = get('Health')
    log('INFO', 'Servidor: ' + server_name +
        ' | Estado: ' + str(state) +
        ' | Health: ' + str(health))
    cd('..')

# -------------------------------------------------------
# 4. Metricas de DataSources (si existen)
# -------------------------------------------------------
log('INFO', '--- JDBC DataSource Runtime ---')
try:
    domainRuntime()
    cd('ServerRuntimes/AdminServer/JDBCServiceRuntime/JDBCDataSourceRuntimeMBeans')
    ds_list = ls(returnMap='true')
    if ds_list:
        for ds in ds_list:
            cd('JDBCDataSourceRuntimeMBeans/' + ds)
            state    = get('State')
            active   = get('ActiveConnectionsCurrentCount')
            waiting  = get('WaitingForConnectionCurrentCount')
            log('INFO', 'DS: ' + ds +
                ' | State: ' + str(state) +
                ' | Active: ' + str(active) +
                ' | Waiting: ' + str(waiting))
            cd('..')
    else:
        log('INFO', 'No hay DataSources con runtime activo en AdminServer')
except Exception as e:
    log('WARN', 'No se pudo acceder a JDBC runtime: ' + str(e))

disconnect()
log('INFO', '=== Fin: Captura de metricas completada ===')
PYEOF
```

2. Ejecuta el script:

```bash
wlst.sh -loadProperties /home/oracle/labs/lab04/env.properties \
        /home/oracle/labs/lab04/scripts/05_runtime_metrics.py
```

**Salida Esperada (valores aproximados):**

```
[...] [INFO] === Inicio: Captura de metricas runtime ===
[...] [INFO] --- JVM Runtime ---
[...] [INFO] Heap Max:  512.0 MB
[...] [INFO] Heap Used: 187.3 MB (36.6%)
[...] [INFO] Heap Free: 324.7 MB
[...] [INFO] --- ThreadPool Runtime ---
[...] [INFO] Hilos Total:    25
[...] [INFO] Hilos Ociosos:  23
[...] [INFO] Peticiones Pendientes: 0
[...] [INFO] --- Estado de Servidores (domainRuntime) ---
[...] [INFO] Servidor: AdminServer | Estado: RUNNING | Health: HEALTH_OK
[...] [INFO] === Fin: Captura de metricas completada ===
```

**Verificación:** El script finaliza sin errores y muestra el estado `RUNNING` y `HEALTH_OK` para el AdminServer.

---

### Paso 8: Generar Credenciales Cifradas con storeUserConfig

**Objetivo:** Eliminar la dependencia de `env.properties` con contraseña en texto claro, usando `storeUserConfig` para generar archivos cifrados de credenciales.

**Instrucciones:**

1. Crea el directorio para los archivos de credenciales:

```bash
mkdir -p /home/oracle/.wlst
chmod 700 /home/oracle/.wlst
```

2. Genera los archivos cifrados mediante WLST interactivo:

```bash
wlst.sh << 'EOF'
connect('weblogic', 'Welcome1', 't3://localhost:7001')
storeUserConfig(userConfigFile='/home/oracle/.wlst/userconfig.secure',
                userKeyFile='/home/oracle/.wlst/userkey.secure')
disconnect()
exit()
EOF
```

3. Verifica que los archivos fueron creados y tienen permisos restrictivos:

```bash
ls -la /home/oracle/.wlst/
chmod 600 /home/oracle/.wlst/userconfig.secure
chmod 600 /home/oracle/.wlst/userkey.secure
```

4. Prueba la conexión usando los archivos cifrados (sin contraseña en texto claro):

```bash
wlst.sh << 'EOF'
connect(userConfigFile='/home/oracle/.wlst/userconfig.secure',
        userKeyFile='/home/oracle/.wlst/userkey.secure',
        url='t3://localhost:7001')
serverRuntime()
cd('JVMRuntime')
print('Heap Max (bytes): ' + str(get('HeapSizeMax')))
disconnect()
exit()
EOF
```

**Salida Esperada:**

```
Connecting to t3://localhost:7001 with userid from userConfigFile ...
Successfully connected to Admin Server "AdminServer" ...
Heap Max (bytes): 536870912
```

**Verificación:** La conexión se establece sin especificar usuario ni contraseña en el comando. Los archivos `.secure` tienen permisos `600`.

---

## Validación y Pruebas

Ejecuta el siguiente bloque de validación completo al finalizar todos los pasos:

```bash
#!/bin/bash
echo "=== VALIDACION FINAL Lab 04-00-01 ==="

# 1. Verificar dominio lab04_domain creado offline
echo "[1] Verificando lab04_domain..."
if [ -f /u01/domains/lab04_domain/config/config.xml ]; then
    PORT=$(grep -oP '(?<=<listen-port>)\d+' /u01/domains/lab04_domain/config/config.xml | head -1)
    echo "    OK: lab04_domain existe. Puerto AdminServer: $PORT"
    [ "$PORT" = "7101" ] && echo "    OK: Puerto 7101 correcto" || echo "    WARN: Puerto inesperado: $PORT"
else
    echo "    ERROR: lab04_domain no encontrado"
fi

# 2. Verificar DataSource offline en lab04_domain
echo "[2] Verificando LabDataSource en lab04_domain..."
grep -q 'LabDataSource' /u01/domains/lab04_domain/config/config.xml \
    && echo "    OK: LabDataSource presente en config.xml" \
    || echo "    ERROR: LabDataSource no encontrado"

# 3. Verificar Administration Server de wdt_domain activo
echo "[3] Verificando Administration Server wdt_domain..."
STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:7001/weblogic/ready)
[ "$STATUS" = "200" ] && echo "    OK: AdminServer respondiendo (HTTP 200)" \
    || echo "    ERROR: AdminServer no responde (HTTP $STATUS)"

# 4. Verificar aplicacion desplegada
echo "[4] Verificando sample-app desplegada..."
APP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:7001/sample-app/)
[ "$APP_STATUS" = "200" ] && echo "    OK: sample-app responde (HTTP 200)" \
    || echo "    WARN: sample-app devuelve HTTP $APP_STATUS"

# 5. Verificar log de auditoria
echo "[5] Verificando log de auditoria..."
[ -f /home/oracle/labs/lab04/logs/deploy_audit.log ] \
    && echo "    OK: deploy_audit.log existe ($(wc -l < /home/oracle/labs/lab04/logs/deploy_audit.log) lineas)" \
    || echo "    WARN: deploy_audit.log no encontrado"

# 6. Verificar credenciales cifradas
echo "[6] Verificando archivos de credenciales cifradas..."
[ -f /home/oracle/.wlst/userconfig.secure ] \
    && echo "    OK: userconfig.secure existe" \
    || echo "    WARN: userconfig.secure no encontrado"

echo "=== FIN VALIDACION ==="
```

```bash
bash /home/oracle/labs/lab04/validate_lab04.sh
```

---

## Comparativa: WLST vs WDT vs REST API

| Criterio                        | WLST                            | WDT                              | REST API                          |
|---------------------------------|---------------------------------|----------------------------------|-----------------------------------|
| **Paradigma**                   | Imperativo (Jython)             | Declarativo (YAML/JSON)          | Imperativo (HTTP)                 |
| **Requiere servidor activo**    | Online: sí / Offline: no        | No (offline) / Sí (online)       | Sí (siempre)                      |
| **Curva de aprendizaje**        | Media (Jython + MBeans)         | Baja (YAML estructurado)         | Baja (HTTP estándar)              |
| **Idoneidad para CI/CD**        | Media                           | Alta                             | Alta                              |
| **Lógica condicional**          | Nativa (Python completo)        | Limitada (variables/tokens)      | Requiere scripting externo        |
| **Portabilidad entre versiones**| Media (APIs cambian)            | Alta (abstracción de versión)    | Alta (REST estable)               |
| **Caso de uso óptimo**          | Ajustes quirúrgicos, métricas, operaciones complejas con lógica | Aprovisionamiento reproducible, IaC, migración de dominios | Integración con sistemas externos, dashboards, automatización HTTP |
| **Acceso a métricas runtime**   | Sí (serverRuntime/domainRuntime)| No                               | Sí (Management REST API)          |
| **Rollback transaccional**      | Sí (`undoEdit`)                 | Parcial (backup automático)      | No nativo                         |

> **Regla práctica:** Usa **WDT** para crear y replicar dominios (infraestructura como código). Usa **WLST** cuando necesites lógica condicional, acceso a métricas runtime o cambios transaccionales con rollback. Usa la **REST API** para integraciones con plataformas externas (Ansible, Terraform, dashboards).

---

## Resolución de Problemas

### Problema 1: `NameError: admin_url is not defined` al ejecutar con `-loadProperties`

**Síntomas:** El script falla inmediatamente con un error como:
```
NameError: admin_url
```
o
```
AttributeError: 'NoneType' object has no attribute ...
```

**Causa:** El archivo `env.properties` usa puntos en los nombres de clave (`admin.url`, `admin.user`, `admin.pass`). WLST convierte los puntos a caracteres que Jython no puede usar como identificadores de variable directamente. Las variables se cargan como `admin_url` (con guión bajo) solo si el nombre de propiedad no contiene caracteres especiales, pero el punto puede causar conflicto según la versión.

**Solución:**

```bash
# Opcion 1: Cambiar las claves en env.properties a guiones bajos
cat > /home/oracle/labs/lab04/env.properties << 'EOF'
admin_url=t3://localhost:7001
admin_user=weblogic
admin_pass=Welcome1
EOF

# Opcion 2: Leer las propiedades manualmente en el script
# Al inicio del script, agregar:
# import java.util.Properties as Properties
# props = Properties()
# props.load(open('/home/oracle/labs/lab04/env.properties'))
# ADMIN_URL  = props.getProperty('admin.url')
# ADMIN_USER = props.getProperty('admin.user')
# ADMIN_PASS = props.getProperty('admin.pass')
```

**Verificación:** El script se ejecuta sin `NameError` y muestra `Conectado a: t3://localhost:7001`.

---

### Problema 2: `EditTimedOutException` o `ActiveEditorSessions` al ejecutar `startEdit()`

**Síntomas:** Al intentar iniciar una sesión de edición, WLST devuelve:
```
WLSTException: Edit lock is already held by another session
```
o
```
com.bea.ns.weblogic.management.EditTimedOutException
```

**Causa:** Una sesión de edición anterior quedó abierta (por ejemplo, porque el script falló sin ejecutar `undoEdit()` o `activate()`). WebLogic solo permite una sesión de edición activa a la vez.

**Solución:**

```python
# Conectarse al servidor y forzar liberacion del lock
wlst.sh << 'EOF'
connect('weblogic', 'Welcome1', 't3://localhost:7001')

# Ver quien tiene el lock activo
edit()
# Si el lock es tuyo (misma sesion), ejecuta:
undoEdit(defaultAnswer='y')

# Si el lock es de otra sesion, usa stopEdit() como administrador:
# stopEdit(defaultAnswer='y')

disconnect()
exit()
EOF
```

```bash
# Alternativa: reiniciar el Administration Server limpia todos los locks
# (solo si no hay cambios criticos pendientes)
```

**Verificación:** Tras ejecutar `undoEdit()` o `stopEdit()`, el siguiente `startEdit()` se completa sin error. El prompt regresa a `wls:/wdt_domain/edit>`.

---

## Limpieza del Laboratorio

Ejecuta los siguientes pasos para dejar el entorno en estado limpio al finalizar:

```bash
# 1. Desplegar la aplicacion de muestra (si se desea mantener limpio wdt_domain)
wlst.sh << 'EOF'
connect('weblogic', 'Welcome1', 't3://localhost:7001')
undeploy('sample-app', targets='AdminServer', block='true')
disconnect()
exit()
EOF

# 2. Eliminar el DataSource creado online (opcional, si no se usa en labs posteriores)
wlst.sh -loadProperties /home/oracle/labs/lab04/env.properties << 'WEOF'
connect(admin_user, admin_pass, admin_url)
edit()
startEdit()
cd('/')
delete('WdtLabDataSource', 'JDBCSystemResource')
save()
activate(block='true')
disconnect()
exit()
WEOF

# 3. Eliminar el dominio lab04_domain creado offline
rm -rf /u01/domains/lab04_domain
echo "Dominio lab04_domain eliminado"

# 4. Eliminar logs del laboratorio (opcional)
rm -f /home/oracle/labs/lab04/logs/deploy_audit.log

# 5. Conservar los scripts para referencia futura
echo "Scripts conservados en: /home/oracle/labs/lab04/scripts/"
ls /home/oracle/labs/lab04/scripts/

# 6. Verificar que wdt_domain sigue operativo
curl -s -o /dev/null -w "wdt_domain AdminServer: HTTP %{http_code}\n" \
  http://localhost:7001/weblogic/ready
```

> **Nota:** Los archivos de credenciales cifradas en `/home/oracle/.wlst/` pueden conservarse para los laboratorios siguientes. Asegúrate de que los permisos son `600`.

---

## Resumen

En este laboratorio has completado el ciclo completo de automatización con WLST:

| Actividad                                  | Modo    | Script                            |
|--------------------------------------------|---------|-----------------------------------|
| Exploración interactiva del árbol offline  | Offline | Interactivo                       |
| Creación de dominio desde plantilla        | Offline | `01_create_domain_offline.py`     |
| Adición de DataSource sin servidor activo  | Offline | `02_add_datasource_offline.py`    |
| Navegación del árbol de MBeans online      | Online  | Interactivo                       |
| Despliegue con validación y rollback       | Online  | `03_deploy_app.py`                |
| Creación de DataSource con sesión de edición | Online | `04_create_jdbc_online.py`       |
| Captura de métricas JVM y ThreadPool       | Online  | `05_runtime_metrics.py`           |
| Generación de credenciales cifradas        | Online  | `storeUserConfig` (interactivo)   |

**Conceptos clave consolidados:**

- El flujo transaccional online (`edit` → `startEdit` → cambios → `save` → `activate`) garantiza consistencia y permite rollback con `undoEdit`.
- Los tres árboles de MBeans tienen propósitos distintos: `serverConfig` (configuración persistente), `serverRuntime` (métricas del servidor actual), `domainRuntime` (vista global del dominio).
- La idempotencia en scripts WLST se logra verificando la existencia de recursos antes de crearlos (`getMBean`).
- `storeUserConfig`/`userKeyFile` elimina la necesidad de contraseñas en texto claro en scripts de producción.
- WLST es la herramienta óptima para lógica condicional compleja, acceso a métricas runtime y cambios transaccionales; WDT es preferible para aprovisionamiento declarativo y reproducible.

### Recursos Adicionales

- [Oracle WLST Command Reference 14.1.1.0](https://docs.oracle.com/en/middleware/standalone/weblogic-server/14.1.1.0/wlstc/index.html)
- [Understanding the WebLogic Scripting Tool](https://docs.oracle.com/en/middleware/standalone/weblogic-server/14.1.1.0/wlstg/overview.html)
- [WebLogic MBean Reference](https://docs.oracle.com/en/middleware/standalone/weblogic-server/14.1.1.0/mbref/index.html)
- [WebLogic Deploy Tooling (WDT) — GitHub](https://github.com/oracle/weblogic-deploy-tooling)

---

---

# Usar la API REST de WebLogic y scripts externos

## 1. Metadatos

| Campo            | Valor                                      |
|------------------|--------------------------------------------|
| **Duración**     | 75 minutos                                 |
| **Complejidad**  | Media                                      |
| **Nivel Bloom**  | Crear (Create)                             |
| **Módulo**       | 4 — Herramientas modernas de gestión       |
| **Laboratorio**  | 04-00-02 (complementa 04-00-01)            |

---

## 2. Descripción General

Este laboratorio aborda la **WebLogic REST Management API** como interfaz de integración con herramientas externas, complementando el uso de WLST visto en la lección 4.1. El alumno explorará los endpoints principales de la API (v1 y v2), ejecutará operaciones CRUD con `curl` documentando peticiones y respuestas JSON, y desarrollará un **script Python 3** que encapsula un mini-framework de administración reutilizable. Se aplicarán buenas prácticas de autenticación HTTP Basic, manejo seguro de credenciales y construcción de un monitor de salud de servidores basado en los endpoints de Health Check de WebLogic 15.

---

## 3. Objetivos de Aprendizaje

- [ ] Explorar y documentar los endpoints principales de la WebLogic REST Management API (v2) navegando la documentación integrada en `/management/weblogic/latest`.
- [ ] Realizar operaciones CRUD sobre recursos WebLogic (servidores, datasources, deployments) usando `curl` desde la línea de comandos.
- [ ] Desarrollar un script Python 3 con la librería `requests` que implemente funciones reutilizables de administración (listar, crear, actualizar, eliminar recursos).
- [ ] Implementar autenticación HTTP Basic segura gestionando credenciales mediante variables de entorno, nunca en código fuente.
- [ ] Construir un script de monitoreo que consuma los endpoints de Health Check y emita alertas cuando un servidor no esté en estado `RUNNING`.

---

## 4. Prerequisitos

### Conocimiento previo
- Laboratorio **04-00-01 completado**: dominio `wl15_domain` con Administration Server (`AdminServer`) y al menos un Managed Server (`ms1`) operativos.
- Comprensión de los conceptos de WLST (lección 4.1): árboles de MBeans, flujo `edit/startEdit/save/activate`.
- Conocimiento básico de HTTP: métodos GET, POST, PATCH, DELETE; códigos de estado (200, 201, 400, 401, 404, 409); cabeceras; cuerpos JSON.
- Autenticación HTTP Basic: codificación Base64 de `usuario:contraseña`.

### Acceso y software requerido
- Usuario `oracle` con acceso al entorno de laboratorio.
- `curl` 7.76+ instalado y funcional.
- Python 3.11+ instalado con el módulo `requests`:
  ```bash
  pip3 install requests
  ```
- Administration Server de WebLogic 15 en ejecución en `localhost:7001`.
- Managed Server `ms1` en ejecución en `localhost:7003` (o el puerto configurado en el lab 04-00-01).

---

## 5. Entorno de Laboratorio

### Resumen de hardware recomendado

| Recurso       | Mínimo          | Recomendado     |
|---------------|-----------------|-----------------|
| RAM           | 16 GB           | 32 GB           |
| CPU           | 4 núcleos       | 8 núcleos       |
| Disco         | 100 GB SSD      | 200 GB SSD      |
| SO            | Oracle Linux 8/9 / RHEL 8/9 | ídem |

### Resumen de software relevante para este lab

| Software          | Versión          | Uso en este lab                        |
|-------------------|------------------|----------------------------------------|
| WebLogic Server   | 15.0             | Servidor objetivo de las llamadas REST |
| JDK               | 21 LTS           | Entorno de ejecución de WebLogic       |
| curl              | 7.76+            | Llamadas REST desde línea de comandos  |
| Python            | 3.11+            | Scripts de automatización              |
| requests (Python) | 2.31+            | Librería HTTP para los scripts         |
| Visual Studio Code| 1.89+            | Edición de scripts                     |

### Configuración inicial del entorno

Abre una terminal como usuario `oracle` y define las variables de entorno que usarás a lo largo de todo el laboratorio. **Nunca pongas contraseñas en el código fuente.**

```bash
# ── Variables de entorno del laboratorio ──────────────────────────────────────
export WL_ADMIN_URL="http://localhost:7001"
export WL_ADMIN_USER="weblogic"
export WL_ADMIN_PASS="Welcome1"          # Solo en lab; en producción usa vault
export WL_API_BASE="${WL_ADMIN_URL}/management/weblogic/latest"

# Verificar que el Administration Server está en ejecución
curl -s -o /dev/null -w "%{http_code}" \
  -u "${WL_ADMIN_USER}:${WL_ADMIN_PASS}" \
  "${WL_API_BASE}/domainConfig"
# Resultado esperado: 200
```

> ⚠️ **Nota de seguridad:** En este laboratorio académico se usan credenciales simplificadas. En producción, las credenciales deben almacenarse en un gestor de secretos (HashiCorp Vault, Oracle Wallet, etc.) y nunca en variables de entorno del shell de producción sin protección adicional.

Crea el directorio de trabajo para este laboratorio:

```bash
mkdir -p ~/lab04-00-02/{scripts,responses,logs}
cd ~/lab04-00-02
```

---

## 6. Pasos del Laboratorio

---

### Paso 1 — Explorar la documentación integrada de la REST API

**Objetivo:** Descubrir los endpoints disponibles en WebLogic 15 navegando la documentación interactiva integrada en el propio servidor.

#### Instrucciones

1. Accede al índice raíz de la API REST para obtener la lista de versiones disponibles:

```bash
curl -s \
  -u "${WL_ADMIN_USER}:${WL_ADMIN_PASS}" \
  -H "Accept: application/json" \
  "${WL_ADMIN_URL}/management/weblogic" \
  | python3 -m json.tool \
  | tee ~/lab04-00-02/responses/01_api_versions.json
```

2. Accede al endpoint raíz de la versión `latest` (v2) para ver las colecciones principales:

```bash
curl -s \
  -u "${WL_ADMIN_USER}:${WL_ADMIN_PASS}" \
  -H "Accept: application/json" \
  "${WL_API_BASE}" \
  | python3 -m json.tool \
  | tee ~/lab04-00-02/responses/02_api_root.json
```

3. Consulta el endpoint de configuración del dominio (`domainConfig`) para ver la estructura de configuración persistente (equivalente al árbol `config` en WLST):

```bash
curl -s \
  -u "${WL_ADMIN_USER}:${WL_ADMIN_PASS}" \
  -H "Accept: application/json" \
  "${WL_API_BASE}/domainConfig" \
  | python3 -m json.tool \
  | tee ~/lab04-00-02/responses/03_domainConfig.json
```

4. Consulta el endpoint de runtime del dominio (`domainRuntime`) — equivalente a `domainRuntime()` en WLST:

```bash
curl -s \
  -u "${WL_ADMIN_USER}:${WL_ADMIN_PASS}" \
  -H "Accept: application/json" \
  "${WL_API_BASE}/domainRuntime" \
  | python3 -m json.tool \
  | tee ~/lab04-00-02/responses/04_domainRuntime.json
```

5. Examina los archivos generados e identifica al menos **tres colecciones** (p. ej., `servers`, `JDBCSystemResources`, `deployments`). Anótalas en el fichero de trabajo:

```bash
echo "=== Colecciones identificadas ===" > ~/lab04-00-02/logs/exploracion.txt
grep '"links"' ~/lab04-00-02/responses/02_api_root.json \
  >> ~/lab04-00-02/logs/exploracion.txt
```

#### Salida esperada (fragmento de `02_api_root.json`)

```json
{
  "links": [
    { "rel": "self",          "href": "http://localhost:7001/management/weblogic/latest" },
    { "rel": "domainConfig",  "href": "http://localhost:7001/management/weblogic/latest/domainConfig" },
    { "rel": "domainRuntime", "href": "http://localhost:7001/management/weblogic/latest/domainRuntime" },
    { "rel": "serverConfig",  "href": "http://localhost:7001/management/weblogic/latest/serverConfig" },
    { "rel": "serverRuntime", "href": "http://localhost:7001/management/weblogic/latest/serverRuntime" },
    { "rel": "edit",          "href": "http://localhost:7001/management/weblogic/latest/edit" }
  ]
}
```

#### Verificación

```bash
# El código HTTP debe ser 200 para todos los endpoints explorados
curl -s -o /dev/null -w "domainConfig: %{http_code}\n" \
  -u "${WL_ADMIN_USER}:${WL_ADMIN_PASS}" \
  "${WL_API_BASE}/domainConfig"

curl -s -o /dev/null -w "domainRuntime: %{http_code}\n" \
  -u "${WL_ADMIN_USER}:${WL_ADMIN_PASS}" \
  "${WL_API_BASE}/domainRuntime"
```

Ambas líneas deben devolver `200`.

---

### Paso 2 — Operaciones GET: listar servidores y sus estados

**Objetivo:** Consultar el listado de servidores del dominio y sus estados de ciclo de vida usando la REST API.

#### Instrucciones

1. Lista todos los servidores definidos en la configuración del dominio:

```bash
curl -s \
  -u "${WL_ADMIN_USER}:${WL_ADMIN_PASS}" \
  -H "Accept: application/json" \
  "${WL_API_BASE}/domainConfig/servers" \
  | python3 -m json.tool \
  | tee ~/lab04-00-02/responses/05_servers_config.json
```

2. Consulta el estado de ciclo de vida de todos los servidores (equivalente a navegar `domainRuntime/ServerLifeCycleRuntimes` en WLST):

```bash
curl -s \
  -u "${WL_ADMIN_USER}:${WL_ADMIN_PASS}" \
  -H "Accept: application/json" \
  "${WL_API_BASE}/domainRuntime/serverLifeCycleRuntimes" \
  | python3 -m json.tool \
  | tee ~/lab04-00-02/responses/06_server_lifecycle.json
```

3. Consulta el estado específico del servidor `AdminServer`:

```bash
curl -s \
  -u "${WL_ADMIN_USER}:${WL_ADMIN_PASS}" \
  -H "Accept: application/json" \
  "${WL_API_BASE}/domainRuntime/serverLifeCycleRuntimes/AdminServer" \
  | python3 -m json.tool \
  | tee ~/lab04-00-02/responses/07_adminserver_state.json
```

4. Extrae únicamente el campo `state` usando Python:

```bash
python3 -c "
import json, sys
with open('responses/07_adminserver_state.json') as f:
    data = json.load(f)
print('Estado AdminServer:', data.get('state', 'N/A'))
"
```

5. Consulta las métricas JVM del Administration Server (equivalente a `serverRuntime()/JVMRuntime` en WLST):

```bash
curl -s \
  -u "${WL_ADMIN_USER}:${WL_ADMIN_PASS}" \
  -H "Accept: application/json" \
  "${WL_API_BASE}/serverRuntime/JVMRuntime" \
  | python3 -m json.tool \
  | tee ~/lab04-00-02/responses/08_jvm_runtime.json
```

#### Salida esperada

```json
{
  "name": "AdminServer",
  "state": "RUNNING",
  "stateVal": 2,
  "links": [...]
}
```

#### Verificación

```bash
# El estado debe ser RUNNING
python3 -c "
import json
with open('responses/07_adminserver_state.json') as f:
    d = json.load(f)
state = d.get('state','UNKNOWN')
print(f'[OK] Estado: {state}' if state == 'RUNNING' else f'[ERROR] Estado inesperado: {state}')
"
```

---

### Paso 3 — Operaciones POST y PATCH: crear y actualizar un Data Source

**Objetivo:** Crear un Data Source JDBC mediante la REST API y actualizar sus propiedades usando PATCH, replicando en REST el flujo `edit/startEdit/save/activate` de WLST.

> **Nota conceptual:** La REST API de WebLogic 15 gestiona los cambios de configuración a través del endpoint `/edit`. Al igual que en WLST con `edit()/startEdit()`, es necesario iniciar una sesión de edición, realizar los cambios, y luego confirmarlos (`activate`).

#### Instrucciones

1. Inicia una sesión de edición (equivalente a `edit()` + `startEdit()` en WLST):

```bash
curl -s -X POST \
  -u "${WL_ADMIN_USER}:${WL_ADMIN_PASS}" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "X-Requested-By: curl-lab04" \
  "${WL_API_BASE}/edit/changeManager/startEdit" \
  | python3 -m json.tool \
  | tee ~/lab04-00-02/responses/09_start_edit.json
```

2. Crea el recurso `JDBCSystemResource` con nombre `LabDataSource`:

```bash
curl -s -X POST \
  -u "${WL_ADMIN_USER}:${WL_ADMIN_PASS}" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "X-Requested-By: curl-lab04" \
  -d '{
    "name": "LabDataSource",
    "targets": [{"identity": ["servers", "AdminServer"]}]
  }' \
  "${WL_API_BASE}/edit/JDBCSystemResources" \
  | python3 -m json.tool \
  | tee ~/lab04-00-02/responses/10_create_datasource.json
```

3. Configura el driver JDBC y la URL de conexión con PATCH:

```bash
curl -s -X PATCH \
  -u "${WL_ADMIN_USER}:${WL_ADMIN_PASS}" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "X-Requested-By: curl-lab04" \
  -d '{
    "JDBCDriverParams": {
      "driverName": "org.h2.Driver",
      "URL": "jdbc:h2:mem:labdb;DB_CLOSE_DELAY=-1",
      "passwordEncrypted": "labpassword"
    },
    "JDBCConnectionPoolParams": {
      "initialCapacity": 1,
      "maxCapacity": 10,
      "testConnectionsOnReserve": true,
      "testTableName": "SQL SELECT 1"
    },
    "JDBCDataSourceParams": {
      "JNDINames": ["jdbc/LabDS"]
    }
  }' \
  "${WL_API_BASE}/edit/JDBCSystemResources/LabDataSource/JDBCResource/LabDataSource" \
  | python3 -m json.tool \
  | tee ~/lab04-00-02/responses/11_patch_datasource.json
```

4. Guarda y activa los cambios (equivalente a `save()` + `activate()` en WLST):

```bash
# Guardar
curl -s -X POST \
  -u "${WL_ADMIN_USER}:${WL_ADMIN_PASS}" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "X-Requested-By: curl-lab04" \
  "${WL_API_BASE}/edit/changeManager/save" \
  | python3 -m json.tool \
  | tee ~/lab04-00-02/responses/12_save.json

# Activar
curl -s -X POST \
  -u "${WL_ADMIN_USER}:${WL_ADMIN_PASS}" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "X-Requested-By: curl-lab04" \
  -d '{"block": true}' \
  "${WL_API_BASE}/edit/changeManager/activate" \
  | python3 -m json.tool \
  | tee ~/lab04-00-02/responses/13_activate.json
```

5. Verifica que el Data Source fue creado consultando su configuración:

```bash
curl -s \
  -u "${WL_ADMIN_USER}:${WL_ADMIN_PASS}" \
  -H "Accept: application/json" \
  "${WL_API_BASE}/domainConfig/JDBCSystemResources/LabDataSource" \
  | python3 -m json.tool
```

#### Salida esperada (fragmento de `10_create_datasource.json`)

```json
{
  "name": "LabDataSource",
  "links": [
    { "rel": "self", "href": "http://localhost:7001/management/weblogic/latest/edit/JDBCSystemResources/LabDataSource" }
  ]
}
```

#### Verificación

```bash
# El código HTTP del GET debe ser 200 si el DS fue creado correctamente
curl -s -o /dev/null -w "HTTP %{http_code}\n" \
  -u "${WL_ADMIN_USER}:${WL_ADMIN_PASS}" \
  "${WL_API_BASE}/domainConfig/JDBCSystemResources/LabDataSource"
# Esperado: HTTP 200
```

---

### Paso 4 — Operación DELETE: eliminar un recurso

**Objetivo:** Eliminar el Data Source creado en el paso anterior usando DELETE, completando el ciclo CRUD.

#### Instrucciones

1. Inicia una nueva sesión de edición:

```bash
curl -s -X POST \
  -u "${WL_ADMIN_USER}:${WL_ADMIN_PASS}" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "X-Requested-By: curl-lab04" \
  "${WL_API_BASE}/edit/changeManager/startEdit" \
  | python3 -m json.tool
```

2. Elimina el Data Source `LabDataSource`:

```bash
curl -s -X DELETE \
  -u "${WL_ADMIN_USER}:${WL_ADMIN_PASS}" \
  -H "Accept: application/json" \
  -H "X-Requested-By: curl-lab04" \
  "${WL_API_BASE}/edit/JDBCSystemResources/LabDataSource" \
  -w "\nHTTP Status: %{http_code}\n" \
  | tee ~/lab04-00-02/responses/14_delete_datasource.json
```

3. Activa los cambios:

```bash
curl -s -X POST \
  -u "${WL_ADMIN_USER}:${WL_ADMIN_PASS}" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "X-Requested-By: curl-lab04" \
  -d '{"block": true}' \
  "${WL_API_BASE}/edit/changeManager/activate" \
  | python3 -m json.tool
```

4. Confirma que el recurso ya no existe:

```bash
curl -s -o /dev/null -w "HTTP %{http_code}\n" \
  -u "${WL_ADMIN_USER}:${WL_ADMIN_PASS}" \
  "${WL_API_BASE}/domainConfig/JDBCSystemResources/LabDataSource"
# Esperado: HTTP 404
```

#### Salida esperada

```
HTTP Status: 204
```
*(204 No Content indica eliminación exitosa)*

#### Verificación

La consulta del paso 4 debe devolver `HTTP 404`, confirmando que el recurso fue eliminado correctamente.

---

### Paso 5 — Desarrollar el mini-framework Python de administración

**Objetivo:** Crear un módulo Python 3 reutilizable (`wl_admin.py`) que encapsule las operaciones REST más comunes, usando credenciales desde variables de entorno.

#### Instrucciones

1. Crea el archivo del framework en el directorio de scripts:

```bash
cat > ~/lab04-00-02/scripts/wl_admin.py << 'PYEOF'
#!/usr/bin/env python3
"""
wl_admin.py — Mini-framework de administración WebLogic REST API
Lab 04-00-02 | WebLogic 15 / Java 21

IMPORTANTE: Las credenciales se leen SIEMPRE desde variables de entorno.
Nunca almacenar contraseñas en este archivo.

Uso:
    export WL_ADMIN_URL="http://localhost:7001"
    export WL_ADMIN_USER="weblogic"
    export WL_ADMIN_PASS="Welcome1"
    python3 wl_admin.py
"""

import os
import sys
import json
import logging
from typing import Optional, Dict, Any, List

import requests
from requests.auth import HTTPBasicAuth

# ── Configuración de logging ───────────────────────────────────────────────────
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
)
log = logging.getLogger("wl_admin")


# ── Clase principal del cliente REST ──────────────────────────────────────────
class WebLogicAdminClient:
    """Cliente REST para WebLogic Management API v2."""

    DEFAULT_HEADERS = {
        "Accept": "application/json",
        "Content-Type": "application/json",
        "X-Requested-By": "wl_admin_py",
    }

    def __init__(
        self,
        base_url: Optional[str] = None,
        username: Optional[str] = None,
        password: Optional[str] = None,
        verify_ssl: bool = False,  # En producción: True con truststore adecuado
    ):
        # Leer credenciales desde variables de entorno (seguro)
        self.base_url = (base_url or os.environ.get("WL_ADMIN_URL", "http://localhost:7001")).rstrip("/")
        self.username = username or os.environ.get("WL_ADMIN_USER")
        self.password = password or os.environ.get("WL_ADMIN_PASS")
        self.verify_ssl = verify_ssl

        if not self.username or not self.password:
            raise EnvironmentError(
                "Credenciales no encontradas. Define WL_ADMIN_USER y WL_ADMIN_PASS "
                "como variables de entorno."
            )

        self.api_base = f"{self.base_url}/management/weblogic/latest"
        self.auth = HTTPBasicAuth(self.username, self.password)
        self.session = requests.Session()
        self.session.auth = self.auth
        self.session.headers.update(self.DEFAULT_HEADERS)
        self.session.verify = self.verify_ssl

        log.info("Cliente WebLogic REST inicializado: %s", self.api_base)

    # ── Métodos HTTP base ──────────────────────────────────────────────────────

    def _get(self, path: str) -> Dict[str, Any]:
        url = f"{self.api_base}{path}"
        log.debug("GET %s", url)
        resp = self.session.get(url)
        resp.raise_for_status()
        return resp.json()

    def _post(self, path: str, payload: Optional[Dict] = None) -> Dict[str, Any]:
        url = f"{self.api_base}{path}"
        log.debug("POST %s | payload=%s", url, payload)
        resp = self.session.post(url, json=payload or {})
        resp.raise_for_status()
        return resp.json() if resp.content else {}

    def _patch(self, path: str, payload: Dict[str, Any]) -> Dict[str, Any]:
        url = f"{self.api_base}{path}"
        log.debug("PATCH %s | payload=%s", url, payload)
        resp = self.session.patch(url, json=payload)
        resp.raise_for_status()
        return resp.json() if resp.content else {}

    def _delete(self, path: str) -> int:
        url = f"{self.api_base}{path}"
        log.debug("DELETE %s", url)
        resp = self.session.delete(url)
        resp.raise_for_status()
        return resp.status_code

    # ── Gestión de sesión de edición (equivalente a edit/startEdit/save/activate) ──

    def start_edit(self) -> Dict:
        log.info("Iniciando sesión de edición...")
        return self._post("/edit/changeManager/startEdit")

    def save_edit(self) -> Dict:
        log.info("Guardando cambios (save)...")
        return self._post("/edit/changeManager/save")

    def activate_edit(self, block: bool = True) -> Dict:
        log.info("Activando cambios (activate)...")
        return self._post("/edit/changeManager/activate", {"block": block})

    def cancel_edit(self) -> Dict:
        log.warning("Cancelando sesión de edición (undoEdit)...")
        return self._post("/edit/changeManager/cancelEdit")

    # ── Operaciones sobre servidores ──────────────────────────────────────────

    def list_servers(self) -> List[Dict]:
        """Lista todos los servidores del dominio con su estado."""
        config = self._get("/domainConfig/servers")
        lifecycle = self._get("/domainRuntime/serverLifeCycleRuntimes")
        servers = config.get("items", [])
        states = {s["name"]: s.get("state", "UNKNOWN")
                  for s in lifecycle.get("items", [])}
        for srv in servers:
            srv["currentState"] = states.get(srv["name"], "N/A")
        return servers

    def get_server_state(self, server_name: str) -> str:
        """Devuelve el estado de ciclo de vida de un servidor."""
        data = self._get(f"/domainRuntime/serverLifeCycleRuntimes/{server_name}")
        return data.get("state", "UNKNOWN")

    # ── Operaciones sobre Data Sources ────────────────────────────────────────

    def list_datasources(self) -> List[Dict]:
        """Lista todos los JDBCSystemResources del dominio."""
        data = self._get("/domainConfig/JDBCSystemResources")
        return data.get("items", [])

    def create_datasource(
        self,
        name: str,
        jndi_name: str,
        driver: str,
        url: str,
        password: str,
        target_server: str = "AdminServer",
        max_capacity: int = 10,
    ) -> Dict:
        """Crea un Data Source JDBC con sesión de edición completa."""
        self.start_edit()
        try:
            # Crear el recurso
            self._post(
                "/edit/JDBCSystemResources",
                {
                    "name": name,
                    "targets": [{"identity": ["servers", target_server]}],
                },
            )
            # Configurar propiedades JDBC
            self._patch(
                f"/edit/JDBCSystemResources/{name}/JDBCResource/{name}",
                {
                    "JDBCDriverParams": {
                        "driverName": driver,
                        "URL": url,
                        "passwordEncrypted": password,
                    },
                    "JDBCConnectionPoolParams": {
                        "initialCapacity": 1,
                        "maxCapacity": max_capacity,
                        "testConnectionsOnReserve": True,
                        "testTableName": "SQL SELECT 1",
                    },
                    "JDBCDataSourceParams": {"JNDINames": [jndi_name]},
                },
            )
            self.save_edit()
            result = self.activate_edit()
            log.info("DataSource '%s' creado y activado correctamente.", name)
            return result
        except Exception as exc:
            log.error("Error creando DataSource '%s': %s", name, exc)
            self.cancel_edit()
            raise

    def delete_datasource(self, name: str) -> int:
        """Elimina un Data Source JDBC."""
        self.start_edit()
        try:
            status = self._delete(f"/edit/JDBCSystemResources/{name}")
            self.save_edit()
            self.activate_edit()
            log.info("DataSource '%s' eliminado (HTTP %d).", name, status)
            return status
        except Exception as exc:
            log.error("Error eliminando DataSource '%s': %s", name, exc)
            self.cancel_edit()
            raise

    # ── Métricas JVM ──────────────────────────────────────────────────────────

    def get_jvm_metrics(self) -> Dict[str, Any]:
        """Devuelve métricas JVM del Administration Server."""
        data = self._get("/serverRuntime/JVMRuntime")
        return {
            "heapFreeCurrent":  data.get("heapFreeCurrent"),
            "heapSizeCurrent":  data.get("heapSizeCurrent"),
            "heapSizeMax":      data.get("heapSizeMax"),
            "uptime":           data.get("uptime"),
        }


# ── Función de demostración ───────────────────────────────────────────────────

def demo():
    client = WebLogicAdminClient()

    print("\n" + "=" * 60)
    print("  DEMO: Mini-Framework WebLogic REST Admin")
    print("=" * 60)

    # 1. Listar servidores
    print("\n[1] Servidores del dominio:")
    for srv in client.list_servers():
        print(f"    • {srv['name']:20s} — Estado: {srv.get('currentState', 'N/A')}")

    # 2. Métricas JVM
    print("\n[2] Métricas JVM del AdminServer:")
    jvm = client.get_jvm_metrics()
    heap_used_mb = (jvm["heapSizeCurrent"] - jvm["heapFreeCurrent"]) / (1024 * 1024)
    heap_max_mb  = jvm["heapSizeMax"] / (1024 * 1024)
    print(f"    Heap usado:  {heap_used_mb:.1f} MB")
    print(f"    Heap máximo: {heap_max_mb:.1f} MB")
    print(f"    Uptime:      {jvm['uptime']} ms")

    # 3. Crear Data Source de prueba
    print("\n[3] Creando DataSource 'DemoDS'...")
    client.create_datasource(
        name="DemoDS",
        jndi_name="jdbc/DemoDS",
        driver="org.h2.Driver",
        url="jdbc:h2:mem:demodb;DB_CLOSE_DELAY=-1",
        password="demopassword",
        target_server="AdminServer",
        max_capacity=5,
    )

    # 4. Listar Data Sources
    print("\n[4] Data Sources en el dominio:")
    for ds in client.list_datasources():
        print(f"    • {ds['name']}")

    # 5. Eliminar Data Source de prueba
    print("\n[5] Eliminando DataSource 'DemoDS'...")
    client.delete_datasource("DemoDS")
    print("    Eliminado correctamente.")

    print("\n" + "=" * 60)
    print("  DEMO completada con éxito.")
    print("=" * 60 + "\n")


if __name__ == "__main__":
    demo()
PYEOF

chmod +x ~/lab04-00-02/scripts/wl_admin.py
```

2. Ejecuta el script de demostración:

```bash
cd ~/lab04-00-02
python3 scripts/wl_admin.py
```

#### Salida esperada

```
2024-10-15 10:23:01 [INFO] Cliente WebLogic REST inicializado: http://localhost:7001/management/weblogic/latest

============================================================
  DEMO: Mini-Framework WebLogic REST Admin
============================================================

[1] Servidores del dominio:
    • AdminServer            — Estado: RUNNING
    • ms1                    — Estado: RUNNING

[2] Métricas JVM del AdminServer:
    Heap usado:  312.4 MB
    Heap máximo: 1024.0 MB
    Uptime:      3842156 ms

[3] Creando DataSource 'DemoDS'...
2024-10-15 10:23:02 [INFO] Iniciando sesión de edición...
2024-10-15 10:23:03 [INFO] DataSource 'DemoDS' creado y activado correctamente.

[4] Data Sources en el dominio:
    • DemoDS

[5] Eliminando DataSource 'DemoDS'...
    Eliminado correctamente.

============================================================
  DEMO completada con éxito.
============================================================
```

#### Verificación

```bash
# Sin errores en la ejecución y código de salida 0
python3 scripts/wl_admin.py && echo "[OK] Script ejecutado correctamente"
```

---

### Paso 6 — Implementar el script de monitoreo con Health Check

**Objetivo:** Crear un script de monitoreo (`wl_health_monitor.py`) que consulte el estado de todos los servidores y emita alertas cuando alguno no esté en estado `RUNNING`.

#### Instrucciones

1. Crea el script de monitoreo:

```bash
cat > ~/lab04-00-02/scripts/wl_health_monitor.py << 'PYEOF'
#!/usr/bin/env python3
"""
wl_health_monitor.py — Monitor de salud de servidores WebLogic
Lab 04-00-02 | WebLogic 15 / Java 21

Consulta el estado de todos los servidores del dominio y de las
aplicaciones desplegadas usando la REST API de WebLogic.
Emite alertas cuando un recurso no está en estado RUNNING/OK.

Credenciales requeridas como variables de entorno:
    WL_ADMIN_URL, WL_ADMIN_USER, WL_ADMIN_PASS
"""

import os
import sys
import json
import logging
from datetime import datetime
from typing import List, Dict, Tuple

import requests
from requests.auth import HTTPBasicAuth

# ── Logging ───────────────────────────────────────────────────────────────────
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
)
log = logging.getLogger("wl_health_monitor")

# ── Constantes ────────────────────────────────────────────────────────────────
HEALTHY_SERVER_STATES = {"RUNNING"}
HEALTHY_APP_STATES    = {"STATE_ACTIVE", "ACTIVE"}
ALERT_PREFIX          = "🔴 [ALERTA]"
OK_PREFIX             = "🟢 [OK]"


def get_env_or_fail(var: str) -> str:
    value = os.environ.get(var)
    if not value:
        log.error("Variable de entorno '%s' no definida.", var)
        sys.exit(1)
    return value


def build_session(username: str, password: str, verify_ssl: bool = False) -> requests.Session:
    session = requests.Session()
    session.auth = HTTPBasicAuth(username, password)
    session.headers.update({
        "Accept": "application/json",
        "X-Requested-By": "wl_health_monitor",
    })
    session.verify = verify_ssl
    return session


def check_servers(session: requests.Session, api_base: str) -> Tuple[List[Dict], bool]:
    """Consulta el estado de ciclo de vida de todos los servidores."""
    url = f"{api_base}/domainRuntime/serverLifeCycleRuntimes"
    resp = session.get(url)
    resp.raise_for_status()
    servers = resp.json().get("items", [])
    alerts = []
    all_healthy = True

    for srv in servers:
        name  = srv.get("name", "UNKNOWN")
        state = srv.get("state", "UNKNOWN")
        healthy = state in HEALTHY_SERVER_STATES
        if not healthy:
            all_healthy = False
            alerts.append({"resource": "SERVER", "name": name, "state": state})

    return alerts, all_healthy


def check_deployments(session: requests.Session, api_base: str) -> Tuple[List[Dict], bool]:
    """Consulta el estado de las aplicaciones desplegadas."""
    url = f"{api_base}/domainRuntime/deploymentManager/deploymentProgressObjects"
    try:
        resp = session.get(url)
        if resp.status_code == 404:
            log.info("Endpoint deploymentProgressObjects no disponible (sin despliegues activos).")
            return [], True
        resp.raise_for_status()
        apps = resp.json().get("items", [])
        alerts = []
        all_healthy = True
        for app in apps:
            name  = app.get("applicationName", "UNKNOWN")
            state = app.get("state", "UNKNOWN")
            healthy = state in HEALTHY_APP_STATES
            if not healthy:
                all_healthy = False
                alerts.append({"resource": "APP", "name": name, "state": state})
        return alerts, all_healthy
    except Exception as exc:
        log.warning("No se pudo consultar deployments: %s", exc)
        return [], True


def check_jvm_health(session: requests.Session, api_base: str, heap_warn_pct: float = 80.0) -> Tuple[List[Dict], bool]:
    """Alerta si el uso de heap supera el umbral configurado."""
    url = f"{api_base}/serverRuntime/JVMRuntime"
    resp = session.get(url)
    resp.raise_for_status()
    data = resp.json()

    heap_free = data.get("heapFreeCurrent", 0)
    heap_size = data.get("heapSizeCurrent", 1)
    heap_max  = data.get("heapSizeMax", 1)
    used_pct  = ((heap_size - heap_free) / heap_max) * 100 if heap_max > 0 else 0

    alerts = []
    all_healthy = True
    if used_pct >= heap_warn_pct:
        all_healthy = False
        alerts.append({
            "resource": "JVM_HEAP",
            "name": "AdminServer",
            "state": f"WARN — Uso heap: {used_pct:.1f}% (umbral: {heap_warn_pct}%)",
        })
    else:
        log.info("JVM Heap AdminServer: %.1f%% usado (umbral: %.1f%%)", used_pct, heap_warn_pct)

    return alerts, all_healthy


def generate_report(all_alerts: List[Dict], timestamp: str) -> str:
    """Genera un informe de salud en formato texto."""
    lines = [
        "=" * 65,
        f"  INFORME DE SALUD WebLogic — {timestamp}",
        "=" * 65,
    ]
    if not all_alerts:
        lines.append(f"\n  {OK_PREFIX} Todos los recursos están en estado saludable.\n")
    else:
        lines.append(f"\n  Se encontraron {len(all_alerts)} alerta(s):\n")
        for alert in all_alerts:
            lines.append(
                f"  {ALERT_PREFIX} [{alert['resource']}] {alert['name']:25s} — {alert['state']}"
            )
        lines.append("")
    lines.append("=" * 65)
    return "\n".join(lines)


def main():
    # Leer configuración desde variables de entorno
    admin_url  = get_env_or_fail("WL_ADMIN_URL").rstrip("/")
    username   = get_env_or_fail("WL_ADMIN_USER")
    password   = get_env_or_fail("WL_ADMIN_PASS")
    api_base   = f"{admin_url}/management/weblogic/latest"
    heap_warn  = float(os.environ.get("WL_HEAP_WARN_PCT", "80"))

    session    = build_session(username, password)
    timestamp  = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    all_alerts = []
    exit_code  = 0

    log.info("Iniciando comprobación de salud: %s", api_base)

    # Comprobar servidores
    srv_alerts, srv_ok = check_servers(session, api_base)
    all_alerts.extend(srv_alerts)
    if not srv_ok:
        exit_code = 1

    # Comprobar despliegues
    app_alerts, app_ok = check_deployments(session, api_base)
    all_alerts.extend(app_alerts)
    if not app_ok:
        exit_code = 1

    # Comprobar JVM
    jvm_alerts, jvm_ok = check_jvm_health(session, api_base, heap_warn)
    all_alerts.extend(jvm_alerts)
    if not jvm_ok:
        exit_code = 1

    # Imprimir informe
    report = generate_report(all_alerts, timestamp)
    print(report)

    # Guardar informe en fichero de log
    log_path = os.path.join(
        os.path.dirname(__file__), "..", "logs",
        f"health_{datetime.now().strftime('%Y%m%d_%H%M%S')}.txt"
    )
    os.makedirs(os.path.dirname(log_path), exist_ok=True)
    with open(log_path, "w") as f:
        f.write(report + "\n")
    log.info("Informe guardado en: %s", os.path.abspath(log_path))

    sys.exit(exit_code)


if __name__ == "__main__":
    main()
PYEOF

chmod +x ~/lab04-00-02/scripts/wl_health_monitor.py
```

2. Ejecuta el monitor con todos los servidores en estado `RUNNING`:

```bash
cd ~/lab04-00-02
python3 scripts/wl_health_monitor.py
```

3. Simula una alerta cambiando el umbral de heap al 1% (siempre se superará):

```bash
WL_HEAP_WARN_PCT=1 python3 scripts/wl_health_monitor.py
```

4. Verifica que el script devuelve código de salida `1` cuando hay alertas:

```bash
WL_HEAP_WARN_PCT=1 python3 scripts/wl_health_monitor.py
echo "Código de salida: $?"
# Esperado: Código de salida: 1
```

#### Salida esperada (sistema saludable)

```
=================================================================
  INFORME DE SALUD WebLogic — 2024-10-15 10:35:22
=================================================================

  🟢 [OK] Todos los recursos están en estado saludable.

=================================================================
```

#### Salida esperada (con alerta de heap al 1%)

```
=================================================================
  INFORME DE SALUD WebLogic — 2024-10-15 10:36:01
=================================================================

  Se encontraron 1 alerta(s):

  🔴 [ALERTA] [JVM_HEAP] AdminServer          — WARN — Uso heap: 30.5% (umbral: 1.0%)

=================================================================
```

#### Verificación

```bash
# Ejecución normal: código de salida 0
python3 scripts/wl_health_monitor.py
echo "Código de salida normal: $?"

# Verificar que se generó el fichero de log
ls -la ~/lab04-00-02/logs/health_*.txt
```

---

## 7. Validación y Pruebas Globales

Ejecuta las siguientes verificaciones para confirmar que todos los objetivos del laboratorio han sido cumplidos:

```bash
#!/bin/bash
# validate_lab04_00_02.sh — Script de validación global

echo "═══════════════════════════════════════════════════════"
echo "  VALIDACIÓN LAB 04-00-02"
echo "═══════════════════════════════════════════════════════"
PASS=0; FAIL=0

check() {
  local desc="$1"; local cmd="$2"; local expected="$3"
  result=$(eval "$cmd" 2>/dev/null)
  if echo "$result" | grep -q "$expected"; then
    echo "  ✅ $desc"
    ((PASS++))
  else
    echo "  ❌ $desc (obtenido: $result)"
    ((FAIL++))
  fi
}

# 1. API REST accesible
check "REST API v2 accesible (HTTP 200)" \
  "curl -s -o /dev/null -w '%{http_code}' -u '${WL_ADMIN_USER}:${WL_ADMIN_PASS}' '${WL_API_BASE}'" \
  "200"

# 2. Endpoint domainConfig disponible
check "Endpoint domainConfig disponible" \
  "curl -s -o /dev/null -w '%{http_code}' -u '${WL_ADMIN_USER}:${WL_ADMIN_PASS}' '${WL_API_BASE}/domainConfig'" \
  "200"

# 3. Endpoint serverLifeCycleRuntimes disponible
check "Endpoint serverLifeCycleRuntimes disponible" \
  "curl -s -o /dev/null -w '%{http_code}' -u '${WL_ADMIN_USER}:${WL_ADMIN_PASS}' '${WL_API_BASE}/domainRuntime/serverLifeCycleRuntimes'" \
  "200"

# 4. AdminServer en estado RUNNING
check "AdminServer en estado RUNNING" \
  "curl -s -u '${WL_ADMIN_USER}:${WL_ADMIN_PASS}' '${WL_API_BASE}/domainRuntime/serverLifeCycleRuntimes/AdminServer'" \
  "RUNNING"

# 5. Script wl_admin.py existe y es ejecutable
check "Script wl_admin.py presente" \
  "test -x ~/lab04-00-02/scripts/wl_admin.py && echo 'OK'" \
  "OK"

# 6. Script wl_health_monitor.py existe y es ejecutable
check "Script wl_health_monitor.py presente" \
  "test -x ~/lab04-00-02/scripts/wl_health_monitor.py && echo 'OK'" \
  "OK"

# 7. Python requests disponible
check "Módulo Python requests disponible" \
  "python3 -c 'import requests; print(requests.__version__)'" \
  "."

# 8. Archivos de respuestas generados
check "Respuestas curl almacenadas (mínimo 5 ficheros)" \
  "ls ~/lab04-00-02/responses/*.json 2>/dev/null | wc -l" \
  "[5-9]\|1[0-9]"

# 9. Monitor genera log
check "Monitor de salud genera fichero de log" \
  "python3 ~/lab04-00-02/scripts/wl_health_monitor.py > /dev/null 2>&1 && ls ~/lab04-00-02/logs/health_*.txt | wc -l" \
  "[1-9]"

echo "═══════════════════════════════════════════════════════"
echo "  Resultado: ${PASS} OK | ${FAIL} FALLOS"
echo "═══════════════════════════════════════════════════════"
[ $FAIL -eq 0 ] && exit 0 || exit 1
```

```bash
chmod +x ~/lab04-00-02/validate_lab04_00_02.sh
cd ~/lab04-00-02
bash validate_lab04_00_02.sh
```

**Resultado esperado:** `Resultado: 9 OK | 0 FALLOS`

---

## 8. Resolución de Problemas

### Problema 1: Error 401 Unauthorized en todas las llamadas curl

**Síntomas:**
```
{"title": "Unauthorized", "status": 401, "detail": "..."}
```
Todas las llamadas `curl` devuelven HTTP 401, incluso con las credenciales correctas.

**Causa:**
Las variables de entorno `WL_ADMIN_USER` o `WL_ADMIN_PASS` no están definidas en la sesión actual, o contienen caracteres especiales (como `!` en bash) que el shell interpreta antes de pasarlos a `curl`. También puede ocurrir si la contraseña contiene `@` o `#` sin escapar.

**Solución:**
```bash
# Verificar que las variables están definidas
echo "Usuario: ${WL_ADMIN_USER}"
echo "URL: ${WL_ADMIN_URL}"

# Si la contraseña contiene caracteres especiales, usar comillas simples
export WL_ADMIN_PASS='Welcome1!'   # Comillas simples evitan interpretación bash

# Alternativa: pasar credenciales directamente en curl para diagnóstico
curl -v -u "weblogic:Welcome1" \
  "http://localhost:7001/management/weblogic/latest/domainConfig"

# Verificar que el Administration Server está en ejecución
curl -s -o /dev/null -w "%{http_code}" http://localhost:7001/console/
# Debe devolver 200 o 302

# Si el problema persiste, verificar que el usuario existe en el dominio
# con WLST (referencia a lección 4.1):
# connect('weblogic','Welcome1','t3://localhost:7001')
# ls('/SecurityConfiguration/base_domain/Realms/myrealm/AuthenticationProviders')
```

---

### Problema 2: Error 409 Conflict al iniciar sesión de edición

**Síntomas:**
```json
{
  "title": "Conflict",
  "status": 409,
  "detail": "Another user currently holds the edit lock"
}
```
El endpoint `startEdit` devuelve HTTP 409, impidiendo cualquier cambio de configuración.

**Causa:**
Una sesión de edición previa (de este laboratorio u otra sesión de WLST/consola) quedó bloqueada sin ser confirmada ni cancelada. WebLogic 15 solo permite una sesión de edición activa a la vez en el dominio, exactamente igual que el bloqueo que ocurre con `edit()/startEdit()` en WLST.

**Solución:**
```bash
# 1. Identificar quién tiene el bloqueo
curl -s \
  -u "${WL_ADMIN_USER}:${WL_ADMIN_PASS}" \
  -H "Accept: application/json" \
  "${WL_API_BASE}/edit/changeManager" \
  | python3 -m json.tool
# Buscar el campo "lockOwner" en la respuesta

# 2. Cancelar la sesión de edición bloqueada
curl -s -X POST \
  -u "${WL_ADMIN_USER}:${WL_ADMIN_PASS}" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "X-Requested-By: curl-lab04" \
  "${WL_API_BASE}/edit/changeManager/cancelEdit" \
  | python3 -m json.tool
# Respuesta esperada: HTTP 200 con estado "unlocked"

# 3. Verificar que el bloqueo fue liberado
curl -s \
  -u "${WL_ADMIN_USER}:${WL_ADMIN_PASS}" \
  -H "Accept: application/json" \
  "${WL_API_BASE}/edit/changeManager" \
  | python3 -m json.tool
# El campo "lockOwner" debe estar vacío o ausente

# 4. Si el bloqueo persiste, también puede liberarse desde WLST
# (referencia a lección 4.1):
# connect('weblogic','Welcome1','t3://localhost:7001')
# edit()
# undoEdit(defaultAnswer='y')
# disconnect()
```

---

## 9. Limpieza del Entorno

Ejecuta los siguientes comandos para dejar el entorno en el estado inicial, listo para el siguiente laboratorio:

```bash
# ── 1. Eliminar cualquier Data Source residual del laboratorio ─────────────────
for DS_NAME in "LabDataSource" "DemoDS" "TestDS"; do
  HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
    -u "${WL_ADMIN_USER}:${WL_ADMIN_PASS}" \
    "${WL_API_BASE}/domainConfig/JDBCSystemResources/${DS_NAME}")
  if [ "$HTTP_CODE" = "200" ]; then
    echo "Eliminando Data Source residual: ${DS_NAME}"
    curl -s -X POST \
      -u "${WL_ADMIN_USER}:${WL_ADMIN_PASS}" \
      -H "Content-Type: application/json" \
      -H "Accept: application/json" \
      -H "X-Requested-By: curl-lab04-cleanup" \
      "${WL_API_BASE}/edit/changeManager/startEdit" > /dev/null

    curl -s -X DELETE \
      -u "${WL_ADMIN_USER}:${WL_ADMIN_PASS}" \
      -H "Accept: application/json" \
      -H "X-Requested-By: curl-lab04-cleanup" \
      "${WL_API_BASE}/edit/JDBCSystemResources/${DS_NAME}" > /dev/null

    curl -s -X POST \
      -u "${WL_ADMIN_USER}:${WL_ADMIN_PASS}" \
      -H "Content-Type: application/json" \
      -H "Accept: application/json" \
      -H "X-Requested-By: curl-lab04-cleanup" \
      -d '{"block": true}' \
      "${WL_API_BASE}/edit/changeManager/activate" > /dev/null
    echo "  ✅ ${DS_NAME} eliminado."
  else
    echo "  ℹ️  ${DS_NAME} no existe, nada que limpiar."
  fi
done

# ── 2. Cancelar cualquier sesión de edición abierta ───────────────────────────
echo "Cancelando sesiones de edición abiertas..."
curl -s -X POST \
  -u "${WL_ADMIN_USER}:${WL_ADMIN_PASS}" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "X-Requested-By: curl-lab04-cleanup" \
  "${WL_API_BASE}/edit/changeManager/cancelEdit" > /dev/null
echo "  ✅ Sesiones de edición canceladas."

# ── 3. Archivar los artefactos del laboratorio ────────────────────────────────
ARCHIVE_NAME="lab04-00-02_$(date +%Y%m%d_%H%M%S).tar.gz"
tar -czf ~/${ARCHIVE_NAME} ~/lab04-00-02/
echo "  ✅ Artefactos archivados en ~/${ARCHIVE_NAME}"

# ── 4. Limpiar variables de entorno de la sesión actual ───────────────────────
unset WL_ADMIN_URL WL_ADMIN_USER WL_ADMIN_PASS WL_API_BASE WL_HEAP_WARN_PCT
echo "  ✅ Variables de entorno del laboratorio eliminadas."

echo ""
echo "Limpieza completada. El entorno está listo para el siguiente laboratorio."
```

---

## 10. Resumen

### Puntos Clave

En este laboratorio has aprendido a:

1. **Navegar la REST API de WebLogic 15** accediendo a la documentación integrada en `/management/weblogic/latest` e identificando los endpoints principales: `domainConfig`, `domainRuntime`, `serverRuntime` y `edit`.

2. **Realizar operaciones CRUD completas** con `curl`: GET para consultar estados y configuraciones, POST para iniciar sesiones de edición y crear recursos, PATCH para actualizar propiedades, y DELETE para eliminar recursos; siempre siguiendo el flujo `startEdit → cambios → save → activate`, análogo al `edit()/startEdit()/save()/activate()` de WLST.

3. **Desarrollar un mini-framework Python 3** (`wl_admin.py`) con la librería `requests` que encapsula las operaciones REST en funciones reutilizables, aplica el principio de responsabilidad única y gestiona errores con cancelación automática de sesiones de edición.

4. **Implementar autenticación segura** leyendo credenciales exclusivamente desde variables de entorno, nunca desde código fuente, y aplicando la cabecera `X-Requested-By` requerida por WebLogic para todas las peticiones de escritura.

5. **Construir un script de monitoreo** (`wl_health_monitor.py`) que consulta el estado de servidores, despliegues y métricas JVM, genera informes con alertas visuales y devuelve códigos de salida adecuados para integración con sistemas de monitoreo externos (Nagios, Prometheus, CI/CD pipelines).

### Relación con el contenido de la lección 4.1

| Concepto WLST (Lección 4.1)          | Equivalente REST API (Este laboratorio)                        |
|---------------------------------------|----------------------------------------------------------------|
| `connect(user, pass, url)`            | HTTPBasicAuth en cada petición                                 |
| `edit() / startEdit()`                | POST `/edit/changeManager/startEdit`                           |
| `save()`                              | POST `/edit/changeManager/save`                                |
| `activate(block='true')`             | POST `/edit/changeManager/activate` con `{"block": true}`      |
| `undoEdit()`                          | POST `/edit/changeManager/cancelEdit`                          |
| `cd('/JDBCSystemResources')` + `set`  | PATCH `/edit/JDBCSystemResources/{name}/JDBCResource/{name}`   |
| `serverRuntime() / JVMRuntime`        | GET `/serverRuntime/JVMRuntime`                                |
| `domainRuntime() / ServerLifeCycle`   | GET `/domainRuntime/serverLifeCycleRuntimes`                   |

### Recursos Adicionales

- [Oracle WebLogic REST Management API Reference (14.1.1.0)](https://docs.oracle.com/en/middleware/standalone/weblogic-server/14.1.1.0/wlrer/index.html)
- [Python `requests` library — Documentación oficial](https://requests.readthedocs.io/en/latest/)
- [curl — Manual oficial](https://curl.se/docs/manpage.html)
- [HTTP Status Codes — MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)
- [OWASP — Credential Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Credential_Storage_Cheat_Sheet.html)

---
*Lab 04-00-02 — WebLogic 15 Administration Course | Nivel: Crear (Bloom) | Duración: 75 minutos*
