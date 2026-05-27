# Creación de un dominio desde cero con WDT. Buenas prácticas y reutilización de plantillas.

## 1. Metadatos

| Campo            | Valor                                      |
|------------------|--------------------------------------------|
| **Duración**     | 150 minutos                                |
| **Complejidad**  | Alta                                       |
| **Nivel Bloom**  | Crear (Create)                             |
| **Lab ID**       | 03-00-01                                   |
| **Módulo**       | 3 — WebLogic Deploy Tooling (WDT)          |

---

## 2. Descripción General

En este laboratorio aplicarás el paradigma **Infrastructure-as-Code (IaC)** a Oracle WebLogic 15 usando WebLogic Deploy Tooling (WDT) 4.x. Diseñarás y escribirás desde cero un modelo YAML completo que define topología, recursos y despliegues de aplicación, y luego lo materializarás en un dominio real ejecutando `createDomain.sh`. Aprenderás a parametrizar el modelo con **Variable Files** para hacerlo reutilizable en múltiples entornos y a empaquetar artefactos en un **Archive File**. Finalizarás demostrando el ciclo de vida completo con `updateDomain.sh` y aplicando buenas prácticas de versionado y documentación.

---

## 3. Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Instalar y configurar WDT 4.x verificando su estructura de directorios y scripts disponibles.
- [ ] Diseñar y escribir un modelo YAML completo con las secciones `domainInfo`, `topology`, `resources` y `appDeployments`.
- [ ] Crear un dominio WebLogic funcional ejecutando `createDomain.sh` a partir del modelo YAML y un Archive File.
- [ ] Utilizar Variable Files (`.properties`) para parametrizar el modelo y reutilizarlo en entornos dev/test/prod.
- [ ] Aplicar el ciclo de vida completo de WDT: `validateModel` → `createDomain` → `updateDomain`, siguiendo buenas prácticas de IaC.

---

## 4. Prerrequisitos

### Conocimientos previos

| Área                        | Nivel requerido                                                                 |
|-----------------------------|---------------------------------------------------------------------------------|
| Administración WebLogic     | Conceptos básicos de dominio, AdminServer y Managed Servers (Lab 02-00-01)     |
| Sintaxis YAML               | Indentación, tipos de datos, listas y mapas                                     |
| Línea de comandos Linux     | Navegación de directorios, variables de entorno, permisos de archivos           |
| Conceptos IaC               | Qué es un modelo declarativo e idempotencia (Lección 3.1)                       |

### Acceso y software

| Componente                          | Estado requerido                                                         |
|-------------------------------------|--------------------------------------------------------------------------|
| WebLogic 15 instalado               | `$MW_HOME` apuntando a `Oracle_Home` (resultado del Lab 01-00-01)       |
| JDK 21 configurado                  | `$JAVA_HOME` y `java -version` devuelven Java 21                        |
| WDT 4.x descargado                  | Archivo `weblogic-deploy.zip` disponible localmente o acceso a internet  |
| Aplicación de muestra Jakarta EE 10 | Archivo `sample-app.war` proporcionado por el instructor                 |
| Usuario del sistema                 | Permisos de escritura en `/u01/oracle/` y `/opt/`                        |

> **⚠️ Nota sobre snapshots:** Se recomienda tomar un snapshot de la VM antes de iniciar este laboratorio. Los cambios en el filesystem del dominio pueden dejar el entorno en estado inconsistente si se producen errores.

---

## 5. Entorno de Laboratorio

### Especificaciones de hardware recomendadas

| Recurso        | Mínimo          | Recomendado     |
|----------------|-----------------|-----------------|
| RAM            | 8 GB            | 16 GB           |
| CPU            | 4 núcleos       | 8 núcleos       |
| Almacenamiento | 20 GB libres    | 50 GB libres    |

### Variables de entorno del laboratorio

A lo largo de este laboratorio se usan las siguientes rutas de referencia. **Ajústalas a tu entorno real** antes de ejecutar cualquier comando:

```bash
# Instalación de WebLogic
export MW_HOME=/u01/oracle/middleware/Oracle_Home

# JDK 21
export JAVA_HOME=/u01/jdk/jdk-21

# Directorio de instalación de WDT
export WDT_HOME=/opt/weblogic-deploy

# Dominio que crearemos en este laboratorio
export DOMAIN_HOME=/u01/oracle/user_projects/domains/lab03Domain

# Directorio de trabajo del laboratorio
export LAB_DIR=/home/oracle/lab03

# Añadir WDT al PATH (se repetirá en el paso de instalación)
export PATH=$WDT_HOME/bin:$JAVA_HOME/bin:$PATH
```

> **Consejo:** Añade estas variables a `~/.bashrc` o crea un archivo `lab03-env.sh` que puedas cargar con `source lab03-env.sh` al inicio de cada sesión.

### Preparación del directorio de trabajo

```bash
mkdir -p $LAB_DIR/{model,variables,archive/wlsdeploy/applications,logs}
echo "Directorio de trabajo creado: $LAB_DIR"
ls -la $LAB_DIR
```

---

## 6. Pasos del Laboratorio

---

### Paso 1: Instalación y verificación de WDT 4.x

**Objetivo:** Instalar WDT en el entorno de laboratorio, verificar su estructura de directorios y confirmar que los scripts principales son ejecutables.

#### Instrucciones

1. **Descarga WDT** (si no está disponible localmente):

```bash
cd /tmp
curl -L -o weblogic-deploy.zip \
  https://github.com/oracle/weblogic-deploy-tooling/releases/latest/download/weblogic-deploy.zip

# Verifica la integridad del archivo descargado
ls -lh /tmp/weblogic-deploy.zip
```

> Si no hay acceso a internet, copia el archivo desde el recurso compartido del instructor:
> ```bash
> cp /mnt/instructor-share/wdt/weblogic-deploy.zip /tmp/
> ```

2. **Descomprime WDT** en el directorio de instalación:

