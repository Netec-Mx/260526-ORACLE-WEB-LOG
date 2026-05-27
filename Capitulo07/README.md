# Desplegar cluster simple y validar failover.

## 1. Metadatos

| Campo            | Valor                                      |
|------------------|--------------------------------------------|
| **Duración**     | 120 minutos                                |
| **Complejidad**  | Alta                                       |
| **Nivel Bloom**  | Crear (Create)                             |
| **Módulo**       | 7 — Alta Disponibilidad y Clustering       |
| **Laboratorio**  | 07-00-01                                   |

---

## 2. Descripción General

En este laboratorio construirás la primera arquitectura de alta disponibilidad del curso. Partiendo de un dominio nuevo, crearás un **cluster estático** denominado `LabCluster` con dos Managed Servers (`ms1` en puerto 7003 y `ms2` en puerto 7004) ejecutándose en el mismo host. Configurarás la **replicación de sesiones HTTP en memoria** (*in-memory session replication*), desplegarás la aplicación de muestra Jakarta EE 10 en el cluster y validarás el failover mediante Apache JMeter y `curl`. Al finalizar, habrás experimentado de primera mano cómo WebLogic preserva el estado de sesión del usuario cuando un nodo cae abruptamente, y comprenderás las diferencias prácticas respecto a la persistencia JDBC de sesiones.

---

## 3. Objetivos de Aprendizaje

- [ ] Diseñar y crear un cluster WebLogic estático con dos Managed Servers en el mismo host usando puertos diferenciados (7003 y 7004).
- [ ] Configurar la replicación de sesiones HTTP en memoria mediante `weblogic.xml` y los descriptores de despliegue de la aplicación.
- [ ] Desplegar la aplicación de muestra Jakarta EE 10 en el cluster y verificar el balanceo de carga a través de nginx.
- [ ] Simular la caída abrupta de `ms1` con `kill -9` y validar que `ms2` continúa el servicio con la sesión preservada.
- [ ] Comparar el comportamiento de *in-memory replication* frente a persistencia JDBC de sesiones, identificando ventajas y limitaciones de cada enfoque.

---

## 4. Prerrequisitos

### Conocimiento previo

| Área                                  | Nivel requerido                                                    |
|---------------------------------------|--------------------------------------------------------------------|
| Creación de dominios WebLogic         | Completado Lab 03-00-01 (WDT o Configuration Wizard)              |
| Gestión de recursos con WRC           | Completado Lab 02-00-01                                            |
| HTTP Sessions (JSESSIONID, cookies)   | Conocimiento básico: session affinity, session state              |
| Alta disponibilidad                   | Conceptos: failover, load balancing, session persistence           |
| Arquitecturas de cluster WLS          | Lección 7.1 completada: cluster estático vs dinámico               |

### Acceso y herramientas

| Herramienta / Recurso                      | Estado requerido                                                  |
|--------------------------------------------|-------------------------------------------------------------------|
| Oracle WebLogic Server 15 instalado        | En `/u01/oracle/wls`                                              |
| JDK 21 configurado (`JAVA_HOME`)           | `java -version` devuelve `21.x`                                   |
| Apache JMeter 5.6.x                        | Instalado y ejecutable                                             |
| nginx                                      | Instalado: `dnf install nginx -y` (o ya disponible)               |
| Aplicación de muestra Jakarta EE 10        | Archivo `sample-app.war` proporcionado por el instructor           |
| Usuario del sistema operativo              | `oracle` con permisos de escritura en `/u01/`                     |
| Puertos libres                             | 7001, 7003, 7004, 8080 sin ocupar                                 |

---

## 5. Entorno de Laboratorio

### Topología objetivo

```
┌─────────────────────────────────────────────────────────────┐
│                        HOST ÚNICO                           │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  nginx (puerto 8080) — Load Balancer / Proxy HTTP    │  │
│  │  Proxy Pass → ms1:7003 / ms2:7004 (round-robin)     │  │
│  └──────────────────────┬───────────────────────────────┘  │
│                         │                                   │
│         ┌───────────────┴────────────────┐                 │
│         ▼                                ▼                 │
│  ┌─────────────┐                ┌─────────────┐           │
│  │     ms1     │◄──── In-Memory ────►│     ms2     │           │
│  │  port 7003  │   Replication  │  port 7004  │           │
│  └─────────────┘                └─────────────┘           │
│         │                                │                  │
│         └──────────────┬─────────────────┘                 │
│                        ▼                                    │
│              ┌──────────────────┐                          │
│              │ Administration   │                          │
│              │ Server (port 7001│                          │
│              └──────────────────┘                          │
│                                                             │
│  Dominio: /u01/domains/cluster-domain                      │
└─────────────────────────────────────────────────────────────┘
```

### Variables de entorno comunes

Añade estas variables a tu sesión de terminal antes de comenzar:

```bash
export ORACLE_HOME=/u01/oracle/wls
export JAVA_HOME=/u01/jdk21
export WLS_HOME=${ORACLE_HOME}/wlserver
export DOMAIN_HOME=/u01/domains/cluster-domain
export PATH=${JAVA_HOME}/bin:${ORACLE_HOME}/oracle_common/common/bin:${PATH}
```

### Verificación rápida del entorno

```bash
# Verificar Java 21
java -version

# Verificar que WDT está disponible
ls /u01/wdt/weblogic-deploy/bin/createDomain.sh

# Verificar nginx
nginx -v

# Verificar puertos libres
ss -tlnp | grep -E '7001|7003|7004|8080'
# La salida debe estar vacía (ningún proceso escuchando aún)
```

---

## 6. Pasos del Laboratorio

---

### Paso 1: Preparar los artefactos del dominio

**Objetivo:** Crear la estructura de directorios y los archivos de modelo WDT necesarios para definir el cluster estático `LabCluster` con sus dos Managed Servers.

#### Instrucciones

**1.1.** Crea el directorio de trabajo del laboratorio:

```bash
mkdir -p /u01/lab07/model
mkdir -p /u01/lab07/apps
cd /u01/lab07
```

**1.2.** Copia la aplicación de muestra al directorio de apps:

```bash
cp /ruta/instructor/sample-app.war /u01/lab07/apps/
# Verifica que el WAR existe
ls -lh /u01/lab07/apps/sample-app.war
```

**1.3.** Crea el archivo de modelo WDT para el dominio con cluster estático. Este modelo refleja directamente los conceptos de la lección 7.1 (cluster estático, servidores explícitos, Machine):

```bash
cat > /u01/lab07/model/cluster-domain.yaml << 'EOF'
domainInfo:
  AdminUserName: '@@PROP:admin.username@@'
  AdminPassword: '@@PROP:admin.password@@'

topology:
  Name: cluster-domain
  AdminServerName: AdminServer

  Cluster:
    LabCluster:
      ClusterMessagingMode: unicast
      ClusterBroadcastChannel: ''

  Machine:
    LocalMachine:
      NodeManager:
        NMType: Plain
        ListenAddress: localhost
        ListenPort: 5556

  Server:
    AdminServer:
      ListenAddress: localhost
      ListenPort: 7001
      Machine: LocalMachine

    ms1:
      ListenAddress: localhost
      ListenPort: 7003
      Cluster: LabCluster
      Machine: LocalMachine
      ServerStart:
        Arguments: '-Xms512m -Xmx512m -XX:+UseG1GC'

    ms2:
      ListenAddress: localhost
      ListenPort: 7004
      Cluster: LabCluster
      Machine: LocalMachine
      ServerStart:
        Arguments: '-Xms512m -Xmx512m -XX:+UseG1GC'

appDeployments:
  Application:
    sample-app:
      SourcePath: /u01/lab07/apps/sample-app.war
      ModuleType: war
      Target: LabCluster
EOF
```

