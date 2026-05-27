# Instalación guiada de WebLogic 15 sobre Java 21. Buenas prácticas iniciales para instalación y soporte.

## 1. Metadatos

| Campo            | Detalle                                      |
|------------------|----------------------------------------------|
| **Duración**     | 120 minutos                                  |
| **Complejidad**  | Media                                        |
| **Nivel Bloom**  | Crear (*Create*)                             |
| **Módulo**       | 01 — Instalación y Arquitectura Base         |
| **Lab ID**       | 01-00-01                                     |

---

## 2. Descripción General

En este laboratorio instalarás Oracle WebLogic 15 desde cero sobre Oracle Linux 8/9 con JDK 21, aplicando las buenas prácticas de producción recomendadas por Oracle: usuario dedicado del sistema operativo, estructura de directorios canónica, permisos mínimos y separación de roles. Al finalizar habrás creado un dominio funcional, arrancado el Administration Server y verificado el acceso, consolidando los conceptos de la lección 1.1 sobre las novedades de WebLogic 15, la alineación con Jakarta EE 10 y el impacto del cambio de espacio de nombres `javax.*` → `jakarta.*`.

---

## 3. Objetivos de Aprendizaje

- [ ] Instalar Oracle WebLogic 15 usando el instalador genérico JAR en modo silencioso sobre JDK 21 en Oracle Linux.
- [ ] Configurar las variables de entorno del sistema (`JAVA_HOME`, `MW_HOME`, `WL_HOME`, `DOMAIN_HOME`) de forma persistente en `.bash_profile`.
- [ ] Aplicar las buenas prácticas de instalación de Oracle: usuario dedicado `oracle`, estructura de directorios, permisos y separación de roles.
- [ ] Crear un dominio básico con el Configuration Wizard y arrancar el Administration Server verificando su estado.
- [ ] Reconocer el impacto del cambio `javax.*` → `jakarta.*` analizando un ejemplo de código antes/después.

---

## 4. Prerequisitos

### Conocimiento Previo
- Comandos básicos de Linux: `mkdir`, `chmod`, `chown`, `export`, `vi`/`nano`, `tail`, `ps`, `netstat`/`ss`.
- Diferencia conceptual entre JDK y JRE; noción de variable de entorno y perfil de shell.
- Comprensión básica de servidores de aplicaciones Java EE/Jakarta EE (qué es un dominio, un Administration Server).