```bash
sudo mkdir -p /opt/weblogic-deploy
sudo unzip /tmp/weblogic-deploy.zip -d /opt/
sudo chown -R oracle:oracle /opt/weblogic-deploy
```

3. **Configura las variables de entorno** (si no lo hiciste en la sección anterior):

```bash
export WDT_HOME=/opt/weblogic-deploy
export PATH=$WDT_HOME/bin:$PATH
echo "WDT_HOME=$WDT_HOME"
```

4. **Explora la estructura de directorios de WDT:**

```bash
ls -la $WDT_HOME/
ls -la $WDT_HOME/bin/
ls -la $WDT_HOME/lib/
```

5. **Verifica que los scripts principales son ejecutables:**

```bash
ls -la $WDT_HOME/bin/*.sh | awk '{print $1, $9}'
```

6. **Comprueba la versión instalada:**

```bash
$WDT_HOME/bin/validateModel.sh -version 2>&1 | head -5
```

#### Salida esperada

La exploración del directorio `bin/` debe mostrar al menos los siguientes scripts:

```
-rwxr-xr-x  createDomain.sh
-rwxr-xr-x  updateDomain.sh
-rwxr-xr-x  deployApps.sh
-rwxr-xr-x  discoverDomain.sh
-rwxr-xr-x  validateModel.sh
-rwxr-xr-x  encryptModel.sh
-rwxr-xr-x  extractDomainResource.sh
```

#### Verificación

```bash
# Debe devolver la ruta sin error
which createDomain.sh

# Debe mostrar información de versión de WDT 4.x
createDomain.sh -help 2>&1 | head -3
```

---

### Paso 2: Preparación del Archive File con la aplicación Jakarta EE 10

**Objetivo:** Empaquetar el WAR de la aplicación de muestra en un Archive File (`.zip`) con la estructura de directorios que WDT espera, de modo que pueda ser referenciado desde el modelo YAML.

#### Instrucciones

1. **Verifica que tienes el WAR de la aplicación de muestra:**

```bash
# El instructor habrá proporcionado este archivo
ls -lh /home/oracle/sample-app.war
```

> Si el archivo está en otra ubicación, ajusta la ruta en el siguiente comando.

2. **Crea la estructura de directorios del archive:**

```bash
mkdir -p $LAB_DIR/archive/wlsdeploy/applications
mkdir -p $LAB_DIR/archive/wlsdeploy/sharedLibraries
```

3. **Copia el WAR a la ubicación correcta dentro del archive:**

```bash
cp /home/oracle/sample-app.war \
   $LAB_DIR/archive/wlsdeploy/applications/sample-app.war

# Verifica la copia
ls -lh $LAB_DIR/archive/wlsdeploy/applications/
```

4. **Genera el Archive File ZIP:**

```bash
cd $LAB_DIR/archive
zip -r $LAB_DIR/lab03-archive.zip wlsdeploy/
cd $LAB_DIR

# Verifica el contenido del ZIP
unzip -l $LAB_DIR/lab03-archive.zip
```

#### Salida esperada

```
Archive:  /home/oracle/lab03/lab03-archive.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
        0  2024-xx-xx xx:xx   wlsdeploy/
        0  2024-xx-xx xx:xx   wlsdeploy/applications/
  xxxxxxx  2024-xx-xx xx:xx   wlsdeploy/applications/sample-app.war
---------                     -------
  xxxxxxx                     3 files
```

#### Verificación

```bash
# El archive debe existir y tener un tamaño mayor que 0
test -s $LAB_DIR/lab03-archive.zip && echo "Archive OK" || echo "ERROR: archive vacío o inexistente"
```

---

### Paso 3: Diseño del modelo YAML — Sección `domainInfo` y `topology`

**Objetivo:** Escribir las secciones `domainInfo` y `topology` del modelo YAML, definiendo el Administration Server, dos Managed Servers y un cluster estático.

#### Instrucciones

1. **Crea el archivo del modelo principal:**

```bash
cat > $LAB_DIR/model/lab03-model.yaml << 'EOF'
# ============================================================
# Modelo WDT - Lab 03-00-01
# Dominio: lab03Domain
# Versión del modelo: 1.0.0
# Autor: <tu-nombre>
# Fecha: <fecha-actual>
# Descripción: Dominio de laboratorio con cluster estático,
#              JDBC, JMS y despliegue de aplicación Jakarta EE 10.
# ============================================================

domainInfo:
  AdminUserName: '@@PROP:admin.username@@'
  AdminPassword: '@@PROP:admin.password@@'
  ServerStartMode: prod

topology:
  Name: '@@PROP:domain.name@@'
  AdminServerName: AdminServer
  ProductionModeEnabled: true

  AdminServer:
    ListenAddress: '@@PROP:admin.listen.address@@'
    ListenPort: '@@PROP:admin.listen.port@@'
    SSL:
      Enabled: false
    ServerStart:
      Arguments: '-Xms512m -Xmx1024m -XX:+UseG1GC'

  Cluster:
    lab03Cluster:
      ClusterMessagingMode: unicast
      ClusterAddress: '@@PROP:cluster.address@@'

  Server:
    ManagedServer1:
      ListenAddress: '@@PROP:ms1.listen.address@@'
      ListenPort: '@@PROP:ms1.listen.port@@'
      Cluster: lab03Cluster
      SSL:
        Enabled: false
      ServerStart:
        Arguments: '-Xms512m -Xmx1024m -XX:+UseG1GC'

    ManagedServer2:
      ListenAddress: '@@PROP:ms2.listen.address@@'
      ListenPort: '@@PROP:ms2.listen.port@@'
      Cluster: lab03Cluster
      SSL:
        Enabled: false
      ServerStart:
        Arguments: '-Xms512m -Xmx1024m -XX:+UseG1GC'

EOF

echo "Sección topology escrita correctamente."
cat $LAB_DIR/model/lab03-model.yaml
```