> **Nota sobre el modelo:** Observa que `ms1` y `ms2` son servidores **explícitos** (cluster estático), tal como describe la lección 7.1. Si usáramos `DynamicServers` con `ServerNamePrefix`, estaríamos ante un cluster dinámico. Aquí optamos por el enfoque estático para tener control individual de puertos (7003 y 7004) y argumentos JVM.

**1.4.** Crea el archivo de variables (nunca incluyas contraseñas directamente en el modelo):

```bash
cat > /u01/lab07/model/variables.properties << 'EOF'
admin.username=weblogic
admin.password=Welcome1
EOF
chmod 600 /u01/lab07/model/variables.properties
```

#### Salida esperada

```
/u01/lab07/
├── apps/
│   └── sample-app.war
└── model/
    ├── cluster-domain.yaml
    └── variables.properties
```

#### Verificación

```bash
# El modelo debe ser YAML válido
python3 -c "import yaml; yaml.safe_load(open('/u01/lab07/model/cluster-domain.yaml'))" && echo "YAML válido"
```

---

### Paso 2: Crear el dominio con WDT

**Objetivo:** Usar WebLogic Deploy Tooling para crear el dominio `cluster-domain` con el cluster estático `LabCluster`.

#### Instrucciones

**2.1.** Ejecuta `createDomain.sh` con el modelo preparado:

```bash
/u01/wdt/weblogic-deploy/bin/createDomain.sh \
  -oracle_home ${ORACLE_HOME} \
  -domain_type WLS \
  -domain_home ${DOMAIN_HOME} \
  -model_file /u01/lab07/model/cluster-domain.yaml \
  -variable_file /u01/lab07/model/variables.properties
```

**2.2.** Verifica la estructura del dominio creado:

```bash
ls -la ${DOMAIN_HOME}/
ls -la ${DOMAIN_HOME}/config/
# Examina el config.xml para confirmar la existencia del cluster y los servidores
grep -A5 'cluster-name\|<name>ms1\|<name>ms2\|LabCluster' ${DOMAIN_HOME}/config/config.xml
```

**2.3.** Examina la sección de servidores en `config.xml` para confirmar que ms1 y ms2 están definidos explícitamente (cluster estático):

```bash
python3 -c "
import xml.etree.ElementTree as ET
tree = ET.parse('${DOMAIN_HOME}/config/config.xml')
root = tree.getroot()
ns = {'wls': 'http://xmlns.oracle.com/weblogic/domain'}
for s in root.findall('.//wls:server', ns):
    name = s.find('wls:name', ns)
    port = s.find('wls:listen-port', ns)
    cluster = s.find('wls:cluster', ns)
    print(f'Server: {name.text if name is not None else \"?\"}, Port: {port.text if port is not None else \"?\"}, Cluster: {cluster.text if cluster is not None else \"N/A\"}')
"
```

#### Salida esperada

```
####<createDomain> <INFO> Domain cluster-domain successfully created at /u01/domains/cluster-domain
```

```
Server: AdminServer,  Port: 7001, Cluster: N/A
Server: ms1,          Port: 7003, Cluster: LabCluster
Server: ms2,          Port: 7004, Cluster: LabCluster
```

#### Verificación

```bash
test -f ${DOMAIN_HOME}/config/config.xml && echo "config.xml existe: OK"
grep -q 'LabCluster' ${DOMAIN_HOME}/config/config.xml && echo "LabCluster encontrado: OK"
grep -q 'ms1' ${DOMAIN_HOME}/config/config.xml && echo "ms1 encontrado: OK"
grep -q 'ms2' ${DOMAIN_HOME}/config/config.xml && echo "ms2 encontrado: OK"
```

---

### Paso 3: Preparar el descriptor de despliegue para replicación de sesiones

**Objetivo:** Configurar `weblogic.xml` en la aplicación de muestra para activar la replicación de sesiones HTTP en memoria entre los nodos del cluster.

#### Instrucciones

**3.1.** Descomprime el WAR para inspeccionar y modificar su contenido:

```bash
mkdir -p /u01/lab07/app-work
cd /u01/lab07/app-work
jar -xf /u01/lab07/apps/sample-app.war
ls -la WEB-INF/
```

**3.2.** Crea o modifica el archivo `WEB-INF/weblogic.xml` para activar la replicación en memoria. Este es el descriptor clave que le indica a WebLogic cómo gestionar las sesiones HTTP en un entorno de cluster:

```bash
cat > /u01/lab07/app-work/WEB-INF/weblogic.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<weblogic-web-app
  xmlns="http://xmlns.oracle.com/weblogic/weblogic-web-app"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://xmlns.oracle.com/weblogic/weblogic-web-app
    http://xmlns.oracle.com/weblogic/weblogic-web-app/2.0/weblogic-web-app.xsd">

  <session-descriptor>
    <!--
      PersistentStoreType: replicated_if_clustered
      Activa replicación en memoria cuando el servidor es miembro de un cluster.
      Si el servidor no está en cluster, usa almacenamiento en memoria local (sin replicación).
      Alternativa: 'replicated' (fuerza replicación, falla si no hay cluster)
      Alternativa JDBC: 'jdbc' (requiere DataSource configurado, ver Paso 7)
    -->
    <persistent-store-type>replicated_if_clustered</persistent-store-type>

    <!-- Número máximo de sesiones activas en memoria por servidor -->
    <max-in-memory-sessions>1000</max-in-memory-sessions>

    <!-- Tiempo de invalidación de sesión (segundos) -->
    <invalidation-interval-secs>60</invalidation-interval-secs>

    <!-- Tiempo de timeout de sesión inactiva (segundos) -->
    <timeout-secs>1800</timeout-secs>

    <!--
      cookie-name: nombre de la cookie de sesión
      El proxy (nginx) usará este nombre para implementar session affinity
    -->
    <cookie-name>JSESSIONID</cookie-name>
    <cookie-http-only>true</cookie-http-only>
  </session-descriptor>

  <context-root>/sample-app</context-root>

</weblogic-web-app>
EOF
```

**3.3.** Verifica también el `web.xml` para confirmar que la aplicación usa sesiones HTTP con estado (distributable):

```bash
# Si web.xml existe, verificar el elemento <distributable/>
if grep -q '<distributable' /u01/lab07/app-work/WEB-INF/web.xml 2>/dev/null; then
    echo "<distributable/> ya presente: OK"
else
    echo "ADVERTENCIA: <distributable/> no encontrado en web.xml"
    echo "Añadiendo <distributable/> al web.xml..."
    # Inserta <distributable/> justo después de <web-app ...>
    sed -i 's|<web-app[^>]*>|&\n  <distributable/>|' /u01/lab07/app-work/WEB-INF/web.xml
fi
```

> **Importante:** El elemento `<distributable/>` en `web.xml` es **obligatorio** para que WebLogic replique las sesiones entre nodos. Sin él, la sesión es local al servidor y no sobrevive a un failover.

**3.4.** Reempaqueta el WAR con los cambios:

```bash
cd /u01/lab07/app-work
jar -cf /u01/lab07/apps/sample-app-clustered.war .
ls -lh /u01/lab07/apps/sample-app-clustered.war
```

#### Salida esperada

```
-rw-r--r-- 1 oracle oracle 2.1M Jun 10 10:15 /u01/lab07/apps/sample-app-clustered.war
```