### Acceso y Recursos Necesarios
- Sistema Oracle Linux 8 o 9 instalado y actualizado (`dnf update` completado).
- Acceso `root` o `sudo` al sistema operativo.
- Archivo `jdk-21.0.3_linux-x64_bin.rpm` (o `.tar.gz`) disponible localmente o descargado desde [Oracle JDK 21](https://www.oracle.com/java/technologies/downloads/#java21).
- Instalador de WebLogic 15: `fmw_15.x.x_wls.jar` descargado en `/tmp` o directorio accesible. Requiere licencia Oracle Developer (OTN) para entornos de laboratorio.
- Acceso a internet (opcional) para descarga de dependencias adicionales.
- Al menos **20 GB** de espacio libre en el sistema de archivos destino.

> **⚠️ Nota sobre licencias:** Oracle WebLogic 15 requiere licencia comercial en producción. En este laboratorio se usa la Oracle Developer License (OTN). Verifica los términos antes de la instalación.

---

## 5. Entorno de Laboratorio

### Hardware Recomendado

| Recurso       | Mínimo          | Recomendado     |
|---------------|-----------------|-----------------|
| RAM           | 8 GB            | 16 GB           |
| CPU           | 4 núcleos       | 8 núcleos       |
| Disco         | 20 GB libres    | 50 GB libres    |
| Red           | Interfaz activa | Interfaz activa |

### Software Requerido

| Componente             | Versión              | Ubicación esperada           |
|------------------------|----------------------|------------------------------|
| Oracle Linux           | 8.x / 9.x (64 bits) | SO host o VM                 |
| Oracle JDK             | 21.0.3+              | `/usr/java/jdk-21/`          |
| WebLogic Installer     | 15.x.x (Generic JAR) | `/tmp/fmw_15.x.x_wls.jar`   |
| Oracle Inventory       | —                    | `/opt/oracle/oraInventory`   |
| Middleware Home        | —                    | `/opt/oracle/middleware`     |

### Verificación Rápida del Entorno (ejecutar como `root` o usuario con `sudo`)

```bash
# Verificar versión del sistema operativo
cat /etc/oracle-release || cat /etc/redhat-release

# Verificar espacio en disco disponible
df -h /opt

# Verificar conectividad básica
ping -c 2 8.8.8.8
```

---

## 6. Procedimiento Paso a Paso

---

### Paso 1: Instalar JDK 21 y Verificar la Plataforma

**Objetivo:** Asegurar que JDK 21 está correctamente instalado y que el sistema cumple los requisitos de plataforma de WebLogic 15.

#### Instrucciones

1. Inicia sesión como `root` o usuario con privilegios `sudo`.

2. Instala JDK 21 usando el paquete RPM:

```bash
# Opción A: Instalación desde RPM (recomendado en Oracle Linux)
sudo rpm -ivh /tmp/jdk-21.0.3_linux-x64_bin.rpm

# Opción B: Si usas tar.gz
sudo mkdir -p /usr/java
sudo tar -xzf /tmp/jdk-21.0.3_linux-x64_bin.tar.gz -C /usr/java/
```

3. Verifica la instalación y la versión:

```bash
/usr/java/jdk-21.0.3/bin/java -version
```

4. Verifica que el JDK incluye el compilador (requisito para WebLogic):

```bash
/usr/java/jdk-21.0.3/bin/javac -version
```

5. Confirma que el sistema operativo es de 64 bits:

```bash
uname -m
# Debe devolver: x86_64
```

6. Instala dependencias de sistema requeridas por el instalador de WebLogic:

```bash
sudo dnf install -y libaio glibc-devel libstdc++ unzip
```

#### Salida Esperada

```
openjdk version "21.0.3" 2024-04-16 LTS
OpenJDK Runtime Environment (build 21.0.3+...)
OpenJDK 64-Bit Server VM (build 21.0.3+..., mixed mode, sharing)

javac 21.0.3

x86_64
```

#### Verificación

```bash
# Confirmar que java está disponible en la ruta correcta
ls -la /usr/java/jdk-21.0.3/bin/java
# Confirmar que es una JVM de 64 bits con soporte de servidor
/usr/java/jdk-21.0.3/bin/java -XX:+PrintFlagsFinal -version 2>&1 | grep -i "UseCompressedOops"
```

---

### Paso 2: Crear el Usuario y Estructura de Directorios Dedicados

**Objetivo:** Aplicar la buena práctica de Oracle de ejecutar WebLogic bajo un usuario del sistema operativo dedicado (`oracle`) sin privilegios de `root`, con una estructura de directorios organizada.

#### Instrucciones

1. Crea el grupo y usuario del sistema operativo `oracle`:

```bash
sudo groupadd -g 1100 oinstall
sudo groupadd -g 1101 dba
sudo useradd -u 1100 -g oinstall -G dba -m -s /bin/bash -d /home/oracle oracle
sudo passwd oracle
# Introduce la contraseña cuando se solicite: Oracle2024# (solo para laboratorio)
```

2. Crea la estructura de directorios de Oracle:

```bash
# Oracle Inventory (registro de instalaciones)
sudo mkdir -p /opt/oracle/oraInventory

# Middleware Home (instalación de WebLogic)
sudo mkdir -p /opt/oracle/middleware

# Directorio de dominios (separado del Middleware Home — buena práctica)
sudo mkdir -p /opt/oracle/domains

# Directorio de logs centralizado
sudo mkdir -p /opt/oracle/logs

# Directorio de scripts de administración
sudo mkdir -p /opt/oracle/scripts
```

3. Asigna la propiedad de los directorios al usuario `oracle`:

```bash
sudo chown -R oracle:oinstall /opt/oracle
sudo chmod -R 750 /opt/oracle
```

4. Crea el archivo `oraInst.loc` que indica al instalador dónde registrar el inventario:

```bash
sudo bash -c 'cat > /etc/oraInst.loc << EOF
inventory_loc=/opt/oracle/oraInventory
inst_group=oinstall
EOF'
sudo chmod 644 /etc/oraInst.loc
```

5. Verifica la estructura creada:

```bash
ls -la /opt/oracle/
```

#### Salida Esperada

```
drwxr-x--- 2 oracle oinstall 4096 ... domains
drwxr-x--- 2 oracle oinstall 4096 ... logs
drwxr-x--- 2 oracle oinstall 4096 ... middleware
drwxr-x--- 2 oracle oinstall 4096 ... oraInventory
drwxr-x--- 2 oracle oinstall 4096 ... scripts
```

#### Verificación

```bash
# Verificar que oracle puede escribir en los directorios
sudo -u oracle touch /opt/oracle/middleware/.test && echo "OK: escritura permitida" && sudo -u oracle rm /opt/oracle/middleware/.test
```

---

### Paso 3: Configurar Variables de Entorno Persistentes

**Objetivo:** Definir las variables de entorno estándar de WebLogic en el perfil del usuario `oracle` para que estén disponibles en cada sesión.

#### Instrucciones

1. Cambia al usuario `oracle`:

```bash
sudo su - oracle
```

2. Edita el archivo `.bash_profile` del usuario `oracle`:

```bash
vi ~/.bash_profile
```

3. Añade el siguiente bloque al final del archivo (después de las líneas existentes):

```bash
# ============================================================
# Configuración WebLogic 15 + JDK 21 — Buenas prácticas Oracle
# ============================================================

# Java Home — apunta al JDK 21 instalado
export JAVA_HOME=/usr/java/jdk-21.0.3

# Middleware Home — directorio raíz de la instalación WebLogic
export MW_HOME=/opt/oracle/middleware

# WebLogic Home — directorio de binarios del servidor
export WL_HOME=${MW_HOME}/wlserver

# Domain Home — directorio del dominio activo (se actualizará por dominio)
export DOMAIN_HOME=/opt/oracle/domains/base_domain

# Oracle Home (alias de MW_HOME, requerido por algunos scripts)
export ORACLE_HOME=${MW_HOME}

# PATH — incluye JDK y scripts WebLogic
export PATH=${JAVA_HOME}/bin:${MW_HOME}/oracle_common/common/bin:${PATH}

# Opciones de JVM para el Administration Server (baseline de laboratorio)
# G1GC como recomendado para WebLogic 15 en Java 21
export USER_MEM_ARGS="-Xms512m -Xmx1024m -XX:+UseG1GC -XX:MaxGCPauseMillis=200"

# Evitar que WebLogic busque un display gráfico para el Configuration Wizard
export CONFIG_JVM_ARGS="-Djava.awt.headless=true"
```

4. Guarda y cierra el archivo (`:wq` en vi). Aplica los cambios:

```bash
source ~/.bash_profile
```

5. Verifica que las variables están definidas correctamente:

```bash
echo "JAVA_HOME  = $JAVA_HOME"
echo "MW_HOME    = $MW_HOME"
echo "WL_HOME    = $WL_HOME"
echo "DOMAIN_HOME= $DOMAIN_HOME"
echo "ORACLE_HOME= $ORACLE_HOME"
java -version
```

#### Salida Esperada

```
JAVA_HOME  = /usr/java/jdk-21.0.3
MW_HOME    = /opt/oracle/middleware
WL_HOME    = /opt/oracle/middleware/wlserver
DOMAIN_HOME= /opt/oracle/domains/base_domain
ORACLE_HOME= /opt/oracle/middleware
openjdk version "21.0.3" ...
```

#### Verificación

```bash
# Confirmar que el PATH incluye el JDK correcto
which java
# Debe devolver: /usr/java/jdk-21.0.3/bin/java
```

---

### Paso 4: Instalar WebLogic 15 en Modo Silencioso

**Objetivo:** Ejecutar el instalador JAR de WebLogic 15 en modo silencioso (*silent mode*) usando un archivo de respuestas, siguiendo la práctica recomendada para entornos reproducibles y automatizables.

#### Instrucciones

1. Como usuario `oracle`, copia el instalador al directorio de trabajo:

```bash
cp /tmp/fmw_15.x.x_wls.jar /home/oracle/
```

2. Crea el archivo de respuestas para la instalación silenciosa:

```bash
cat > /home/oracle/wls_install_response.rsp << 'EOF'
[ENGINE]
Response File Version=1.0.0.0.0

[GENERIC]
# Directorio de instalación (Middleware Home)
ORACLE_HOME=/opt/oracle/middleware

# Tipo de instalación: WLS — solo WebLogic Server (sin FMW completo)
INSTALL_TYPE=WebLogic Server

# Aceptación de licencia (MUST be true)
DECLINE_SECURITY_UPDATES=true
SECURITY_UPDATES_VIA_MYORACLESUPPORT=false
EOF
```

3. Ejecuta la instalación silenciosa:

```bash
java -Xmx1024m -jar /home/oracle/fmw_15.x.x_wls.jar \
     -silent \
     -responseFile /home/oracle/wls_install_response.rsp \
     -invPtrLoc /etc/oraInst.loc \
     -jreLoc ${JAVA_HOME} \
     -logLevel INFO \
     -logDir /opt/oracle/logs
```

> **Nota:** La instalación puede tardar entre 5 y 15 minutos dependiendo del hardware. El instalador no muestra barra de progreso en modo silencioso; monitoriza el log para seguimiento.

4. Monitoriza el progreso en otra terminal (abre una segunda sesión como `oracle`):

```bash
# En otra terminal, sigue el log de instalación
tail -f /opt/oracle/logs/install*.log
```

5. Una vez finalizada la instalación, verifica los directorios creados:

```bash
ls -la /opt/oracle/middleware/
```

#### Salida Esperada (estructura de directorios post-instalación)

```
drwxr-x--- oracle oinstall ... coherence/
drwxr-x--- oracle oinstall ... oracle_common/
drwxr-x--- oracle oinstall ... oui/
drwxr-x--- oracle oinstall ... wlserver/
-rw-r----- oracle oinstall ... domain-registry.xml
```

#### Verificación

```bash
# Verificar la versión instalada de WebLogic
java -cp ${WL_HOME}/server/lib/weblogic.jar weblogic.version

# Verificar que el inventario registró la instalación
cat /opt/oracle/oraInventory/ContentsXML/inventory.xml | grep -i weblogic
```

**Salida esperada de `weblogic.version`:**
```
WebLogic Server 15.x.x.x ... Oracle WebLogic Server Module Dependencies 15.x ...
```

---

### Paso 5: Crear el Dominio Base con el Configuration Wizard

**Objetivo:** Crear un dominio funcional de WebLogic usando el Configuration Wizard en modo de línea de comandos (headless), aplicando configuración mínima para un Administration Server.

#### Instrucciones

1. Verifica que el Configuration Wizard está disponible:

```bash
ls -la ${MW_HOME}/oracle_common/common/bin/config.sh
```

2. Crea el archivo de respuestas para el Configuration Wizard (modo silencioso):

```bash
cat > /home/oracle/domain_create.py << 'EOF'
# Script WLST para crear un dominio base — Laboratorio 01-00-01
# WebLogic 15 sobre Java 21

import os

# ---- Parámetros del dominio ----
domain_name     = 'base_domain'
admin_name      = 'AdminServer'
admin_port      = 7001
admin_ssl_port  = 7002
admin_user      = 'weblogic'
admin_password  = 'Welcome1'  # SOLO para laboratorio — nunca usar en producción
domain_path     = '/opt/oracle/domains/' + domain_name
template_path   = os.environ.get('WL_HOME') + '/common/templates/wls/wls.jar'

print('==> Seleccionando template de dominio: ' + template_path)
selectTemplate(template_path)
loadTemplates()

print('==> Configurando credenciales del dominio...')
cd('Servers/AdminServer')
set('ListenAddress', '0.0.0.0')
set('ListenPort', admin_port)

# Habilitar SSL
create('AdminServer', 'SSL')
cd('SSL/AdminServer')
set('Enabled', 'True')
set('ListenPort', admin_ssl_port)

cd('/')

print('==> Configurando usuario administrador...')
cd('Security/base_domain/User/' + admin_user)
cmo.setPassword(admin_password)

print('==> Asignando nombre al dominio...')
setOption('DomainName', domain_name)
setOption('JavaHome', os.environ.get('JAVA_HOME'))
setOption('ServerStartMode', 'prod')

print('==> Escribiendo dominio en: ' + domain_path)
writeDomain(domain_path)
closeTemplate()

print('==> Dominio creado exitosamente.')
EOF
```

3. Ejecuta el script WLST para crear el dominio:

```bash
${MW_HOME}/oracle_common/common/bin/wlst.sh /home/oracle/domain_create.py
```

4. Verifica la estructura del dominio creado:

```bash
ls -la /opt/oracle/domains/base_domain/
```

5. Crea el archivo de boot credentials para arranque desatendido (buena práctica):

```bash
mkdir -p /opt/oracle/domains/base_domain/servers/AdminServer/security
cat > /opt/oracle/domains/base_domain/servers/AdminServer/security/boot.properties << 'EOF'
username=weblogic
password=Welcome1
EOF
chmod 600 /opt/oracle/domains/base_domain/servers/AdminServer/security/boot.properties
```

#### Salida Esperada

```
==> Seleccionando template de dominio: /opt/oracle/middleware/wlserver/common/templates/wls/wls.jar
==> Configurando credenciales del dominio...
==> Configurando usuario administrador...
==> Asignando nombre al dominio...
==> Escribiendo dominio en: /opt/oracle/domains/base_domain
==> Dominio creado exitosamente.
```

#### Verificación

```bash
# Verificar que los archivos de configuración del dominio existen
ls /opt/oracle/domains/base_domain/config/config.xml

# Verificar que el nombre del dominio está en config.xml
grep -i "domain-name\|<name>" /opt/oracle/domains/base_domain/config/config.xml | head -5
```

---

### Paso 6: Arrancar el Administration Server y Verificar el Acceso

**Objetivo:** Iniciar el Administration Server y confirmar que WebLogic 15 está operativo, verificando el estado mediante comandos de línea y la API REST de gestión.

#### Instrucciones

1. Actualiza la variable `DOMAIN_HOME` para apuntar al dominio recién creado:

```bash
export DOMAIN_HOME=/opt/oracle/domains/base_domain
echo "DOMAIN_HOME actualizado: $DOMAIN_HOME"
```

2. Inicia el Administration Server en segundo plano:

```bash
nohup ${DOMAIN_HOME}/startWebLogic.sh > /opt/oracle/logs/adminserver_start.log 2>&1 &
echo "PID del proceso de arranque: $!"
```

3. Monitoriza el arranque hasta que aparezca el mensaje `RUNNING`:

```bash
# Espera activa — el arranque puede tardar 60-120 segundos
tail -f /opt/oracle/logs/adminserver_start.log | grep -E "RUNNING|FAILED|Exception|Started"
```

> **Presiona `Ctrl+C`** una vez que veas el mensaje `<Server state changed to: RUNNING>`.

4. Verifica que el proceso de Java está activo:

```bash
ps -ef | grep weblogic | grep -v grep
```

5. Verifica que el puerto 7001 está escuchando:

```bash
ss -tlnp | grep 7001
```

6. Verifica la versión de WebLogic en ejecución usando la REST Management API:

```bash
curl -s -u weblogic:Welcome1 \
  http://localhost:7001/management/weblogic/latest/serverRuntime?links=none \
  | python3 -m json.tool | grep -E "weblogicVersion|name|state"
```

7. Verifica el acceso a la Consola de Administración (si hay entorno gráfico o acceso remoto):

```bash
# Confirma que la consola responde (código HTTP 200 o 302)
curl -s -o /dev/null -w "%{http_code}" http://localhost:7001/console
```

#### Salida Esperada

```
# Del log de arranque:
<Server state changed to: RUNNING>
<Server started in RUNNING mode>

# Del proceso:
oracle   12345  ... java ... weblogic.Server

# Del puerto:
LISTEN 0 128 0.0.0.0:7001 ...

# De la REST API:
"weblogicVersion" : "WebLogic Server 15.x.x.x ...",
"name" : "AdminServer",
"state" : "RUNNING"

# Del curl a la consola:
200
```

#### Verificación

```bash
# Verificación completa de estado mediante REST API
curl -s -u weblogic:Welcome1 \
  "http://localhost:7001/management/weblogic/latest/domainRuntime/serverRuntimes/AdminServer?links=none" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print('Estado:', d.get('state','N/A')); print('Versión:', d.get('weblogicVersion','N/A'))"
```

---

### Paso 7: Verificar las Novedades de WebLogic 15 — Análisis javax.* vs jakarta.*

**Objetivo:** Consolidar el conocimiento teórico de la lección 1.1 analizando el impacto práctico del cambio de namespace `javax.*` → `jakarta.*` en código de aplicación real.

#### Instrucciones

1. Crea un directorio de análisis:

```bash
mkdir -p /home/oracle/lab01/jakarta_migration
cd /home/oracle/lab01/jakarta_migration
```

2. Crea el archivo de ejemplo con código **antes** de la migración (Java EE / `javax.*`):

```bash
cat > ServletJavaEE.java << 'EOF'
// ANTES: Aplicación Java EE con namespace javax.*
// Este código NO compila ni funciona en WebLogic 15 + Jakarta EE 10

import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.persistence.EntityManager;
import javax.persistence.PersistenceContext;
import javax.inject.Inject;
import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import javax.ws.rs.core.MediaType;
import java.io.IOException;

@WebServlet("/legacy")
public class ServletJavaEE extends HttpServlet {

    @PersistenceContext
    private EntityManager em;  // javax.persistence — INCOMPATIBLE con Jakarta EE 10

    @Inject
    private MiServicio servicio;  // javax.inject — INCOMPATIBLE

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {
        resp.getWriter().println("Hello from Java EE (javax.*)");
    }
}
EOF
```

3. Crea el archivo de ejemplo **después** de la migración (Jakarta EE 10 / `jakarta.*`):

```bash
cat > ServletJakartaEE.java << 'EOF'
// DESPUÉS: Aplicación Jakarta EE 10 con namespace jakarta.*
// Compatible con WebLogic 15 + Jakarta EE 10

import jakarta.servlet.ServletException;
import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import java.io.IOException;

@WebServlet("/modern")
public class ServletJakartaEE extends HttpServlet {

    @PersistenceContext
    private EntityManager em;  // jakarta.persistence — COMPATIBLE con Jakarta EE 10

    @Inject
    private MiServicio servicio;  // jakarta.inject — COMPATIBLE

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {
        resp.getWriter().println("Hello from Jakarta EE 10 (jakarta.*)");
    }
}
EOF
```

4. Genera un reporte de diferencias para visualizar el cambio de namespace:

```bash
diff ServletJavaEE.java ServletJakartaEE.java
```

5. Crea un script de búsqueda rápida para detectar usos de `javax.*` en proyectos existentes (simulación de auditoría):

```bash
cat > /home/oracle/scripts/audit_javax.sh << 'EOF'
#!/bin/bash
# audit_javax.sh — Detecta importaciones javax.* en proyectos Java
# Uso: ./audit_javax.sh <directorio_proyecto>

TARGET_DIR="${1:-.}"
echo "======================================================"
echo " Auditoría de namespace javax.* en: ${TARGET_DIR}"
echo " Fecha: $(date)"
echo "======================================================"

echo ""
echo "--- Archivos .java con importaciones javax.* ---"
grep -rn "^import javax\." "${TARGET_DIR}" --include="*.java" 2>/dev/null | \
    awk -F: '{print $1}' | sort -u | \
    while read f; do
        count=$(grep -c "^import javax\." "$f")
        echo "  [${count} imports] $f"
    done

echo ""
echo "--- Archivos de configuración con referencias javax.* ---"
grep -rn "javax\." "${TARGET_DIR}" --include="*.xml" --include="*.properties" 2>/dev/null | \
    grep -v ".class" | head -20

echo ""
echo "--- Resumen ---"
total_java=$(grep -rl "^import javax\." "${TARGET_DIR}" --include="*.java" 2>/dev/null | wc -l)
echo "  Total archivos .java que requieren migración: ${total_java}"
echo "======================================================"
EOF
chmod +x /home/oracle/scripts/audit_javax.sh
```

6. Ejecuta la auditoría sobre el directorio de ejemplos:

```bash
/home/oracle/scripts/audit_javax.sh /home/oracle/lab01/jakarta_migration
```

7. Documenta las diferencias clave observadas:

```bash
cat > /home/oracle/lab01/jakarta_migration/NOTAS_MIGRACION.md << 'EOF'
# Notas de Migración javax.* → jakarta.*

## Cambios Principales Identificados

| Namespace Antiguo (Java EE) | Namespace Nuevo (Jakarta EE 10) | Componente    |
|-----------------------------|---------------------------------|---------------|
| javax.servlet.*             | jakarta.servlet.*               | Servlet API   |
| javax.persistence.*         | jakarta.persistence.*           | JPA 3.1       |
| javax.inject.*              | jakarta.inject.*                | CDI 4.0       |
| javax.ws.rs.*               | jakarta.ws.rs.*                 | JAX-RS 3.1    |
| javax.ejb.*                 | jakarta.ejb.*                   | EJB           |
| javax.transaction.*         | jakarta.transaction.*           | JTA           |
| javax.validation.*          | jakarta.validation.*            | Bean Validation|

## Herramientas de Migración Disponibles
- Eclipse Transformer: renombrado automático durante el build
- OpenRewrite: refactorización a nivel de código fuente
- WDT (WebLogic Deploy Tooling): para configuración de dominios

## Impacto en WebLogic 15
- NO hay compatibilidad binaria automática entre javax.* y jakarta.*
- Las aplicaciones deben ser recompiladas con las nuevas dependencias
- Los descriptores de despliegue (web.xml, ejb-jar.xml) también requieren actualización
EOF
```

#### Salida Esperada del `diff`

```diff
< // ANTES: Aplicación Java EE con namespace javax.*
< // Este código NO compila ni funciona en WebLogic 15 + Jakarta EE 10
---
> // DESPUÉS: Aplicación Jakarta EE 10 con namespace jakarta.*
> Compatible con WebLogic 15 + Jakarta EE 10
5c5
< import javax.servlet.ServletException;
---
> import jakarta.servlet.ServletException;
... (una línea por cada import cambiado)
```

#### Verificación

```bash
# Confirmar que los archivos de análisis fueron creados
ls -la /home/oracle/lab01/jakarta_migration/
# Confirmar que el script de auditoría es ejecutable
ls -la /home/oracle/scripts/audit_javax.sh
```

---

### Paso 8: Verificar la Versión y Configuración Final de WebLogic 15

**Objetivo:** Realizar una verificación completa de la instalación, confirmando la versión de WebLogic, la configuración de TLS y las variables de entorno, tal como se haría en un entorno de producción.

#### Instrucciones

1. Verifica la versión de WebLogic desde la línea de comandos:

```bash
java -cp ${WL_HOME}/server/lib/weblogic.jar weblogic.version
```

2. Verifica la información completa del dominio mediante REST API:

```bash
curl -s -u weblogic:Welcome1 \
  "http://localhost:7001/management/weblogic/latest/domainRuntime?links=none&fields=name,state" \
  | python3 -m json.tool
```

3. Verifica las variables de entorno configuradas en `.bash_profile`:

```bash
# Cierra y reabre la sesión del usuario oracle para probar persistencia
# En la sesión actual, simula recargando el perfil:
source ~/.bash_profile
env | grep -E "JAVA_HOME|MW_HOME|WL_HOME|DOMAIN_HOME|ORACLE_HOME" | sort
```

4. Genera un resumen del entorno instalado:

```bash
cat > /home/oracle/lab01/RESUMEN_INSTALACION.txt << EOF
========================================
 RESUMEN DE INSTALACIÓN — LAB 01-00-01
 Fecha: $(date)
========================================

Sistema Operativo:
$(cat /etc/oracle-release 2>/dev/null || cat /etc/redhat-release)

Java:
$(java -version 2>&1)

WebLogic:
$(java -cp ${WL_HOME}/server/lib/weblogic.jar weblogic.version 2>/dev/null | head -3)

Variables de Entorno:
  JAVA_HOME   = ${JAVA_HOME}
  MW_HOME     = ${MW_HOME}
  WL_HOME     = ${WL_HOME}
  DOMAIN_HOME = ${DOMAIN_HOME}

Dominio:
  Nombre      = base_domain
  Admin Server= AdminServer
  Puerto HTTP = 7001
  Puerto HTTPS= 7002

Estado del servidor:
$(curl -s -u weblogic:Welcome1 "http://localhost:7001/management/weblogic/latest/serverRuntime?links=none&fields=state,name" 2>/dev/null | python3 -c "import sys,json; d=json.load(sys.stdin); print('  Estado:', d.get('state','N/A'))" 2>/dev/null)

========================================
EOF

cat /home/oracle/lab01/RESUMEN_INSTALACION.txt
```

#### Salida Esperada

```
========================================
 RESUMEN DE INSTALACIÓN — LAB 01-00-01
========================================
Sistema Operativo: Oracle Linux Server release 8.x
Java: openjdk version "21.0.3" ...
WebLogic: WebLogic Server 15.x.x.x ...
Variables de Entorno:
  JAVA_HOME   = /usr/java/jdk-21.0.3
  MW_HOME     = /opt/oracle/middleware
  ...
Estado del servidor:
  Estado: RUNNING
========================================
```

#### Verificación

```bash
# Verificación final completa
echo "=== VERIFICACIÓN FINAL ==="
echo -n "1. JDK 21 instalado: "
java -version 2>&1 | grep -q "21" && echo "OK" || echo "FALLO"

echo -n "2. Middleware Home existe: "
[ -d "${MW_HOME}/wlserver" ] && echo "OK" || echo "FALLO"

echo -n "3. Dominio creado: "
[ -f "${DOMAIN_HOME}/config/config.xml" ] && echo "OK" || echo "FALLO"

echo -n "4. Admin Server RUNNING: "
curl -s -u weblogic:Welcome1 "http://localhost:7001/management/weblogic/latest/serverRuntime?links=none&fields=state" 2>/dev/null | grep -q "RUNNING" && echo "OK" || echo "FALLO"

echo -n "5. Variables de entorno persistentes: "
grep -q "MW_HOME" ~/.bash_profile && echo "OK" || echo "FALLO"
```

---

## 7. Validación y Pruebas

Ejecuta la siguiente secuencia de validación completa para confirmar que el laboratorio está correctamente completado:

```bash
#!/bin/bash
# validate_lab01.sh — Script de validación completa del Lab 01-00-01

echo "=============================================="
echo " VALIDACIÓN LAB 01-00-01"
echo " WebLogic 15 + JDK 21 — Instalación Base"
echo "=============================================="

PASS=0
FAIL=0

check() {
    local desc="$1"
    local cmd="$2"
    if eval "$cmd" > /dev/null 2>&1; then
        echo "  [PASS] $desc"
        ((PASS++))
    else
        echo "  [FAIL] $desc"
        ((FAIL++))
    fi
}

# Sección 1: JDK
echo ""
echo "--- JDK 21 ---"
check "JDK 21 instalado en JAVA_HOME" "[ -x '${JAVA_HOME}/bin/java' ]"
check "Versión Java es 21" "java -version 2>&1 | grep -q '21'"
check "javac disponible (JDK completo, no solo JRE)" "[ -x '${JAVA_HOME}/bin/javac' ]"

# Sección 2: Usuario y permisos
echo ""
echo "--- Usuario oracle y estructura de directorios ---"
check "Usuario oracle existe" "id oracle"
check "Directorio MW_HOME existe" "[ -d '/opt/oracle/middleware' ]"
check "Directorio DOMAIN_HOME existe" "[ -d '/opt/oracle/domains' ]"
check "oraInst.loc configurado" "[ -f '/etc/oraInst.loc' ]"

# Sección 3: Instalación WebLogic
echo ""
echo "--- Instalación WebLogic 15 ---"
check "WL_HOME/server/lib/weblogic.jar existe" "[ -f '${WL_HOME}/server/lib/weblogic.jar' ]"
check "Configuration Wizard disponible" "[ -x '${MW_HOME}/oracle_common/common/bin/config.sh' ]"
check "WLST disponible" "[ -x '${MW_HOME}/oracle_common/common/bin/wlst.sh' ]"

# Sección 4: Dominio
echo ""
echo "--- Dominio base_domain ---"
check "config.xml existe" "[ -f '/opt/oracle/domains/base_domain/config/config.xml' ]"
check "startWebLogic.sh existe" "[ -f '/opt/oracle/domains/base_domain/startWebLogic.sh' ]"
check "boot.properties configurado" "[ -f '/opt/oracle/domains/base_domain/servers/AdminServer/security/boot.properties' ]"

# Sección 5: Servidor en ejecución
echo ""
echo "--- Administration Server ---"
check "Puerto 7001 escuchando" "ss -tlnp | grep -q ':7001'"
check "Admin Server RUNNING (REST API)" "curl -s -u weblogic:Welcome1 'http://localhost:7001/management/weblogic/latest/serverRuntime?links=none&fields=state' | grep -q 'RUNNING'"

# Sección 6: Variables de entorno
echo ""
echo "--- Variables de entorno ---"
check "JAVA_HOME en .bash_profile" "grep -q 'JAVA_HOME' ~/.bash_profile"
check "MW_HOME en .bash_profile" "grep -q 'MW_HOME' ~/.bash_profile"
check "DOMAIN_HOME en .bash_profile" "grep -q 'DOMAIN_HOME' ~/.bash_profile"

# Resumen
echo ""
echo "=============================================="
echo " RESULTADO: ${PASS} PASS | ${FAIL} FAIL"
[ ${FAIL} -eq 0 ] && echo " ✓ Laboratorio completado exitosamente." || echo " ✗ Revisar los puntos fallidos."
echo "=============================================="
```

```bash
# Guarda y ejecuta el script de validación
chmod +x /home/oracle/scripts/validate_lab01.sh
/home/oracle/scripts/validate_lab01.sh
```

**Criterio de éxito:** Todos los checks deben mostrar `[PASS]`. Un mínimo de 15/16 checks en PASS es aceptable si el fallo es no crítico (por ejemplo, acceso gráfico a la consola no disponible).

---

## 8. Resolución de Problemas

### Problema 1: El instalador JAR falla con "java.lang.UnsupportedClassVersionError" o el proceso termina sin completarse

**Síntomas:**
- El comando `java -jar fmw_15.x.x_wls.jar` termina inmediatamente sin instalar.
- El log de instalación muestra `UnsupportedClassVersionError` o `Unsupported major.minor version`.
- Posible mensaje: `This installer requires Java version 11 or higher`.

**Causa:**
El instalador está siendo ejecutado con un JDK incorrecto. El `java` en el `PATH` apunta a una versión anterior (Java 8 o 11 del sistema) en lugar del JDK 21 configurado en `JAVA_HOME`, o la variable `JAVA_HOME` no está exportada correctamente en la sesión actual.

**Solución:**

```bash
# 1. Verificar qué java está usando el shell actual
which java
java -version

# 2. Si apunta a una versión incorrecta, forzar el uso del JDK 21
export JAVA_HOME=/usr/java/jdk-21.0.3
export PATH=${JAVA_HOME}/bin:${PATH}

# 3. Confirmar que ahora usa el correcto
which java
java -version  # Debe mostrar 21.x.x

# 4. Verificar si hay alternativas de sistema configuradas que interfieran
update-alternatives --display java

# 5. Volver a ejecutar el instalador especificando explícitamente el JDK
java -jreLoc /usr/java/jdk-21.0.3 -Xmx1024m -jar /home/oracle/fmw_15.x.x_wls.jar \
     -silent -responseFile /home/oracle/wls_install_response.rsp \
     -invPtrLoc /etc/oraInst.loc

# 6. Recargar .bash_profile para asegurar persistencia
source ~/.bash_profile
```

---

### Problema 2: El Administration Server no arranca — Error "Address already in use" o estado FAILED_NOT_RESTARTABLE

**Síntomas:**
- El log `adminserver_start.log` muestra: `java.net.BindException: Address already in use`.
- El servidor entra en estado `FAILED_NOT_RESTARTABLE` en lugar de `RUNNING`.
- `ss -tlnp | grep 7001` muestra que el puerto ya está ocupado.

**Causa:**
El puerto 7001 (o 7002) está siendo utilizado por otro proceso, posiblemente una instancia anterior de WebLogic que no fue detenida correctamente, o un proceso del sistema que ocupa ese puerto. También puede ocurrir si el dominio fue creado con una dirección de escucha incorrecta.

**Solución:**

```bash
# 1. Identificar qué proceso ocupa el puerto 7001
ss -tlnp | grep ':7001'
# O con lsof si está disponible:
sudo lsof -i :7001

# 2. Si es una instancia anterior de WebLogic, terminarla limpiamente
# Primero intenta parada controlada:
${DOMAIN_HOME}/bin/stopWebLogic.sh weblogic Welcome1 t3://localhost:7001

# Si no responde, identifica el PID y termina el proceso:
PID=$(ps -ef | grep "weblogic.Server" | grep -v grep | awk '{print $2}')
echo "PID WebLogic: $PID"
kill -15 $PID  # SIGTERM — parada limpia
sleep 10
# Si persiste:
kill -9 $PID   # SIGKILL — forzado

# 3. Limpiar archivos de bloqueo del dominio
rm -f ${DOMAIN_HOME}/.wlnotdelete
rm -f ${DOMAIN_HOME}/servers/AdminServer/tmp/*.lck 2>/dev/null

# 4. Verificar que el puerto está libre
ss -tlnp | grep ':7001'
# No debe devolver ninguna línea

# 5. Si otro servicio del sistema ocupa el puerto, cambiar el puerto del Admin Server
# Edita config.xml y cambia ListenPort:
sed -i 's/<listen-port>7001<\/listen-port>/<listen-port>7101<\/listen-port>/' \
    ${DOMAIN_HOME}/config/config.xml

# 6. Reiniciar el Administration Server
nohup ${DOMAIN_HOME}/startWebLogic.sh > /opt/oracle/logs/adminserver_start.log 2>&1 &
tail -f /opt/oracle/logs/adminserver_start.log | grep -E "RUNNING|FAILED|Exception"
```

---

## 9. Limpieza del Laboratorio

> **⚠️ Importante:** Ejecuta la limpieza **únicamente** si deseas deshacer completamente la instalación o preparar el entorno para una reinstalación. Los laboratorios posteriores (02 en adelante) dependen de esta instalación. **Se recomienda tomar un snapshot de la VM antes de continuar con otros laboratorios.**

```bash
# ---- LIMPIEZA OPCIONAL — Solo si se requiere reinstalación ----
# Ejecutar como usuario oracle o root según corresponda

# 1. Detener el Administration Server (si está en ejecución)
${DOMAIN_HOME}/bin/stopWebLogic.sh weblogic Welcome1 t3://localhost:7001 2>/dev/null || \
    kill -15 $(ps -ef | grep "weblogic.Server" | grep -v grep | awk '{print $2}') 2>/dev/null
sleep 5

# 2. Eliminar el dominio (si se quiere recrear)
# rm -rf /opt/oracle/domains/base_domain

# 3. Eliminar la instalación de WebLogic (desinstalación completa)
# PRECAUCIÓN: esto elimina todos los archivos de WebLogic
# java -jar ${MW_HOME}/oui/lib/OraInstaller.jar -deinstall \
#      -invPtrLoc /etc/oraInst.loc -jreLoc ${JAVA_HOME} -silent \
#      -responseFile /home/oracle/wls_deinstall.rsp

# 4. Limpiar logs de laboratorio (siempre seguro)
rm -f /opt/oracle/logs/adminserver_start.log
rm -f /opt/oracle/logs/install*.log

# 5. Limpiar archivos temporales del laboratorio
rm -f /home/oracle/fmw_15.x.x_wls.jar
rm -f /home/oracle/wls_install_response.rsp
rm -f /home/oracle/domain_create.py

echo "Limpieza completada."
```

**Acción recomendada post-laboratorio:**
```bash
# Tomar un snapshot de la VM con el nombre: "Lab01-Completado"
# Esto permite volver a este estado en cualquier momento
# VMware: VM > Snapshot > Take Snapshot
# VirtualBox: Machine > Take Snapshot
```

---

## 10. Resumen

En este laboratorio has completado la instalación completa y verificada de Oracle WebLogic 15 sobre JDK 21 en Oracle Linux, siguiendo las buenas prácticas de producción recomendadas por Oracle. Los logros principales son:

| Tarea Completada | Resultado |
|-----------------|-----------|
| JDK 21 instalado y verificado | `/usr/java/jdk-21.0.3` operativo |
| Usuario `oracle` y estructura de directorios | Permisos mínimos aplicados |
| Variables de entorno persistentes | `.bash_profile` configurado |
| WebLogic 15 instalado (modo silencioso) | `MW_HOME=/opt/oracle/middleware` |
| Dominio `base_domain` creado | Administration Server en puerto 7001/7002 |
| Admin Server en estado `RUNNING` | Verificado via REST API |
| Análisis javax.* → jakarta.* | Script de auditoría y ejemplos documentados |

### Conceptos Clave Reforzados

- **WebLogic 15** alinea tres vectores: Java 21 LTS, Jakarta EE 10 y operación cloud-native (WDT/WRC/Operator).
- El cambio `javax.*` → `jakarta.*` **no es automático**: requiere refactorización de código, actualización de dependencias y pruebas de regresión.
- Las variables `JAVA_HOME`, `MW_HOME`, `WL_HOME` y `DOMAIN_HOME` son el punto de partida de toda administración de WebLogic.
- La separación de roles (usuario `oracle` sin `root`) y la estructura de directorios organizada son prácticas de seguridad fundamentales.
- El modo silencioso de instalación y los scripts WLST son la base de la automatización que se profundizará con WDT en laboratorios posteriores.

### Recursos Adicionales

| Recurso | URL |
|---------|-----|
| Documentación WebLogic Server 15 | https://docs.oracle.com/en/middleware/fusion-middleware/weblogic-server/ |
| Oracle JDK 21 Downloads | https://www.oracle.com/java/technologies/downloads/#java21 |
| WebLogic Deploy Tooling (WDT) | https://github.com/oracle/weblogic-deploy-tooling |
| Jakarta EE 10 Specifications | https://jakarta.ee/release/10/ |
| Eclipse Transformer (migración javax→jakarta) | https://github.com/eclipse/transformer |
| Oracle Lifetime Support Policy | https://www.oracle.com/assets/lifetime-support-middleware-069163.pdf |

> **Próximo laboratorio:** Lab 02 — Configuración avanzada del dominio, Node Manager y arranque automático de servidores administrados. Prerequisito: este laboratorio completado con el Administration Server en estado RUNNING.

---
*Lab 01-00-01 — WebLogic 15 Administration Course © Laboratorio Académico — Uso exclusivo bajo Oracle Developer License (OTN)*