> **Nota sobre la sintaxis `@@PROP:nombre@@`:** Esta es la sintaxis de WDT para referenciar variables externas definidas en un Variable File (`.properties`). Ningún valor sensible queda en claro en el modelo YAML.

#### Verificación preliminar

```bash
# Verifica que el archivo existe y tiene contenido
wc -l $LAB_DIR/model/lab03-model.yaml
```

---

### Paso 4: Diseño del modelo YAML — Sección `resources` (JDBC, JMS, Mail Session)

**Objetivo:** Añadir la sección `resources` al modelo YAML, incluyendo un JDBC Data Source, un JMS Module con una cola y una Mail Session.

#### Instrucciones

1. **Añade la sección `resources` al modelo:**

```bash
cat >> $LAB_DIR/model/lab03-model.yaml << 'EOF'

resources:

  # ---- JDBC Data Source ----
  JDBCSystemResource:
    lab03DataSource:
      Target: 'lab03Cluster'
      JdbcResource:
        JDBCDataSourceParams:
          JNDIName:
            - jdbc/lab03DS
          GlobalTransactionsProtocol: TwoPhaseCommit
        JDBCDriverParams:
          DriverName: oracle.jdbc.OracleDriver
          URL: '@@PROP:jdbc.url@@'
          PasswordEncrypted: '@@PROP:jdbc.password@@'
          Properties:
            user:
              Value: '@@PROP:jdbc.username@@'
        JDBCConnectionPoolParams:
          InitialCapacity: 5
          MaxCapacity: 25
          MinCapacity: 5
          TestConnectionsOnReserve: true
          TestTableName: SQL SELECT 1 FROM DUAL

  # ---- JMS Module ----
  JMSSystemResource:
    lab03JMSModule:
      Target: 'lab03Cluster'
      JmsResource:
        ConnectionFactory:
          lab03ConnectionFactory:
            JNDIName: jms/lab03CF
            DefaultDeliveryParams:
              DefaultDeliveryMode: Persistent
        Queue:
          lab03Queue:
            JNDIName: jms/lab03Queue
            DeliveryFailureParams:
              RedeliveryLimit: 3

  # ---- Mail Session ----
  MailSession:
    lab03MailSession:
      JNDIName: mail/lab03Mail
      Target: 'lab03Cluster'
      Properties:
        mail.transport.protocol: smtp
        mail.smtp.host: '@@PROP:mail.smtp.host@@'
        mail.smtp.port: '@@PROP:mail.smtp.port@@'
        mail.smtp.auth: false
        mail.from: '@@PROP:mail.from@@'

EOF

echo "Sección resources añadida correctamente."
```

#### Verificación

```bash
grep -c "@@PROP:" $LAB_DIR/model/lab03-model.yaml
echo "Variables referenciadas en el modelo (debe ser > 0)"
```

---

### Paso 5: Diseño del modelo YAML — Sección `appDeployments`

**Objetivo:** Añadir la sección `appDeployments` al modelo YAML para referenciar el WAR empaquetado en el Archive File del Paso 2.

#### Instrucciones

1. **Añade la sección `appDeployments` al modelo:**

```bash
cat >> $LAB_DIR/model/lab03-model.yaml << 'EOF'

appDeployments:
  Application:
    sample-app:
      SourcePath: 'wlsdeploy/applications/sample-app.war'
      ModuleType: war
      Target: 'lab03Cluster'
      StagingMode: stage
      PlanStagingMode: stage

EOF

echo "Sección appDeployments añadida correctamente."
```

2. **Visualiza el modelo completo para revisión:**

```bash
cat $LAB_DIR/model/lab03-model.yaml
```

3. **Cuenta las líneas totales del modelo:**

```bash
wc -l $LAB_DIR/model/lab03-model.yaml
echo "El modelo completo debe tener aproximadamente 90-110 líneas."
```

---

### Paso 6: Creación del Variable File para el entorno de desarrollo

**Objetivo:** Crear un archivo de variables (`.properties`) que externalice todos los valores dependientes del entorno, permitiendo reutilizar el mismo modelo YAML en dev/test/prod cambiando únicamente este archivo.

#### Instrucciones

1. **Crea el Variable File para el entorno `dev`:**

```bash
cat > $LAB_DIR/variables/lab03-dev.properties << 'EOF'
# ============================================================
# Variable File - Entorno: DEV
# Lab 03-00-01 | lab03Domain
# ADVERTENCIA: Este archivo contiene credenciales de laboratorio.
# NUNCA usar estas credenciales en producción.
# Excluir de control de versiones o usar vault/secrets manager.
# ============================================================

# --- Credenciales de administración ---
admin.username=weblogic
admin.password=Welcome1

# --- Configuración del dominio ---
domain.name=lab03Domain

# --- Administration Server ---
admin.listen.address=localhost
admin.listen.port=7001

# --- Cluster ---
cluster.address=localhost

# --- Managed Server 1 ---
ms1.listen.address=localhost
ms1.listen.port=8001

# --- Managed Server 2 ---
ms2.listen.address=localhost
ms2.listen.port=8002

# --- JDBC ---
jdbc.url=jdbc:oracle:thin:@localhost:1521/XEPDB1
jdbc.username=lab03user
jdbc.password=Lab03Pass1

# --- Mail ---
mail.smtp.host=localhost
mail.smtp.port=25
mail.from=lab03@example.com
EOF

echo "Variable File DEV creado."
cat $LAB_DIR/variables/lab03-dev.properties
```

2. **Crea un Variable File para el entorno `prod`** (plantilla con valores de referencia — sin credenciales reales):