#### Verificación

```bash
# Confirmar que weblogic.xml está en el WAR
jar -tf /u01/lab07/apps/sample-app-clustered.war | grep weblogic.xml
# Debe mostrar: WEB-INF/weblogic.xml

# Confirmar persistent-store-type
jar -xf /u01/lab07/apps/sample-app-clustered.war WEB-INF/weblogic.xml -C /tmp/check 2>/dev/null || \
  unzip -p /u01/lab07/apps/sample-app-clustered.war WEB-INF/weblogic.xml | grep persistent-store-type
```

---

### Paso 4: Iniciar el Administration Server y los Managed Servers

**Objetivo:** Arrancar el dominio completo (AdminServer + ms1 + ms2) y verificar que el cluster `LabCluster` está activo.

#### Instrucciones

**4.1.** Inicia el Administration Server en segundo plano:

```bash
cd ${DOMAIN_HOME}
nohup ./startWebLogic.sh > /u01/lab07/logs/admin-server.log 2>&1 &
echo "AdminServer PID: $!"
```

**4.2.** Espera a que el Administration Server esté completamente levantado (puede tardar 60-90 segundos):

```bash
# Espera activa con timeout de 120 segundos
TIMEOUT=120
ELAPSED=0
echo "Esperando AdminServer en puerto 7001..."
while ! curl -s -o /dev/null -w "%{http_code}" http://localhost:7001/console | grep -q "302\|200"; do
    sleep 5
    ELAPSED=$((ELAPSED+5))
    echo "  ${ELAPSED}s transcurridos..."
    if [ $ELAPSED -ge $TIMEOUT ]; then
        echo "ERROR: Timeout esperando AdminServer"
        tail -20 /u01/lab07/logs/admin-server.log
        exit 1
    fi
done
echo "AdminServer activo."
```

**4.3.** Inicia `ms1` usando el script del dominio:

```bash
mkdir -p /u01/lab07/logs
nohup ${DOMAIN_HOME}/bin/startManagedWebLogic.sh ms1 http://localhost:7001 \
  > /u01/lab07/logs/ms1.log 2>&1 &
echo "ms1 PID: $!"
```

**4.4.** Inicia `ms2`:

```bash
nohup ${DOMAIN_HOME}/bin/startManagedWebLogic.sh ms2 http://localhost:7001 \
  > /u01/lab07/logs/ms2.log 2>&1 &
echo "ms2 PID: $!"
```

**4.5.** Espera a que ambos Managed Servers estén en estado `RUNNING` (puede tardar 60-90 segundos cada uno):

```bash
# Función de espera para Managed Server
wait_for_ms() {
    local MS_NAME=$1
    local MS_PORT=$2
    local TIMEOUT=120
    local ELAPSED=0
    echo "Esperando ${MS_NAME} en puerto ${MS_PORT}..."
    while ! curl -s -u weblogic:Welcome1 \
        "http://localhost:7001/management/weblogic/latest/domainRuntime/serverLifeCycleRuntimes/${MS_NAME}" \
        | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('state',''))" 2>/dev/null \
        | grep -q "RUNNING"; do
        sleep 5
        ELAPSED=$((ELAPSED+5))
        echo "  ${MS_NAME}: ${ELAPSED}s..."
        if [ $ELAPSED -ge $TIMEOUT ]; then
            echo "TIMEOUT esperando ${MS_NAME}"
            return 1
        fi
    done
    echo "${MS_NAME}: RUNNING ✓"
}

wait_for_ms ms1 7003
wait_for_ms ms2 7004
```

**4.6.** Verifica el estado del cluster mediante la REST API:

```bash
curl -s -u weblogic:Welcome1 \
  "http://localhost:7001/management/weblogic/latest/domainRuntime/serverLifeCycleRuntimes" \
  | python3 -m json.tool | grep -E '"name"|"state"'
```

#### Salida esperada

```
AdminServer: RUNNING ✓
ms1: RUNNING ✓
ms2: RUNNING ✓
```

```json
"name": "AdminServer",
"state": "RUNNING",
"name": "ms1",
"state": "RUNNING",
"name": "ms2",
"state": "RUNNING",
```

#### Verificación

```bash
# Verificar que ms1 y ms2 son miembros del cluster LabCluster
curl -s -u weblogic:Welcome1 \
  "http://localhost:7001/management/weblogic/latest/domainRuntime/clusterRuntimes/LabCluster" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print('Cluster:', d.get('name'), '| Servers:', d.get('serverNames','?'))"
```

---

### Paso 5: Desplegar la aplicación en el cluster

**Objetivo:** Desplegar `sample-app-clustered.war` en `LabCluster` y verificar que es accesible desde ambos nodos.

#### Instrucciones

**5.1.** Despliega la aplicación usando la REST API de WebLogic (sin consola gráfica):

```bash
# Paso 1: Iniciar edición
curl -s -u weblogic:Welcome1 \
  -H "Content-Type: application/json" \
  -X POST \
  "http://localhost:7001/management/weblogic/latest/edit/changeManager/startEdit"

echo ""

# Paso 2: Subir y desplegar el WAR en LabCluster
curl -s -u weblogic:Welcome1 \
  -H "Prefer: respond-async" \
  -F "model={\"name\":\"sample-app\",\"targets\":[{\"identity\":[\"clusters\",\"LabCluster\"]}]}" \
  -F "sourcePath=@/u01/lab07/apps/sample-app-clustered.war" \
  -X POST \
  "http://localhost:7001/management/weblogic/latest/edit/appDeployments"

echo ""

# Paso 3: Activar cambios
curl -s -u weblogic:Welcome1 \
  -H "Content-Type: application/json" \
  -X POST \
  "http://localhost:7001/management/weblogic/latest/edit/changeManager/activate"
```

**5.2.** Alternativamente, si prefieres usar WLST para el despliegue:

```bash
${WLS_HOME}/common/bin/wlst.sh << 'WLST_EOF'
connect('weblogic', 'Welcome1', 't3://localhost:7001')
deploy('sample-app',
       '/u01/lab07/apps/sample-app-clustered.war',
       targets='LabCluster',
       stageMode='stage')
disconnect()
exit()
WLST_EOF
```

**5.3.** Verifica que la aplicación está activa en ambos nodos:

```bash
# Acceso directo a ms1
curl -s -o /dev/null -w "ms1 (7003): HTTP %{http_code}\n" \
  http://localhost:7003/sample-app/

# Acceso directo a ms2
curl -s -o /dev/null -w "ms2 (7004): HTTP %{http_code}\n" \
  http://localhost:7004/sample-app/
```

#### Salida esperada

```
ms1 (7003): HTTP 200
ms2 (7004): HTTP 200
```

#### Verificación

```bash
# Verificar estado del despliegue via REST
curl -s -u weblogic:Welcome1 \
  "http://localhost:7001/management/weblogic/latest/domainRuntime/deploymentManager/appDeploymentRuntimes/sample-app" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print('App:', d.get('name'), '| State:', d.get('state','?'))"
```

---

### Paso 6: Configurar nginx como proxy HTTP con balanceo de carga

**Objetivo:** Configurar nginx para distribuir el tráfico entre `ms1` y `ms2`, implementando *session affinity* basada en cookie `JSESSIONID`.

#### Instrucciones

**6.1.** Crea el archivo de configuración de nginx para el cluster:

```bash
sudo tee /etc/nginx/conf.d/weblogic-cluster.conf << 'EOF'
# Upstream: pool de servidores del cluster LabCluster
upstream labcluster {
    # ip_hash garantiza que un cliente siempre va al mismo servidor
    # Esto implementa session affinity básica por IP
    # Para affinity por cookie JSESSIONID, se necesita nginx Plus o sticky module
    ip_hash;

    server localhost:7003 weight=1 max_fails=3 fail_timeout=30s;
    server localhost:7004 weight=1 max_fails=3 fail_timeout=30s;

    keepalive 32;
}

server {
    listen 8080;
    server_name localhost;

    # Logs específicos para este lab
    access_log /var/log/nginx/weblogic-cluster-access.log;
    error_log  /var/log/nginx/weblogic-cluster-error.log;

    location /sample-app/ {
        proxy_pass         http://labcluster;
        proxy_http_version 1.1;
        proxy_set_header   Connection        "";
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;

        # Timeouts
        proxy_connect_timeout 10s;
        proxy_send_timeout    60s;
        proxy_read_timeout    60s;

        # Buffer
        proxy_buffering    on;
        proxy_buffer_size  4k;
        proxy_buffers      8 4k;
    }

    # Health check endpoint
    location /nginx-health {
        access_log off;
        return 200 "nginx OK\n";
        add_header Content-Type text/plain;
    }
}
EOF
```

**6.2.** Verifica la configuración de nginx y reinícialo:

```bash
sudo nginx -t
sudo systemctl restart nginx
sudo systemctl status nginx --no-pager
```

**6.3.** Abre el puerto 8080 en el firewall (si está activo):

```bash
sudo firewall-cmd --add-port=8080/tcp --permanent 2>/dev/null || true
sudo firewall-cmd --reload 2>/dev/null || true
```

**6.4.** Verifica el acceso a través del proxy:

```bash
# Primera petición
curl -s -o /dev/null -w "nginx→cluster: HTTP %{http_code}\n" \
  http://localhost:8080/sample-app/

# Petición con captura de cookie de sesión
curl -s -c /tmp/session-cookies.txt \
  -o /tmp/response1.html \
  -w "HTTP %{http_code} | Bytes: %{size_download}\n" \
  http://localhost:8080/sample-app/

# Mostrar la cookie JSESSIONID recibida
grep JSESSIONID /tmp/session-cookies.txt || echo "ADVERTENCIA: No se recibió JSESSIONID"
```

#### Salida esperada

```
nginx syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
nginx→cluster: HTTP 200
HTTP 200 | Bytes: 3842
```

```
localhost	FALSE	/	FALSE	0	JSESSIONID	abc123...!ms1!...
```

> **Nota sobre el JSESSIONID:** WebLogic codifica el servidor de origen en el JSESSIONID con el formato `session_id!server_id`. Esto permite al proxy saber a qué servidor enrutar la petición para mantener la afinidad. En la configuración con `ip_hash` de nginx, la afinidad se mantiene por IP del cliente.

#### Verificación

```bash
curl -f http://localhost:8080/nginx-health && echo "nginx proxy: OK"
```

---

### Paso 7: Ejecutar prueba de carga con JMeter y verificar distribución

**Objetivo:** Crear y ejecutar un plan de prueba JMeter que mantenga sesiones activas y confirme que el tráfico se distribuye entre ambos nodos.

#### Instrucciones

**7.1.** Crea un plan de prueba JMeter en línea de comandos (JMX):

```bash
cat > /u01/lab07/jmeter-cluster-test.jmx << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<jmeterTestPlan version="1.2" properties="5.0" jmeter="5.6.3">
  <hashTree>
    <TestPlan guiclass="TestPlanGui" testclass="TestPlan" testname="LabCluster Session Test">
      <boolProp name="TestPlan.serialize_threadgroups">false</boolProp>
    </TestPlan>
    <hashTree>
      <!-- Gestor de cookies: mantiene JSESSIONID entre peticiones -->
      <CookieManager guiclass="CookiePanel" testclass="CookieManager"
                     testname="HTTP Cookie Manager" enabled="true">
        <boolProp name="CookieManager.clearEachIteration">false</boolProp>
        <stringProp name="CookieManager.implementation">
          org.apache.jmeter.protocol.http.control.HC4CookieHandler
        </stringProp>
      </CookieManager>
      <hashTree/>

      <!-- Grupo de hilos: 10 usuarios, 60 segundos de rampa, 5 iteraciones -->
      <ThreadGroup guiclass="ThreadGroupGui" testclass="ThreadGroup"
                   testname="Cluster Users" enabled="true">
        <stringProp name="ThreadGroup.num_threads">10</stringProp>
        <stringProp name="ThreadGroup.ramp_time">30</stringProp>
        <intProp name="ThreadGroup.num_threads">10</intProp>
        <boolProp name="ThreadGroup.same_user_on_next_iteration">true</boolProp>
        <stringProp name="ThreadGroup.duration">60</stringProp>
        <boolProp name="ThreadGroup.scheduler">true</boolProp>
        <elementProp name="ThreadGroup.main_controller" elementType="LoopController">
          <boolProp name="LoopController.continue_forever">true</boolProp>
          <intProp name="LoopController.loops">-1</intProp>
        </elementProp>
      </ThreadGroup>
      <hashTree>
        <!-- Petición HTTP al cluster a través de nginx -->
        <HTTPSamplerProxy guiclass="HttpTestSampleGui" testclass="HTTPSamplerProxy"
                          testname="GET /sample-app/" enabled="true">
          <stringProp name="HTTPSampler.domain">localhost</stringProp>
          <stringProp name="HTTPSampler.port">8080</stringProp>
          <stringProp name="HTTPSampler.protocol">http</stringProp>
          <stringProp name="HTTPSampler.path">/sample-app/</stringProp>
          <stringProp name="HTTPSampler.method">GET</stringProp>
          <boolProp name="HTTPSampler.follow_redirects">true</boolProp>
          <boolProp name="HTTPSampler.use_keepalive">true</boolProp>
        </HTTPSamplerProxy>
        <hashTree/>

        <!-- Extractor de respuesta para capturar el servidor que respondió -->
        <RegexExtractor guiclass="RegexExtractorGui" testclass="RegexExtractor"
                        testname="Extract Server Name" enabled="true">
          <stringProp name="RegexExtractor.useHeaders">true</stringProp>
          <stringProp name="RegexExtractor.refname">server_responded</stringProp>
          <stringProp name="RegexExtractor.regex">X-Powered-By: (.*)</stringProp>
          <stringProp name="RegexExtractor.template">$1$</stringProp>
          <stringProp name="RegexExtractor.default">UNKNOWN</stringProp>
        </RegexExtractor>
        <hashTree/>

        <!-- Pausa entre peticiones: 1-3 segundos -->
        <UniformRandomTimer guiclass="UniformRandomTimerGui"
                            testclass="UniformRandomTimer"
                            testname="Think Time" enabled="true">
          <stringProp name="ConstantTimer.delay">1000</stringProp>
          <stringProp name="RandomTimer.range">2000</stringProp>
        </UniformRandomTimer>
        <hashTree/>
      </hashTree>

      <!-- Listener: guardar resultados en CSV -->
      <ResultCollector guiclass="SimpleDataWriter" testclass="ResultCollector"
                       testname="Results CSV" enabled="true">
        <boolProp name="ResultCollector.error_logging">false</boolProp>
        <objProp>
          <name>saveConfig</name>
          <value class="SampleSaveConfiguration">
            <time>true</time>
            <latency>true</latency>
            <responseCode>true</responseCode>
            <responseMessage>true</responseMessage>
            <threadName>true</threadName>
            <dataType>true</dataType>
            <encoding>false</encoding>
            <assertions>true</assertions>
            <subresults>true</subresults>
            <responseData>false</responseData>
            <samplerData>false</samplerData>
            <xml>false</xml>
            <fieldNames>true</fieldNames>
            <responseHeaders>true</responseHeaders>
            <requestHeaders>false</requestHeaders>
            <responseDataOnError>false</responseDataOnError>
            <saveAssertionResultsFailureMessage>true</saveAssertionResultsFailureMessage>
            <bytes>true</bytes>
            <url>true</url>
            <threadCounts>true</threadCounts>
            <idleTime>true</idleTime>
            <connectTime>true</connectTime>
          </value>
        </objProp>
        <stringProp name="filename">/u01/lab07/jmeter-results.csv</stringProp>
      </ResultCollector>
      <hashTree/>
    </hashTree>
  </hashTree>
</jmeterTestPlan>
EOF
```