```bash
cat > $LAB_DIR/variables/lab03-prod.properties << 'EOF'
# ============================================================
# Variable File - Entorno: PROD (PLANTILLA)
# Lab 03-00-01 | lab03Domain
# INSTRUCCIONES: Rellenar con valores reales antes de usar.
# En producción, gestionar con Oracle Vault o HashiCorp Vault.
# ============================================================

# --- Credenciales de administración ---
admin.username=CHANGE_ME
admin.password=CHANGE_ME

# --- Configuración del dominio ---
domain.name=lab03DomainProd

# --- Administration Server ---
admin.listen.address=wls-admin.prod.example.com
admin.listen.port=7001

# --- Cluster ---
cluster.address=wls-ms1.prod.example.com,wls-ms2.prod.example.com

# --- Managed Server 1 ---
ms1.listen.address=wls-ms1.prod.example.com
ms1.listen.port=8001

# --- Managed Server 2 ---
ms2.listen.address=wls-ms2.prod.example.com
ms2.listen.port=8002

# --- JDBC ---
jdbc.url=jdbc:oracle:thin:@db.prod.example.com:1521/PRODDB
jdbc.username=CHANGE_ME
jdbc.password=CHANGE_ME

# --- Mail ---
mail.smtp.host=smtp.prod.example.com
mail.smtp.port=587
mail.from=weblogic@prod.example.com
EOF

echo "Variable File PROD (plantilla) creado."
```

3. **Establece permisos restrictivos en los archivos de variables** (buena práctica de seguridad):

```bash
chmod 600 $LAB_DIR/variables/lab03-dev.properties
chmod 600 $LAB_DIR/variables/lab03-prod.properties
ls -la $LAB_DIR/variables/
```

#### Verificación

```bash
# Verifica que todas las variables del modelo están definidas en el properties
echo "=== Variables en el modelo ==="
grep -o '@@PROP:[^@]*@@' $LAB_DIR/model/lab03-model.yaml | sort -u

echo ""
echo "=== Claves en el Variable File DEV ==="
grep -v '^#' $LAB_DIR/variables/lab03-dev.properties | grep '=' | cut -d'=' -f1 | sort
```

> Asegúrate de que cada `@@PROP:clave@@` del modelo tiene su correspondiente `clave=valor` en el archivo de variables.

---

### Paso 7: Validación del modelo con `validateModel.sh`

**Objetivo:** Ejecutar `validateModel.sh` para detectar errores de sintaxis, referencias inválidas o secciones mal estructuradas antes de crear el dominio.

#### Instrucciones

1. **Ejecuta la validación del modelo:**

```bash
$WDT_HOME/bin/validateModel.sh \
  -oracle_home $MW_HOME \
  -model_file $LAB_DIR/model/lab03-model.yaml \
  -variable_file $LAB_DIR/variables/lab03-dev.properties \
  -archive_file $LAB_DIR/lab03-archive.zip \
  2>&1 | tee $LAB_DIR/logs/validate-output.log
```

2. **Revisa el resultado de la validación:**

```bash
# Busca errores o advertencias
grep -E "ERROR|WARNING|SEVERE" $LAB_DIR/logs/validate-output.log || echo "Sin errores ni advertencias"

# Busca la línea de resultado final
grep -E "Validation completed|validation" $LAB_DIR/logs/validate-output.log | tail -5
```

3. **Si hay errores, corrígelos antes de continuar.** Los errores más comunes en este punto son:
   - Indentación YAML incorrecta.
   - Referencia a una variable `@@PROP:clave@@` sin definición en el `.properties`.
   - Sección `SourcePath` del archive apuntando a una ruta que no existe en el ZIP.

#### Salida esperada

```
<fecha> <hora> INFO Performing model validation...
<fecha> <hora> INFO Model file validation is complete
<fecha> <hora> INFO Validation completed successfully
```

#### Verificación

```bash
# La validación debe terminar con código de salida 0
echo "Código de salida de validateModel: $?"
test $? -eq 0 && echo "✅ Validación exitosa" || echo "❌ Validación fallida — revisar logs"
```

---

### Paso 8: Creación del dominio con `createDomain.sh`

**Objetivo:** Ejecutar `createDomain.sh` para crear el dominio WebLogic completo en modo offline a partir del modelo YAML, el Variable File y el Archive File.

#### Instrucciones

1. **Crea el directorio destino del dominio:**

```bash
mkdir -p $DOMAIN_HOME
echo "Directorio del dominio: $DOMAIN_HOME"
```

2. **Ejecuta `createDomain.sh`:**

```bash
$WDT_HOME/bin/createDomain.sh \
  -oracle_home $MW_HOME \
  -domain_home $DOMAIN_HOME \
  -domain_type WLS \
  -model_file $LAB_DIR/model/lab03-model.yaml \
  -variable_file $LAB_DIR/variables/lab03-dev.properties \
  -archive_file $LAB_DIR/lab03-archive.zip \
  2>&1 | tee $LAB_DIR/logs/createDomain-output.log
```

> **⏱️ Tiempo estimado:** La creación del dominio puede tardar entre 3 y 8 minutos dependiendo del hardware. Es normal ver mensajes de progreso durante este tiempo.

3. **Monitoriza el progreso en tiempo real** (en una segunda terminal):

```bash
tail -f $LAB_DIR/logs/createDomain-output.log
```

4. **Una vez completado, verifica la estructura del dominio creado:**

```bash
ls -la $DOMAIN_HOME/
ls -la $DOMAIN_HOME/config/
ls -la $DOMAIN_HOME/servers/
```

#### Salida esperada

Al finalizar correctamente, el log debe mostrar:

```
<fecha> <hora> INFO Domain creation is complete
<fecha> <hora> INFO Successfully created domain: lab03Domain
```

La estructura de directorios debe incluir:

```
$DOMAIN_HOME/
├── bin/
├── config/
│   ├── config.xml
│   ├── jdbc/
│   └── jms/
├── security/
├── servers/
│   ├── AdminServer/
│   ├── ManagedServer1/
│   └── ManagedServer2/
└── startWebLogic.sh
```

#### Verificación

```bash
# Verifica que config.xml existe y contiene el nombre del dominio
grep "<name>lab03Domain</name>" $DOMAIN_HOME/config/config.xml && \
  echo "✅ Dominio lab03Domain creado correctamente" || \
  echo "❌ ERROR: config.xml no encontrado o nombre incorrecto"

# Verifica que los servidores están definidos en config.xml
grep -E "ManagedServer1|ManagedServer2|lab03Cluster" $DOMAIN_HOME/config/config.xml

# Verifica que el JDBC descriptor fue creado
ls -la $DOMAIN_HOME/config/jdbc/

# Verifica que la aplicación fue desplegada
ls -la $DOMAIN_HOME/config/deployments/ 2>/dev/null || \
  find $DOMAIN_HOME -name "*.war" 2>/dev/null | head -5
```

---

### Paso 9: Arranque del Administration Server y validación del dominio

**Objetivo:** Arrancar el Administration Server del dominio recién creado y verificar que todos los recursos (JDBC, JMS, aplicación) están correctamente configurados.

#### Instrucciones

1. **Arranca el Administration Server:**

```bash
nohup $DOMAIN_HOME/startWebLogic.sh \
  > $LAB_DIR/logs/adminserver-startup.log 2>&1 &

echo "PID del proceso: $!"
echo "Monitorizando arranque..."
```

2. **Monitoriza el arranque hasta que el servidor esté listo:**

```bash
# Espera hasta que aparezca el mensaje de servidor listo (máx. 3 minutos)
timeout 180 bash -c \
  'until grep -q "Server state changed to RUNNING" \
   /home/oracle/lab03/logs/adminserver-startup.log 2>/dev/null; do
     echo "Esperando arranque del AdminServer..."
     sleep 5
   done && echo "✅ AdminServer en estado RUNNING"'
```

3. **Verifica la conectividad HTTP:**

```bash
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" \
  http://localhost:7001/console/login/LoginForm.jsp
```

4. **Verifica el estado del servidor via REST API de WebLogic:**

```bash
curl -s -u weblogic:Welcome1 \
  http://localhost:7001/management/weblogic/latest/domainRuntime/serverLifeCycleRuntimes \
  | python3 -m json.tool 2>/dev/null | grep -E '"name"|"state"' | head -20
```

5. **Verifica los JDBC Data Sources vía REST:**

```bash
curl -s -u weblogic:Welcome1 \
  "http://localhost:7001/management/weblogic/latest/domainConfig/JDBCSystemResources" \
  | python3 -m json.tool 2>/dev/null | grep '"name"'
```

#### Salida esperada

```
HTTP Status: 200
```

Para la consulta REST de servidores:

```json
"name": "AdminServer",
"state": "RUNNING",
```

#### Verificación

```bash
# Verifica que el AdminServer responde en el puerto 7001
nc -zv localhost 7001 2>&1 | grep -E "Connected|succeeded" && \
  echo "✅ Puerto 7001 accesible" || \
  echo "❌ Puerto 7001 no accesible — revisar logs de arranque"
```

---

### Paso 10: Modificación del modelo y ciclo de vida con `updateDomain.sh`

**Objetivo:** Demostrar el ciclo de vida completo de IaC modificando el modelo YAML y aplicando los cambios con `updateDomain.sh`, tanto en modo offline como online.

#### Instrucciones

1. **Crea una versión modificada del modelo** que añade un parámetro de JVM adicional y cambia la capacidad máxima del pool JDBC:

```bash
# Crea una copia del modelo para la actualización
cp $LAB_DIR/model/lab03-model.yaml $LAB_DIR/model/lab03-model-v1.1.yaml

# Edita la copia — modifica MaxCapacity del pool JDBC
sed -i 's/MaxCapacity: 25/MaxCapacity: 50/' \
  $LAB_DIR/model/lab03-model-v1.1.yaml

# Verifica el cambio
grep "MaxCapacity" $LAB_DIR/model/lab03-model-v1.1.yaml
```

2. **Documenta el cambio en el encabezado del modelo** (buena práctica de versionado):

```bash
# Actualiza la versión y descripción en el encabezado del modelo
sed -i 's/# Versión del modelo: 1.0.0/# Versión del modelo: 1.1.0/' \
  $LAB_DIR/model/lab03-model-v1.1.yaml
sed -i 's/# Fecha: <fecha-actual>/# Fecha: '"$(date +%Y-%m-%d)"'/' \
  $LAB_DIR/model/lab03-model-v1.1.yaml

head -10 $LAB_DIR/model/lab03-model-v1.1.yaml
```

3. **Valida el modelo actualizado antes de aplicar:**

```bash
$WDT_HOME/bin/validateModel.sh \
  -oracle_home $MW_HOME \
  -model_file $LAB_DIR/model/lab03-model-v1.1.yaml \
  -variable_file $LAB_DIR/variables/lab03-dev.properties \
  -archive_file $LAB_DIR/lab03-archive.zip \
  2>&1 | tail -5
```

4. **Detén el Administration Server** para realizar la actualización en modo offline:

```bash
# Detiene el AdminServer de forma ordenada
$DOMAIN_HOME/bin/stopWebLogic.sh \
  weblogic Welcome1 t3://localhost:7001 \
  2>&1 | tee $LAB_DIR/logs/stopAdminServer.log

# Espera a que se detenga completamente
sleep 15
echo "AdminServer detenido."
```

5. **Aplica los cambios con `updateDomain.sh` en modo offline:**

```bash
$WDT_HOME/bin/updateDomain.sh \
  -oracle_home $MW_HOME \
  -domain_home $DOMAIN_HOME \
  -model_file $LAB_DIR/model/lab03-model-v1.1.yaml \
  -variable_file $LAB_DIR/variables/lab03-dev.properties \
  -archive_file $LAB_DIR/lab03-archive.zip \
  2>&1 | tee $LAB_DIR/logs/updateDomain-output.log
```