**7.2.** Ejecuta JMeter en modo no-GUI durante 60 segundos:

```bash
/u01/jmeter/bin/jmeter -n \
  -t /u01/lab07/jmeter-cluster-test.jmx \
  -l /u01/lab07/jmeter-results.csv \
  -e -o /u01/lab07/jmeter-report \
  2>&1 | tee /u01/lab07/logs/jmeter-run1.log
```

**7.3.** Mientras JMeter corre, en otra terminal monitoriza los logs de acceso de nginx para confirmar la distribución:

```bash
# En terminal separada: observar a qué servidor va cada petición
# (el JSESSIONID codifica el servidor de origen)
tail -f /var/log/nginx/weblogic-cluster-access.log | \
  grep -oP 'JSESSIONID=[^!]+![^!]+' | sort | uniq -c
```

**7.4.** Verifica el resumen de resultados de JMeter:

```bash
# Analizar el CSV de resultados
python3 << 'PYEOF'
import csv
total = 0
errors = 0
with open('/u01/lab07/jmeter-results.csv', 'r') as f:
    reader = csv.DictReader(f)
    for row in reader:
        total += 1
        if row.get('responseCode', '200') != '200':
            errors += 1
print(f"Total peticiones: {total}")
print(f"Peticiones exitosas: {total - errors}")
print(f"Errores: {errors}")
print(f"Tasa de éxito: {((total-errors)/total*100):.1f}%" if total > 0 else "Sin datos")
PYEOF
```

#### Salida esperada

```
summary = 600 in 00:01:00 = 10.0/s  Avg: 95  Min: 23  Max: 450  Err: 0 (0.00%)
Total peticiones: 600
Peticiones exitosas: 600
Errores: 0
Tasa de éxito: 100.0%
```

#### Verificación

```bash
# Verificar que ambos servidores están respondiendo (distribución de carga)
echo "=== Distribución de carga por servidor ==="
grep -oP '!\w+!' /tmp/session-cookies.txt 2>/dev/null | sort | uniq -c || \
  echo "Verificar logs de nginx para distribución"
```

---

### Paso 8: Simular failover — caída abrupta de ms1

**Objetivo:** Detener `ms1` con `kill -9` mientras JMeter mantiene sesiones activas, y verificar que `ms2` continúa el servicio con el estado de sesión preservado gracias a la replicación en memoria.

#### Instrucciones

**8.1.** Antes de matar ms1, captura el estado de sesión actual para comparar después:

```bash
# Establece una sesión con la aplicación y guarda la cookie
curl -s -c /tmp/pre-failover-cookies.txt \
     -b /tmp/pre-failover-cookies.txt \
     -o /tmp/pre-failover-response.html \
     http://localhost:8080/sample-app/

echo "=== Estado PRE-FAILOVER ==="
echo "Cookie de sesión:"
cat /tmp/pre-failover-cookies.txt
echo ""
echo "Primeras líneas de respuesta:"
head -20 /tmp/pre-failover-response.html
```

**8.2.** Inicia JMeter en segundo plano para mantener carga durante el failover:

```bash
/u01/jmeter/bin/jmeter -n \
  -t /u01/lab07/jmeter-cluster-test.jmx \
  -l /u01/lab07/jmeter-failover-results.csv \
  2>&1 > /u01/lab07/logs/jmeter-failover.log &
JMETER_PID=$!
echo "JMeter corriendo con PID: ${JMETER_PID}"

# Espera 15 segundos para que JMeter establezca sesiones
sleep 15
```

**8.3.** Identifica el PID de ms1 y mátalo abruptamente con `kill -9`:

```bash
# Encuentra el PID del proceso ms1
MS1_PID=$(ps aux | grep '[w]eblogic.Name=ms1' | awk '{print $2}')
echo "PID de ms1: ${MS1_PID}"

# Verifica que ms1 está respondiendo antes del kill
curl -s -o /dev/null -w "ms1 ANTES del kill: HTTP %{http_code}\n" \
  http://localhost:7003/sample-app/

# KILL -9: simula fallo abrupto (no graceful shutdown)
echo ">>> Ejecutando kill -9 en ms1 (PID: ${MS1_PID})..."
kill -9 ${MS1_PID}
echo ">>> ms1 terminado a las $(date '+%H:%M:%S')"
```

**8.4.** Inmediatamente después del kill, verifica que el servicio continúa a través de nginx:

```bash
# Espera 2 segundos para que nginx detecte el fallo
sleep 2

echo "=== Verificación POST-FAILOVER ==="
for i in $(seq 1 10); do
    HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
      -b /tmp/pre-failover-cookies.txt \
      -c /tmp/pre-failover-cookies.txt \
      http://localhost:8080/sample-app/)
    echo "  Petición $i: HTTP ${HTTP_CODE} a las $(date '+%H:%M:%S')"
    sleep 1
done
```

**8.5.** Verifica que la sesión se preservó en ms2 (el contador de visitas debe continuar):

```bash
# Accede a la app con la MISMA cookie de sesión
curl -s \
     -b /tmp/pre-failover-cookies.txt \
     -c /tmp/pre-failover-cookies.txt \
     -o /tmp/post-failover-response.html \
     http://localhost:8080/sample-app/

echo "=== Comparación de sesión ==="
echo "--- Respuesta PRE-failover (contador de visitas) ---"
grep -i 'visit\|session\|counter\|count' /tmp/pre-failover-response.html | head -5

echo "--- Respuesta POST-failover (contador de visitas) ---"
grep -i 'visit\|session\|counter\|count' /tmp/post-failover-response.html | head -5

echo ""
echo "Si el contador continúa desde donde lo dejó (no reinició a 1),"
echo "la replicación de sesión en memoria funcionó correctamente."
```

**8.6.** Verifica el estado del cluster desde la REST API:

```bash
curl -s -u weblogic:Welcome1 \
  "http://localhost:7001/management/weblogic/latest/domainRuntime/serverLifeCycleRuntimes" \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
for item in data.get('items', []):
    print(f\"  {item.get('name','?'):20s} -> {item.get('state','?')}\")"
```

**8.7.** Detén JMeter y analiza los resultados del failover:

```bash
kill ${JMETER_PID} 2>/dev/null
sleep 2

# Analizar resultados con foco en el momento del failover
python3 << 'PYEOF'
import csv
from datetime import datetime

total = errors = 0
failover_errors = []

with open('/u01/lab07/jmeter-failover-results.csv', 'r') as f:
    reader = csv.DictReader(f)
    for row in reader:
        total += 1
        code = row.get('responseCode', '200')
        if code not in ('200', '302'):
            errors += 1
            failover_errors.append({
                'time': row.get('timeStamp', '?'),
                'code': code,
                'msg': row.get('responseMessage', '?')
            })

print(f"=== Análisis de Failover ===")
print(f"Total peticiones durante prueba: {total}")
print(f"Peticiones exitosas: {total - errors}")
print(f"Errores durante failover: {errors}")
print(f"Tasa de éxito: {((total-errors)/total*100):.1f}%" if total > 0 else "Sin datos")
if failover_errors:
    print(f"\nPrimeros errores detectados (momento del failover):")
    for e in failover_errors[:5]:
        print(f"  t={e['time']} | HTTP {e['code']} | {e['msg']}")
PYEOF
```

#### Salida esperada

```
>>> ms1 terminado a las 10:32:15

=== Verificación POST-FAILOVER ===
  Petición 1: HTTP 200 a las 10:32:17
  Petición 2: HTTP 200 a las 10:32:18
  ...
  Petición 10: HTTP 200 a las 10:32:26

=== Análisis de Failover ===
Total peticiones durante prueba: 450
Peticiones exitosas: 447
Errores durante failover: 3
Tasa de éxito: 99.3%
```

> **Nota:** Es normal observar 1-5 errores en el momento exacto del kill, correspondientes a peticiones en vuelo cuando ms1 cayó. La replicación en memoria garantiza que las **sesiones establecidas** sobreviven; las peticiones en tránsito en el momento del fallo pueden recibir un error de conexión que el cliente debe reintentar.

#### Verificación

```bash
# ms1 debe estar en FAILED o SHUTDOWN
curl -s -u weblogic:Welcome1 \
  "http://localhost:7001/management/weblogic/latest/domainRuntime/serverLifeCycleRuntimes/ms1" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print('ms1 state:', d.get('state','?'))"

# ms2 debe estar en RUNNING
curl -s -u weblogic:Welcome1 \
  "http://localhost:7001/management/weblogic/latest/domainRuntime/serverLifeCycleRuntimes/ms2" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print('ms2 state:', d.get('state','?'))"
```

---

### Paso 9: Comparación — Persistencia JDBC vs In-Memory Replication

**Objetivo:** Comprender las diferencias entre `replicated_if_clustered` (in-memory) y `jdbc` (JDBC persistence) como estrategias de replicación de sesiones, y configurar la opción JDBC de forma teórico-práctica.

#### Instrucciones

**9.1.** Examina las diferencias clave entre ambas estrategias:

```bash
cat << 'EOF'
=== COMPARATIVA: In-Memory Replication vs JDBC Session Persistence ===

┌─────────────────────────┬─────────────────────────┬─────────────────────────┐
│ Característica          │ In-Memory Replication   │ JDBC Persistence        │
├─────────────────────────┼─────────────────────────┼─────────────────────────┤
│ Configuración           │ weblogic.xml:            │ weblogic.xml:           │
│                         │ replicated_if_clustered  │ jdbc                    │
├─────────────────────────┼─────────────────────────┼─────────────────────────┤
│ Requisitos adicionales  │ Ninguno (solo cluster)  │ DataSource + tabla BD   │
├─────────────────────────┼─────────────────────────┼─────────────────────────┤
│ Rendimiento             │ Alto (memoria RAM)       │ Medio (I/O base datos)  │
├─────────────────────────┼─────────────────────────┼─────────────────────────┤
│ Escalabilidad           │ Limitada por RAM         │ Escalable (BD externa)  │
├─────────────────────────┼─────────────────────────┼─────────────────────────┤
│ Durabilidad             │ Se pierde si cae TODO    │ Persiste tras reinicio  │
│                         │ el cluster               │ completo del cluster    │
├─────────────────────────┼─────────────────────────┼─────────────────────────┤
│ Failover dentro cluster │ Transparente             │ Transparente            │
├─────────────────────────┼─────────────────────────┼─────────────────────────┤
│ Failover cluster total  │ Sesiones perdidas        │ Sesiones recuperables   │
├─────────────────────────┼─────────────────────────┼─────────────────────────┤
│ Caso de uso ideal       │ Sesiones cortas,         │ Sesiones largas,        │
│                         │ alta frecuencia,         │ datos críticos,         │
│                         │ baja latencia            │ recuperación completa   │
└─────────────────────────┴─────────────────────────┴─────────────────────────┘
EOF
```

**9.2.** Crea el descriptor `weblogic.xml` alternativo para persistencia JDBC (referencia, no se despliega en este lab):

```bash
cat > /u01/lab07/weblogic-jdbc-session.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<weblogic-web-app
  xmlns="http://xmlns.oracle.com/weblogic/weblogic-web-app"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">

  <session-descriptor>
    <!--
      JDBC Session Persistence:
      Requiere:
      1. DataSource 'SessionDS' configurado en el dominio
      2. Tabla WL_SERVLET_SESSIONS en la base de datos
         CREATE TABLE WL_SERVLET_SESSIONS (
           WL_ID             VARCHAR(100) NOT NULL PRIMARY KEY,
           WL_CONTEXT_PATH   VARCHAR(100) NOT NULL,
           WL_IS_NEW         CHAR(1),
           WL_CREATE_TIME    NUMERIC(20),
           WL_IS_VALID       CHAR(1),
           WL_SESSION_VALUES MEDIUMBLOB,
           WL_ACCESS_TIME    NUMERIC(20),
           WL_MAX_INACTIVE_INTERVAL INT
         );
    -->
    <persistent-store-type>jdbc</persistent-store-type>
    <persistent-store-pool>SessionDS</persistent-store-pool>
    <persistent-store-table>WL_SERVLET_SESSIONS</persistent-store-table>
    <cache-size>128</cache-size>
    <invalidation-interval-secs>60</invalidation-interval-secs>
    <timeout-secs>1800</timeout-secs>
    <cookie-name>JSESSIONID</cookie-name>
  </session-descriptor>

  <context-root>/sample-app</context-root>

</weblogic-web-app>
EOF

echo "Descriptor JDBC creado en: /u01/lab07/weblogic-jdbc-session.xml"
echo "NOTA: Este archivo es de referencia. Para usarlo en producción,"
echo "configura primero el DataSource 'SessionDS' en el dominio."
```

**9.3.** Documenta la decisión de diseño para este lab:

```bash
cat << 'EOF'
=== DECISIÓN DE DISEÑO PARA ESTE LABORATORIO ===

Usamos 'replicated_if_clustered' porque:
✓ El lab tiene 2 nodos en el mismo host (baja latencia de replicación)
✓ No hay base de datos configurada en el dominio de este lab
✓ Las sesiones son de corta duración (contador de visitas)
✓ El objetivo es validar failover rápido sin dependencias externas

En producción, considera JDBC si:
→ Las sesiones contienen datos de negocio críticos
→ Necesitas recuperar sesiones tras un reinicio completo del cluster
→ Tienes más de 4 nodos (la replicación in-memory crece O(n²))
→ La base de datos ya es parte de tu arquitectura HA

EOF
```

#### Verificación

```bash
# Confirmar que el WAR desplegado usa in-memory replication
unzip -p /u01/lab07/apps/sample-app-clustered.war WEB-INF/weblogic.xml | \
  grep persistent-store-type
# Debe mostrar: <persistent-store-type>replicated_if_clustered</persistent-store-type>
```

---

## 7. Validación y Pruebas Finales

### Checklist de validación completa

Ejecuta los siguientes comandos para confirmar que todos los objetivos del laboratorio se han cumplido:

```bash
echo "=========================================="
echo "  VALIDACIÓN FINAL - Lab 07-00-01"
echo "=========================================="

# 1. Cluster estático creado
echo -n "[1] Cluster LabCluster existe en config.xml: "
grep -q 'LabCluster' ${DOMAIN_HOME}/config/config.xml && echo "OK ✓" || echo "FALLO ✗"

# 2. ms1 y ms2 con puertos correctos
echo -n "[2] ms1 en puerto 7003: "
grep -A5 '<name>ms1</name>' ${DOMAIN_HOME}/config/config.xml | grep -q '7003' && echo "OK ✓" || echo "FALLO ✗"

echo -n "[3] ms2 en puerto 7004: "
grep -A5 '<name>ms2</name>' ${DOMAIN_HOME}/config/config.xml | grep -q '7004' && echo "OK ✓" || echo "FALLO ✗"

# 3. Replicación de sesiones configurada
echo -n "[4] In-memory replication en weblogic.xml: "
unzip -p /u01/lab07/apps/sample-app-clustered.war WEB-INF/weblogic.xml 2>/dev/null | \
  grep -q 'replicated_if_clustered' && echo "OK ✓" || echo "FALLO ✗"

# 4. nginx proxy activo
echo -n "[5] nginx escuchando en puerto 8080: "
ss -tlnp | grep -q ':8080' && echo "OK ✓" || echo "FALLO ✗"

# 5. AdminServer running
echo -n "[6] AdminServer en estado RUNNING: "
curl -s -u weblogic:Welcome1 \
  "http://localhost:7001/management/weblogic/latest/domainRuntime/serverLifeCycleRuntimes/AdminServer" \
  2>/dev/null | python3 -c "import sys,json; d=json.load(sys.stdin); print('OK ✓' if d.get('state')=='RUNNING' else 'FALLO ✗')" 2>/dev/null || echo "FALLO ✗"

# 6. ms2 running (ms1 fue matado en el paso 8)
echo -n "[7] ms2 en estado RUNNING tras failover: "
curl -s -u weblogic:Welcome1 \
  "http://localhost:7001/management/weblogic/latest/domainRuntime/serverLifeCycleRuntimes/ms2" \
  2>/dev/null | python3 -c "import sys,json; d=json.load(sys.stdin); print('OK ✓' if d.get('state')=='RUNNING' else 'FALLO ✗')" 2>/dev/null || echo "FALLO ✗"

# 7. Aplicación accesible tras failover
echo -n "[8] Aplicación accesible en ms2 directamente: "
curl -s -o /dev/null -w "%{http_code}" http://localhost:7004/sample-app/ | \
  grep -q '200\|302' && echo "OK ✓" || echo "FALLO ✗"

echo -n "[9] Aplicación accesible via nginx tras failover: "
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/sample-app/ | \
  grep -q '200\|302' && echo "OK ✓" || echo "FALLO ✗"

# 8. Tasa de éxito JMeter > 95%
echo -n "[10] Tasa de éxito JMeter > 95%: "
python3 -c "
import csv
total = errors = 0
try:
    with open('/u01/lab07/jmeter-failover-results.csv') as f:
        for row in csv.DictReader(f):
            total += 1
            if row.get('responseCode','200') not in ('200','302'):
                errors += 1
    rate = (total-errors)/total*100 if total > 0 else 0
    print(f'OK ✓ ({rate:.1f}%)' if rate >= 95 else f'FALLO ✗ ({rate:.1f}%)')
except FileNotFoundError:
    print('ADVERTENCIA: archivo de resultados no encontrado')
"

echo "=========================================="
```

### Prueba de recuperación: reiniciar ms1

```bash
# Reinicia ms1 para restaurar el cluster completo
nohup ${DOMAIN_HOME}/bin/startManagedWebLogic.sh ms1 http://localhost:7001 \
  > /u01/lab07/logs/ms1-restart.log 2>&1 &

# Espera y verifica
sleep 60
curl -s -u weblogic:Welcome1 \
  "http://localhost:7001/management/weblogic/latest/domainRuntime/serverLifeCycleRuntimes/ms1" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print('ms1 reiniciado:', d.get('state','?'))"
```

---

## 8. Resolución de Problemas

### Problema 1: Las sesiones no se replican — el contador reinicia a 1 tras el failover

**Síntomas:**
- Tras matar ms1, las peticiones continúan en ms2 (HTTP 200), pero el contador de visitas de la sesión reinicia a 1 en lugar de continuar desde el valor anterior.
- En los logs de WebLogic se observa: `Session replication disabled` o `Session is not distributable`.

**Causa:**
El problema más común es que el WAR no incluye el elemento `<distributable/>` en su `web.xml`, o que el `persistent-store-type` en `weblogic.xml` no está configurado correctamente. Sin `<distributable/>`, WebLogic trata la sesión como local y no la replica al nodo secundario, aunque el servidor esté en un cluster. Otra causa es que los objetos almacenados en sesión no implementan `java.io.Serializable`, lo que impide su serialización para la replicación.

**Solución:**

```bash
# 1. Verificar <distributable/> en web.xml del WAR
unzip -p /u01/lab07/apps/sample-app-clustered.war WEB-INF/web.xml | grep distributable
# Debe mostrar: <distributable/>

# Si no está presente, reempaquetar con el elemento añadido:
mkdir -p /tmp/war-fix
cd /tmp/war-fix
jar -xf /u01/lab07/apps/sample-app-clustered.war
# Editar WEB-INF/web.xml y añadir <distributable/> dentro de <web-app>
sed -i 's|<web-app[^>]*>|&\n  <distributable/>|' WEB-INF/web.xml
jar -cf /u01/lab07/apps/sample-app-clustered.war .

# 2. Verificar que los objetos de sesión son Serializable
# Revisar el código fuente de la aplicación: todos los objetos guardados con
# session.setAttribute() deben implementar java.io.Serializable

# 3. Verificar persistent-store-type en weblogic.xml
unzip -p /u01/lab07/apps/sample-app-clustered.war WEB-INF/weblogic.xml | \
  grep persistent-store-type
# Debe mostrar: <persistent-store-type>replicated_if_clustered</persistent-store-type>

# 4. Redeployar la aplicación corregida
${WLS_HOME}/common/bin/wlst.sh << 'WLST_EOF'
connect('weblogic', 'Welcome1', 't3://localhost:7001')
undeploy('sample-app', targets='LabCluster')
deploy('sample-app', '/u01/lab07/apps/sample-app-clustered.war',
       targets='LabCluster', stageMode='stage')
disconnect()
exit()
WLST_EOF
```

---

### Problema 2: nginx devuelve HTTP 502 Bad Gateway tras el failover de ms1

**Síntomas:**
- Tras ejecutar `kill -9` en ms1, las peticiones a través del puerto 8080 de nginx devuelven `502 Bad Gateway`.
- El error persiste incluso después de que ms2 está claramente en estado `RUNNING`.
- Los logs de nginx muestran: `connect() failed (111: Connection refused) while connecting to upstream`.

**Causa:**
nginx tiene configurado `max_fails=3 fail_timeout=30s` para cada servidor upstream. Si el primer intento de reconexión a ms1 ocurre antes de que nginx marque el servidor como "down", nginx puede intentar enviar peticiones a ms1 (que ya no responde) antes de hacer failover a ms2. Adicionalmente, si `proxy_next_upstream` no está configurado, nginx no reintenta la petición en otro servidor tras un error de conexión, devolviendo directamente el 502.

**Solución:**