6. **Verifica que el cambio se aplicó en `config.xml`:**

```bash
grep -A5 "lab03DataSource" $DOMAIN_HOME/config/jdbc/lab03DataSource*.xml 2>/dev/null | \
  grep -i "max-capacity\|MaxCapacity" || \
  grep "max-capacity" $DOMAIN_HOME/config/jdbc/*.xml
```

7. **(Alternativa) Demostración de `updateDomain.sh` en modo online** — Vuelve a arrancar el AdminServer y aplica un cambio menor online:

```bash
# Arranca el AdminServer nuevamente
nohup $DOMAIN_HOME/startWebLogic.sh \
  > $LAB_DIR/logs/adminserver-startup2.log 2>&1 &

# Espera al estado RUNNING
timeout 180 bash -c \
  'until grep -q "Server state changed to RUNNING" \
   /home/oracle/lab03/logs/adminserver-startup2.log 2>/dev/null; do
     sleep 5; done && echo "AdminServer listo"'

# Aplica un cambio online (ejemplo: añadir una propiedad de descripción)
# Nota: updateDomain online requiere que el modelo solo contenga
# cambios compatibles con modo online
$WDT_HOME/bin/updateDomain.sh \
  -oracle_home $MW_HOME \
  -admin_url t3://localhost:7001 \
  -user weblogic \
  -password Welcome1 \
  -model_file $LAB_DIR/model/lab03-model-v1.1.yaml \
  -variable_file $LAB_DIR/variables/lab03-dev.properties \
  2>&1 | tee $LAB_DIR/logs/updateDomain-online-output.log
```

#### Salida esperada

```
<fecha> <hora> INFO Domain update is complete
<fecha> <hora> INFO Successfully updated domain: lab03Domain
```

#### Verificación

```bash
grep -E "ERROR|SEVERE" $LAB_DIR/logs/updateDomain-output.log || \
  echo "✅ updateDomain completado sin errores"
```

---

### Paso 11: Exploración de `discoverDomain.sh` y buenas prácticas de versionado

**Objetivo:** Usar `discoverDomain.sh` para generar un modelo desde el dominio creado, compararlo con el modelo original y establecer buenas prácticas de versionado y documentación.

#### Instrucciones

1. **Ejecuta `discoverDomain.sh`** para generar un modelo desde el dominio existente:

```bash
mkdir -p $LAB_DIR/discovered

$WDT_HOME/bin/discoverDomain.sh \
  -oracle_home $MW_HOME \
  -domain_home $DOMAIN_HOME \
  -model_file $LAB_DIR/discovered/discovered-model.yaml \
  -archive_file $LAB_DIR/discovered/discovered-archive.zip \
  -variable_file $LAB_DIR/discovered/discovered-variables.properties \
  2>&1 | tee $LAB_DIR/logs/discoverDomain-output.log
```

2. **Compara el modelo descubierto con el original:**

```bash
# Comparación de estructura de secciones
echo "=== Secciones en modelo original ==="
grep -E "^[a-zA-Z]" $LAB_DIR/model/lab03-model.yaml

echo ""
echo "=== Secciones en modelo descubierto ==="
grep -E "^[a-zA-Z]" $LAB_DIR/discovered/discovered-model.yaml
```

3. **Establece la estructura de versionado recomendada:**

```bash
# Crea una estructura de repositorio Git simulada
mkdir -p $LAB_DIR/git-repo/{models,variables,archives}

# Copia los artefactos del laboratorio
cp $LAB_DIR/model/lab03-model.yaml \
   $LAB_DIR/git-repo/models/lab03-model-v1.0.0.yaml
cp $LAB_DIR/model/lab03-model-v1.1.yaml \
   $LAB_DIR/git-repo/models/lab03-model-v1.1.0.yaml
cp $LAB_DIR/variables/lab03-dev.properties \
   $LAB_DIR/git-repo/variables/lab03-dev.properties.template
cp $LAB_DIR/variables/lab03-prod.properties \
   $LAB_DIR/git-repo/variables/lab03-prod.properties.template

# Crea un README de buenas prácticas
cat > $LAB_DIR/git-repo/README.md << 'EOF'
# Repositorio de Modelos WDT - lab03Domain

## Estructura
- `models/`    — Modelos YAML versionados (formato: nombre-vMAJOR.MINOR.PATCH.yaml)
- `variables/` — Plantillas de Variable Files por entorno (sin credenciales reales)
- `archives/`  — Archive Files de aplicaciones

## Convenciones de Nomenclatura
- Modelos: `<dominio>-model-vX.Y.Z.yaml`
- Variables: `<dominio>-<entorno>.properties.template`
- Archives: `<dominio>-archive-vX.Y.Z.zip`

## Reglas de Seguridad
- NUNCA commitear archivos .properties con credenciales reales
- Usar .gitignore para excluir archivos sensibles
- Gestionar secrets con Oracle Vault o HashiCorp Vault en producción

## Ciclo de Vida
1. Editar modelo en rama feature/
2. Ejecutar validateModel.sh en CI
3. Revisar cambios vía Pull Request
4. Aplicar en dev con createDomain/updateDomain
5. Promover a prod con Variable File de producción
EOF

echo "Estructura de repositorio creada:"
find $LAB_DIR/git-repo -type f | sort
```

4. **Crea un `.gitignore` para excluir archivos sensibles:**

```bash
cat > $LAB_DIR/git-repo/.gitignore << 'EOF'
# Archivos de variables con credenciales reales (NO commitear)
variables/*.properties
!variables/*.properties.template

# Archivos de contraseñas cifradas de WDT
*.enc

# Logs de WDT
logs/

# Directorios de dominio generados
domains/

# Archivos temporales
*.tmp
*.bak
EOF

cat $LAB_DIR/git-repo/.gitignore
```

#### Verificación

```bash
# Verifica que el modelo descubierto contiene las secciones principales
for section in topology resources appDeployments; do
  grep -q "^$section:" $LAB_DIR/discovered/discovered-model.yaml && \
    echo "✅ Sección '$section' presente en modelo descubierto" || \
    echo "⚠️  Sección '$section' NO encontrada en modelo descubierto"
done
```

---

## 7. Validación y Pruebas Finales

Una vez completados todos los pasos, ejecuta la siguiente secuencia de validación integral:

```bash
echo "=============================================="
echo " VALIDACIÓN INTEGRAL - Lab 03-00-01"
echo "=============================================="

# 1. Verificar WDT instalado
echo "[1/6] WDT instalado..."
test -x $WDT_HOME/bin/createDomain.sh && echo "  ✅ OK" || echo "  ❌ FALLO"

# 2. Verificar modelo YAML completo
echo "[2/6] Modelo YAML con todas las secciones..."
for section in domainInfo topology resources appDeployments; do
  grep -q "^$section:" $LAB_DIR/model/lab03-model-v1.1.yaml && \
    echo "  ✅ Sección '$section' presente" || \
    echo "  ❌ Sección '$section' AUSENTE"
done

# 3. Verificar Variable Files
echo "[3/6] Variable Files..."
test -f $LAB_DIR/variables/lab03-dev.properties && echo "  ✅ DEV OK" || echo "  ❌ DEV FALLO"
test -f $LAB_DIR/variables/lab03-prod.properties && echo "  ✅ PROD OK" || echo "  ❌ PROD FALLO"

# 4. Verificar Archive File
echo "[4/6] Archive File..."
unzip -l $LAB_DIR/lab03-archive.zip | grep -q "sample-app.war" && \
  echo "  ✅ WAR en archive OK" || echo "  ❌ WAR NO encontrado en archive"

# 5. Verificar dominio creado
echo "[5/6] Dominio creado..."
test -f $DOMAIN_HOME/config/config.xml && \
  echo "  ✅ config.xml presente" || echo "  ❌ config.xml AUSENTE"
grep -q "lab03Cluster" $DOMAIN_HOME/config/config.xml && \
  echo "  ✅ Cluster definido" || echo "  ❌ Cluster NO encontrado"

# 6. Verificar AdminServer activo
echo "[6/6] Administration Server..."
curl -s -o /dev/null -w "  HTTP %{http_code}" \
  http://localhost:7001/console/login/LoginForm.jsp 2>/dev/null && echo " ✅" || \
  echo "  ⚠️  AdminServer no disponible (puede estar detenido)"

echo "=============================================="
echo " Validación completada."
echo "=============================================="
```

### Revisión manual adicional

Accede a la consola de administración de WebLogic en `http://localhost:7001/console` con las credenciales `weblogic/Welcome1` y verifica visualmente:

| Elemento                     | Ruta en consola                                        | Estado esperado    |
|------------------------------|--------------------------------------------------------|--------------------|
| Dominio `lab03Domain`        | Home → Domain Structure                               | Visible            |
| Cluster `lab03Cluster`       | Environment → Clusters                                | Activo             |
| `ManagedServer1` y `MS2`     | Environment → Servers                                 | Configurados       |
| `lab03DataSource`            | Services → Data Sources                               | Configurado        |
| `lab03JMSModule`             | Services → Messaging → JMS Modules                    | Configurado        |
| `sample-app`                 | Deployments                                           | Desplegado         |

---

## 8. Resolución de Problemas

### Problema 1: `createDomain.sh` falla con "WLSDPLY-12400: Unable to find domain template"

**Síntomas:**
- El comando `createDomain.sh` termina con código de error distinto de 0.
- El log contiene un mensaje similar a: `WLSDPLY-12400: Unable to find domain template for domain type WLS`.
- El proceso se detiene antes de crear cualquier archivo en `$DOMAIN_HOME`.

**Causa probable:**
La variable `$MW_HOME` no apunta a la instalación correcta de WebLogic, o la instalación de WebLogic está incompleta. WDT necesita localizar los templates de dominio incluidos en `$MW_HOME/wlserver/common/templates/wls/`.

**Solución:**

```bash
# 1. Verifica que MW_HOME apunta a la instalación correcta
echo "MW_HOME=$MW_HOME"
ls $MW_HOME/wlserver/common/templates/wls/

# 2. Verifica que el template base de WLS existe
ls $MW_HOME/wlserver/common/templates/wls/wls.jar

# 3. Si MW_HOME es incorrecto, corrígelo
# Busca la instalación real de WebLogic
find /u01 -name "wls.jar" -path "*/templates/*" 2>/dev/null | head -3

# 4. Actualiza MW_HOME con la ruta correcta
export MW_HOME=/ruta/correcta/Oracle_Home

# 5. Vuelve a ejecutar createDomain.sh con la variable corregida
$WDT_HOME/bin/createDomain.sh \
  -oracle_home $MW_HOME \
  -domain_home $DOMAIN_HOME \
  -domain_type WLS \
  -model_file $LAB_DIR/model/lab03-model.yaml \
  -variable_file $LAB_DIR/variables/lab03-dev.properties \
  -archive_file $LAB_DIR/lab03-archive.zip
```

---

### Problema 2: `validateModel.sh` reporta "WLSDPLY-05004: Variable ... not found in variable file"

**Síntomas:**
- `validateModel.sh` termina con errores de tipo `WLSDPLY-05004`.
- El mensaje indica que una variable `@@PROP:nombre.variable@@` referenciada en el modelo no tiene correspondencia en el Variable File.
- La validación falla aunque el YAML esté sintácticamente correcto.

**Causa probable:**
Hay una discrepancia entre las claves `@@PROP:clave@@` usadas en el modelo YAML y las claves definidas en el archivo `.properties`. Puede deberse a un error tipográfico en el nombre de la clave, a que se añadió una nueva variable al modelo pero no al archivo de variables, o a que se está usando el Variable File de un entorno diferente.