```bash
# 1. Verificar el estado de los upstream en nginx
# (requiere nginx stub_status o nginx Plus)
curl -s http://localhost:8080/nginx-health

# 2. Actualizar la configuración de nginx para mejorar el comportamiento de failover:
sudo tee /etc/nginx/conf.d/weblogic-cluster.conf << 'EOF'
upstream labcluster {
    ip_hash;
    server localhost:7003 weight=1 max_fails=1 fail_timeout=10s;
    server localhost:7004 weight=1 max_fails=1 fail_timeout=10s;
    keepalive 32;
}

server {
    listen 8080;
    server_name localhost;

    access_log /var/log/nginx/weblogic-cluster-access.log;
    error_log  /var/log/nginx/weblogic-cluster-error.log warn;

    location /sample-app/ {
        proxy_pass         http://labcluster;
        proxy_http_version 1.1;
        proxy_set_header   Connection        "";
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;

        # CLAVE: reintentar en el siguiente servidor ante errores de conexión
        proxy_next_upstream error timeout http_502 http_503;
        proxy_next_upstream_tries 2;
        proxy_next_upstream_timeout 5s;

        proxy_connect_timeout 5s;
        proxy_send_timeout    30s;
        proxy_read_timeout    30s;
    }
}
EOF

# 3. Recargar nginx sin interrupción del servicio
sudo nginx -t && sudo nginx -s reload

# 4. Verificar que el proxy funciona correctamente ahora
for i in $(seq 1 5); do
    curl -s -o /dev/null -w "Petición $i: HTTP %{http_code}\n" \
      http://localhost:8080/sample-app/
    sleep 1
done
```

> **Explicación:** `max_fails=1 fail_timeout=10s` hace que nginx marque un servidor como no disponible después del primer fallo, con un tiempo de espera de 10 segundos (más agresivo que los 30s originales). `proxy_next_upstream` instruye a nginx a reintentar la petición en el siguiente servidor disponible ante errores de conexión o respuestas 502/503, eliminando el error visible para el usuario final.

---

## 9. Limpieza del Entorno

Ejecuta los siguientes pasos para dejar el entorno en un estado limpio al finalizar el laboratorio:

```bash
echo "=== Iniciando limpieza del Lab 07-00-01 ==="

# 1. Detener JMeter si sigue corriendo
pkill -f jmeter 2>/dev/null && echo "JMeter detenido" || echo "JMeter no estaba corriendo"

# 2. Detener ms2 (ms1 ya fue matado en el paso 8)
${DOMAIN_HOME}/bin/stopManagedWebLogic.sh ms2 http://localhost:7001 weblogic Welcome1 2>/dev/null || \
  pkill -f 'weblogic.Name=ms2' 2>/dev/null
echo "ms2 detenido"

# 3. Si ms1 fue reiniciado, detenerlo también
${DOMAIN_HOME}/bin/stopManagedWebLogic.sh ms1 http://localhost:7001 weblogic Welcome1 2>/dev/null || \
  pkill -f 'weblogic.Name=ms1' 2>/dev/null
echo "ms1 detenido"

# 4. Detener el Administration Server
${DOMAIN_HOME}/bin/stopWebLogic.sh weblogic Welcome1 http://localhost:7001 2>/dev/null || \
  pkill -f 'weblogic.Name=AdminServer' 2>/dev/null
echo "AdminServer detenido"

# 5. Esperar a que los procesos terminen
sleep 10

# 6. Verificar que no hay procesos WebLogic corriendo
REMAINING=$(ps aux | grep '[w]eblogic.Server' | wc -l)
echo "Procesos WebLogic restantes: ${REMAINING}"

# 7. Detener nginx (o restaurar configuración previa)
sudo rm -f /etc/nginx/conf.d/weblogic-cluster.conf
sudo nginx -t && sudo systemctl reload nginx
echo "Configuración nginx restaurada"

# 8. Archivar los resultados del laboratorio para referencia futura
mkdir -p /u01/lab07/archive
cp /u01/lab07/jmeter-results.csv /u01/lab07/archive/ 2>/dev/null
cp /u01/lab07/jmeter-failover-results.csv /u01/lab07/archive/ 2>/dev/null
cp /u01/lab07/logs/*.log /u01/lab07/archive/ 2>/dev/null
echo "Logs archivados en /u01/lab07/archive/"

# 9. (Opcional) Eliminar el dominio si no se necesita en laboratorios posteriores
# DESCOMENTA SOLO SI ESTÁS SEGURO:
# rm -rf ${DOMAIN_HOME}
# echo "Dominio eliminado"

echo "=== Limpieza completada ==="
```

> **Nota:** El dominio `cluster-domain` en `/u01/domains/cluster-domain` se conserva por defecto, ya que puede ser útil como referencia para los laboratorios posteriores (Lab 08 de diagnóstico). Si deseas eliminarlo, descomenta la línea correspondiente en el script de limpieza.

---

## 10. Resumen

### Lo que has construido

En este laboratorio has diseñado e implementado una arquitectura completa de alta disponibilidad con WebLogic 15:

1. **Cluster estático `LabCluster`** con dos Managed Servers (`ms1:7003`, `ms2:7004`) en el mismo host, definidos explícitamente en `config.xml` mediante WDT — aplicando directamente los conceptos de la lección 7.1 sobre cluster estático vs dinámico.

2. **Replicación de sesiones HTTP en memoria** configurada mediante `weblogic.xml` con `persistent-store-type=replicated_if_clustered` y el elemento `<distributable/>` en `web.xml` — los dos requisitos fundamentales para que WebLogic propague el estado de sesión entre nodos.

3. **Proxy HTTP con nginx** distribuyendo carga entre `ms1` y `ms2` con `ip_hash` para session affinity, y configurado con `proxy_next_upstream` para failover transparente.

4. **Prueba de failover real** con `kill -9` en `ms1` mientras JMeter mantenía sesiones activas — confirmando que `ms2` continuó el servicio con tasa de éxito >99% y las sesiones preservadas.

5. **Análisis comparativo** entre in-memory replication y JDBC session persistence, identificando los criterios de elección para cada estrategia en entornos de producción.

### Conceptos clave consolidados

| Concepto                        | Implementación en el lab                                      |
|---------------------------------|---------------------------------------------------------------|
| Cluster estático                | Servidores explícitos en `config.xml` via WDT                 |
| `<distributable/>`              | Obligatorio en `web.xml` para replicación de sesión           |
| `replicated_if_clustered`       | `persistent-store-type` en `weblogic.xml`                     |
| Session affinity                | `ip_hash` en nginx upstream                                   |
| Failover transparente           | `proxy_next_upstream` en nginx + replicación in-memory        |
| JSESSIONID con server encoding  | `session_id!server_id` — permite routing inteligente          |
| Diferencia estático vs dinámico | Estático: control fino; Dinámico: elasticidad y homogeneidad  |

### Recursos adicionales

- **Documentación oficial:** [WebLogic Cluster Configuration Guide](https://docs.oracle.com/en/middleware/standalone/weblogic-server/14.1.2/clust/index.html)
- **WDT Reference:** [WebLogic Deploy Tooling — Model Reference](https://oracle.github.io/weblogic-deploy-tooling/)
- **Session Replication:** Oracle Fusion Middleware — *Developing Web Applications, Servlets, and JSPs for Oracle WebLogic Server* — Chapter: HTTP Session State Replication
- **Próximo laboratorio:** Lab 08-00-01 — Diagnóstico con WLDF, thread dumps y análisis de volcados de memoria bajo carga de cluster

---