**Solución:**

```bash
# 1. Extrae todas las variables referenciadas en el modelo
echo "=== Variables en el modelo YAML ==="
grep -o '@@PROP:[^@]*@@' $LAB_DIR/model/lab03-model.yaml | \
  sed 's/@@PROP://;s/@@//' | sort -u > /tmp/model-vars.txt
cat /tmp/model-vars.txt

# 2. Extrae todas las claves definidas en el Variable File
echo ""
echo "=== Claves en el Variable File ==="
grep -v '^#' $LAB_DIR/variables/lab03-dev.properties | \
  grep '=' | cut -d'=' -f1 | sort -u > /tmp/props-keys.txt
cat /tmp/props-keys.txt

# 3. Encuentra las variables que faltan en el properties
echo ""
echo "=== Variables del modelo SIN definición en .properties ==="
comm -23 /tmp/model-vars.txt /tmp/props-keys.txt

# 4. Para cada variable faltante, añádela al archivo .properties
# Ejemplo: si falta 'mail.smtp.port'
echo "mail.smtp.port=25" >> $LAB_DIR/variables/lab03-dev.properties

# 5. Vuelve a validar
$WDT_HOME/bin/validateModel.sh \
  -oracle_home $MW_HOME \
  -model_file $LAB_DIR/model/lab03-model.yaml \
  -variable_file $LAB_DIR/variables/lab03-dev.properties \
  -archive_file $LAB_DIR/lab03-archive.zip \
  2>&1 | tail -5
```

---

## 9. Limpieza del Entorno

> **⚠️ Advertencia:** Ejecuta estos pasos solo si deseas eliminar completamente el dominio y los artefactos del laboratorio. Los laboratorios posteriores (04-00-01, 04-00-02, etc.) **requieren** el dominio creado en este laboratorio. Consulta las dependencias antes de proceder.

```bash
# 1. Detén el Administration Server si está en ejecución
if nc -z localhost 7001 2>/dev/null; then
  echo "Deteniendo Administration Server..."
  $DOMAIN_HOME/bin/stopWebLogic.sh \
    weblogic Welcome1 t3://localhost:7001 2>/dev/null || true
  sleep 10
fi

# 2. Elimina el dominio creado
# PRECAUCIÓN: Esto es irreversible
read -p "¿Eliminar el dominio $DOMAIN_HOME? [s/N]: " confirm
if [[ "$confirm" =~ ^[Ss]$ ]]; then
  rm -rf $DOMAIN_HOME
  echo "Dominio eliminado: $DOMAIN_HOME"
else
  echo "Operación cancelada. Dominio conservado."
fi

# 3. (Opcional) Elimina los artefactos del laboratorio
read -p "¿Eliminar directorio de trabajo $LAB_DIR? [s/N]: " confirm2
if [[ "$confirm2" =~ ^[Ss]$ ]]; then
  rm -rf $LAB_DIR
  echo "Directorio de trabajo eliminado: $LAB_DIR"
fi

# 4. WDT NO se elimina — se reutilizará en laboratorios posteriores
echo "WDT conservado en $WDT_HOME para uso en laboratorios futuros."
```

---

## 10. Resumen

En este laboratorio has completado el ciclo de vida completo de **Infrastructure-as-Code con WebLogic Deploy Tooling**:

| Actividad realizada                                    | Herramienta/Concepto           |
|--------------------------------------------------------|-------------------------------|
| Instalación y verificación de WDT 4.x                 | `$WDT_HOME/bin/`               |
| Empaquetado de aplicación Jakarta EE 10               | Archive File (`.zip`)          |
| Diseño de modelo YAML completo (4 secciones)          | `lab03-model.yaml`             |
| Parametrización con Variable Files                    | `lab03-dev.properties`         |
| Validación de modelo antes de aplicar                 | `validateModel.sh`             |
| Creación de dominio desde cero                        | `createDomain.sh`              |
| Actualización incremental del dominio                 | `updateDomain.sh`              |
| Descubrimiento de configuración existente             | `discoverDomain.sh`            |
| Buenas prácticas de versionado y seguridad            | Git, `.gitignore`, `chmod 600` |

### Conceptos clave aprendidos

- **Declarativo vs. imperativo:** WDT permite expresar el estado deseado del dominio sin scripting paso a paso.
- **Idempotencia:** Aplicar el mismo modelo varias veces converge al estado deseado sin efectos secundarios.
- **Separación de concerns:** El modelo YAML es agnóstico de entorno; los Variable Files contienen los valores específicos.
- **Ciclo de vida completo:** `validate → create → update → discover` forma un pipeline reproducible y auditable.
- **Seguridad:** Las credenciales nunca deben estar en el modelo YAML; se gestionan en Variable Files con permisos restrictivos o en vaults.

### Recursos adicionales

| Recurso                                                | URL                                                                      |
|--------------------------------------------------------|--------------------------------------------------------------------------|
| Repositorio oficial WDT en GitHub                     | https://github.com/oracle/weblogic-deploy-tooling                       |
| Guía de usuario oficial de WDT                        | https://oracle.github.io/weblogic-deploy-tooling/                       |
| Referencia del modelo YAML de WDT                     | https://oracle.github.io/weblogic-deploy-tooling/userguide/model/       |
| Matriz de compatibilidad WDT                          | https://oracle.github.io/weblogic-deploy-tooling/userguide/install/     |
| WebLogic Kubernetes Operator: Model in Image           | https://oracle.github.io/weblogic-kubernetes-operator/                  |
| Documentación WebLogic Server 14.1.2 (referencia base)| https://docs.oracle.com/en/middleware/fusion-middleware/weblogic-server/ |

> **Próximo laboratorio:** Lab 04-00-01 — Administración avanzada con WebLogic Remote Console (WRC) y REST API. Asegúrate de que el dominio `lab03Domain` y el Administration Server estén operativos antes de comenzar.

---
