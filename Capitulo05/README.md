# Ajuste de parámetros de memoria y heap dumps, ajustar parámetros JVM y observar su impacto

## 1. Metadatos

| Campo            | Valor                                      |
|------------------|--------------------------------------------|
| **Duración**     | 120 minutos                                |
| **Complejidad**  | Alta                                       |
| **Nivel Bloom**  | Crear (Create)                             |
| **Módulo**       | 5 — Optimización de JVM en WebLogic 15     |
| **Lab ID**       | 05-00-01                                   |

---

## 2. Descripción General

En este laboratorio el alumno experimentará de forma empírica el impacto de los parámetros de memoria JVM sobre el comportamiento de WebLogic 15. Partiendo de una configuración base, se aplicará carga real con Apache JMeter, se capturarán métricas de GC con el sistema de logging unificado de Java 21 (`-Xlog:gc*`), y se comparará el comportamiento de **G1GC** frente a **ZGC**. Finalmente, se provocará un `OutOfMemoryError` controlado, se capturará el heap dump resultante y se analizará con **Eclipse Memory Analyzer Tool (MAT)** para identificar objetos dominantes y posibles fugas de memoria.

---

## 3. Objetivos de Aprendizaje

- [ ] Configurar los parámetros de memoria JVM (`-Xms`, `-Xmx`, `-XX:MaxMetaspaceSize`) en `setDomainEnv.sh` y verificar los flags efectivos con `jcmd`.
- [ ] Comparar el comportamiento de G1GC y ZGC bajo carga real midiendo latencia de GC, throughput y pausas mediante los logs unificados de Java 21.
- [ ] Generar y analizar heap dumps (`.hprof`) con Eclipse MAT para identificar objetos que consumen más memoria y detectar fugas controladas.
- [ ] Aplicar los principios de ergonomía JVM de Java 21 para justificar una configuración óptima para el entorno de laboratorio.
- [ ] Documentar un baseline de configuración JVM con justificación técnica reproducible mediante `setDomainEnv.sh` o modelo WDT.

---

## 4. Prerrequisitos

### Conocimiento previo
- Haber completado el **Laboratorio 03-00-01** o **04-00-01**: dominio WebLogic 15 operativo con al menos un Managed Server activo.
- Comprensión básica de los ciclos de recolección de basura (generaciones, STW, fases concurrentes).
- Familiaridad con los flags JVM fundamentales: `-Xms`, `-Xmx`, `-XX:MetaspaceSize`, `-XX:MaxMetaspaceSize`.
- Lectura de la lección **5.1 — Ergonomía de la JVM en WebLogic 15** (material del curso).

### Acceso y herramientas requeridas
- Apache JMeter 5.6.x instalado y funcional (ruta: `/opt/jmeter`).
- Eclipse MAT 1.14.x instalado con heap configurado a ≥ 4 GB (ruta: `/opt/mat`).
- Aplicación de muestra Jakarta EE 10 desplegada en el Managed Server (proporcionada por el instructor).
- Aplicación generadora de fugas de memoria (`leak-generator.war`) — incluida en los recursos del laboratorio.
- Acceso SSH al servidor donde corre WebLogic con privilegios de escritura sobre el dominio.
- Variables de entorno `$DOMAIN_HOME` y `$JAVA_HOME` correctamente definidas.

---

## 5. Entorno del Laboratorio

### Hardware mínimo recomendado

| Recurso      | Mínimo         | Recomendado     |
|--------------|----------------|-----------------|
| RAM          | 16 GB          | 32 GB           |
| CPU          | 4 núcleos      | 8 núcleos       |
| Disco libre  | 20 GB          | 40 GB (SSD)     |

### Software relevante

| Componente               | Versión        | Ruta por defecto              |
|--------------------------|----------------|-------------------------------|
| Oracle JDK               | 21.0.3+        | `/opt/java/jdk-21`            |
| WebLogic Server          | 15.0           | `/opt/oracle/wls15`           |
| Dominio de laboratorio   | lab-domain     | `/opt/oracle/domains/lab`     |
| Apache JMeter            | 5.6.x          | `/opt/jmeter`                 |
| Eclipse MAT              | 1.14.x         | `/opt/mat`                    |
| Logs GC                  | —              | `/opt/oracle/domains/lab/logs/gc` |

### Preparación inicial del entorno

Ejecuta los siguientes comandos antes de comenzar los pasos del laboratorio:

```bash
# 1. Verificar que Java 21 está activo
java -version

# 2. Confirmar que las variables de entorno están definidas
echo "DOMAIN_HOME: $DOMAIN_HOME"
echo "JAVA_HOME:   $JAVA_HOME"

# 3. Crear directorio para logs de GC si no existe
mkdir -p $DOMAIN_HOME/logs/gc

# 4. Verificar que el Administration Server está en ejecución
curl -s -o /dev/null -w "%{http_code}" \
  http://localhost:7001/console/login/LoginForm.jsp
# Resultado esperado: 200

# 5. Verificar que el Managed Server (ms1) está en ejecución
curl -s -o /dev/null -w "%{http_code}" \
  http://localhost:8001/sampleapp/index.jsp
# Resultado esperado: 200
```

> **Nota:** Si alguno de los servidores no está activo, arráncalos antes de continuar usando los scripts `startWebLogic.sh` y `startManagedWebLogic.sh` de tu dominio.

---

## 6. Pasos del Laboratorio

---

### Paso 1 — Capturar el baseline de configuración JVM por defecto

**Objetivo:** Registrar la configuración JVM inicial del Managed Server antes de realizar cualquier cambio, para tener una referencia de comparación.

#### Instrucciones

1. Obtén el PID del proceso Java del Managed Server:

```bash
# Buscar el proceso del Managed Server (ms1)
jps -l | grep weblogic.Server
# Anota el PID; se usará como <PID_MS1> en los pasos siguientes
```

2. Inspecciona los flags JVM efectivos:

```bash
jcmd <PID_MS1> VM.flags
```

3. Filtra los flags más relevantes para este laboratorio:

```bash
jcmd <PID_MS1> VM.flags | egrep \
  "UseG1GC|UseZGC|MaxGCPauseMillis|InitialHeapSize|MaxHeapSize|\
MaxMetaspaceSize|MetaspaceSize|ActiveProcessorCount|ExitOnOutOfMemoryError"
```

4. Consulta el estado actual del heap:

```bash
jcmd <PID_MS1> GC.heap_info
```

5. Registra los valores en tu cuaderno de laboratorio o en el archivo de baseline:

```bash
# Guardar baseline en archivo de texto
jcmd <PID_MS1> VM.flags > $DOMAIN_HOME/logs/gc/baseline_flags.txt
jcmd <PID_MS1> GC.heap_info >> $DOMAIN_HOME/logs/gc/baseline_flags.txt
echo "Fecha: $(date)" >> $DOMAIN_HOME/logs/gc/baseline_flags.txt
```

**Salida esperada (ejemplo):**
```
 -XX:InitialHeapSize=268435456 -XX:MaxHeapSize=536870912
 -XX:+UseG1GC -XX:MaxGCPauseMillis=200
```
> Los valores exactos dependerán de la RAM disponible en tu máquina. Java 21 aplica ergonomía automática (~25 % de la RAM para el heap inicial, ~50 % para el máximo cuando no se especifican explícitamente).

**Verificación:**
```bash
# El archivo de baseline debe existir y contener datos
cat $DOMAIN_HOME/logs/gc/baseline_flags.txt | head -20
```

---

### Paso 2 — Configurar G1GC con logging detallado y heap explícito (512m)

**Objetivo:** Establecer la primera configuración de prueba con heap pequeño (512m) y habilitar el logging unificado de GC para capturar métricas de referencia.

#### Instrucciones

1. Abre el archivo `setDomainEnv.sh` del dominio:

```bash
cp $DOMAIN_HOME/bin/setDomainEnv.sh \
   $DOMAIN_HOME/bin/setDomainEnv.sh.bak_original
vi $DOMAIN_HOME/bin/setDomainEnv.sh
```

2. Localiza la sección donde se define `USER_MEM_ARGS` (busca la cadena `USER_MEM_ARGS`). Si no existe, añádela antes de la línea `export USER_MEM_ARGS`. Configura el bloque con los parámetros de la **Configuración A — G1GC 512m**:

```bash
# ============================================================
# LAB 05-00-01 - Configuración A: G1GC heap 512m
# ============================================================
USER_MEM_ARGS="-Xms512m -Xmx512m \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:MaxMetaspaceSize=256m \
  -XX:+ExitOnOutOfMemoryError \
  -Xlog:gc*,safepoint,class+unload=info\
:file=${DOMAIN_HOME}/logs/gc/ms1-gc-g1-512m.log\
:tags,uptime,time,level\
:filecount=5,filesize=20m"
export USER_MEM_ARGS
```

> **Importante:** Asegúrate de que la variable se establece **después** de cualquier bloque condicional que pueda sobreescribirla, o añade la lógica `if [ -z "$USER_MEM_ARGS" ]` según la estructura de tu `setDomainEnv.sh`.

3. Guarda el archivo y reinicia el Managed Server:

```bash
# Detener Managed Server (desde el nodo del MS o via WLST)
$DOMAIN_HOME/bin/stopManagedWebLogic.sh ms1 \
  t3://localhost:7001 weblogic Welcome1

# Esperar 15 segundos y arrancar de nuevo
sleep 15
nohup $DOMAIN_HOME/bin/startManagedWebLogic.sh ms1 \
  t3://localhost:7001 > $DOMAIN_HOME/logs/ms1-start.log 2>&1 &
```

4. Espera a que el servidor esté activo (aproximadamente 30–60 segundos):

```bash
# Monitorear el arranque
tail -f $DOMAIN_HOME/logs/ms1-start.log | grep -E "RUNNING|ERROR|Exception"
# Ctrl+C cuando veas: <Server state changed to RUNNING>
```

5. Verifica que los flags se aplicaron correctamente:

```bash
PID_MS1=$(jps -l | grep weblogic.Server | awk '{print $1}')
jcmd $PID_MS1 VM.flags | egrep \
  "UseG1GC|MaxGCPauseMillis|InitialHeapSize|MaxHeapSize|MaxMetaspaceSize"
```

**Salida esperada:**
```
-XX:InitialHeapSize=536870912 -XX:MaxHeapSize=536870912
-XX:MaxGCPauseMillis=200 -XX:MaxMetaspaceSize=268435456
-XX:+UseG1GC
```
> `536870912` bytes = 512 MiB.

**Verificación:**
```bash
# El log de GC debe haberse creado
ls -lh $DOMAIN_HOME/logs/gc/ms1-gc-g1-512m.log
# Mostrar las primeras líneas del log de GC
head -30 $DOMAIN_HOME/logs/gc/ms1-gc-g1-512m.log
```

---

### Paso 3 — Ejecutar prueba de carga con JMeter (G1GC 512m) y capturar métricas

**Objetivo:** Generar carga HTTP sobre el Managed Server con JMeter y registrar las métricas de GC para la configuración G1GC 512m.

#### Instrucciones

1. Abre JMeter y carga el plan de prueba proporcionado por el instructor (`lab05-load-test.jmx`), o crea uno básico desde la línea de comandos:

```bash
# Ejecutar JMeter en modo no-GUI (recomendado para carga real)
$JMETER_HOME/bin/jmeter -n \
  -t $HOME/lab-resources/lab05-load-test.jmx \
  -l $DOMAIN_HOME/logs/gc/jmeter-results-g1-512m.jtl \
  -e -o $DOMAIN_HOME/logs/gc/jmeter-report-g1-512m \
  -Jthreads=50 \
  -Jrampup=30 \
  -Jduration=120 \
  -Jtarget_host=localhost \
  -Jtarget_port=8001
```

> Si no dispones del archivo `.jmx` del instructor, usa el siguiente plan mínimo en línea de comandos como alternativa rápida:

```bash
# Alternativa: usar curl en bucle para simular carga básica
for i in $(seq 1 500); do
  curl -s -o /dev/null http://localhost:8001/sampleapp/index.jsp &
  if (( i % 20 == 0 )); then sleep 1; fi
done
wait
echo "Carga básica completada"
```

2. Mientras la carga está activa, monitorea el GC en tiempo real en otra terminal:

```bash
# Terminal 2: seguir el log de GC en tiempo real
tail -f $DOMAIN_HOME/logs/gc/ms1-gc-g1-512m.log | \
  grep -E "GC\(|Pause|Concurrent"
```

3. Consulta el estado del heap durante la carga:

```bash
# Terminal 3: monitoreo periódico del heap
watch -n 5 "jcmd $(jps -l | grep weblogic.Server | awk '{print \$1}') GC.heap_info"
```

4. Al finalizar la carga (120 segundos), extrae las métricas clave del log de GC:

```bash
# Contar el número total de pausas GC
grep "GC(" $DOMAIN_HOME/logs/gc/ms1-gc-g1-512m.log | wc -l

# Extraer duración de cada pausa (en ms)
grep "GC pause" $DOMAIN_HOME/logs/gc/ms1-gc-g1-512m.log | \
  awk '{print $NF}' | sort -n | \
  awk 'BEGIN{sum=0;count=0} {sum+=$1;count++} END{
    print "Total pausas:", count;
    print "Pausa máxima:", $1, "ms";
    print "Pausa media:", sum/count, "ms"
  }'

# Guardar resumen en archivo
echo "=== G1GC 512m ===" > $DOMAIN_HOME/logs/gc/summary-g1-512m.txt
grep "GC pause" $DOMAIN_HOME/logs/gc/ms1-gc-g1-512m.log \
  >> $DOMAIN_HOME/logs/gc/summary-g1-512m.txt
```

**Salida esperada (ejemplo orientativo):**
```
Total pausas: 47
Pausa máxima: 312 ms
Pausa media: 89 ms
```
> Con 512m de heap y carga moderada, es probable observar GC frecuente (alta frecuencia de Young GC) y algunas pausas superiores al objetivo de 200ms.

**Verificación:**
```bash
# El reporte de JMeter debe existir
ls $DOMAIN_HOME/logs/gc/jmeter-report-g1-512m/index.html
# El resumen de GC debe tener datos
wc -l $DOMAIN_HOME/logs/gc/summary-g1-512m.txt
```

---

### Paso 4 — Repetir la prueba con heap mayor (G1GC 4g) y comparar

**Objetivo:** Aumentar el heap a 4 GB manteniendo G1GC y comparar el impacto en la frecuencia de GC y las pausas.

#### Instrucciones

1. Modifica `setDomainEnv.sh` para la **Configuración B — G1GC 4g**:

```bash
# Editar setDomainEnv.sh
vi $DOMAIN_HOME/bin/setDomainEnv.sh
```

Actualiza el bloque `USER_MEM_ARGS`:

```bash
# ============================================================
# LAB 05-00-01 - Configuración B: G1GC heap 4g
# ============================================================
USER_MEM_ARGS="-Xms4g -Xmx4g \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=150 \
  -XX:MaxMetaspaceSize=512m \
  -XX:+UseStringDeduplication \
  -XX:+ExitOnOutOfMemoryError \
  -Xlog:gc*,safepoint,class+unload=info\
:file=${DOMAIN_HOME}/logs/gc/ms1-gc-g1-4g.log\
:tags,uptime,time,level\
:filecount=5,filesize=20m"
export USER_MEM_ARGS
```

2. Reinicia el Managed Server:

```bash
$DOMAIN_HOME/bin/stopManagedWebLogic.sh ms1 \
  t3://localhost:7001 weblogic Welcome1
sleep 20
nohup $DOMAIN_HOME/bin/startManagedWebLogic.sh ms1 \
  t3://localhost:7001 > $DOMAIN_HOME/logs/ms1-start.log 2>&1 &

# Esperar estado RUNNING
sleep 60
curl -s -o /dev/null -w "%{http_code}" \
  http://localhost:8001/sampleapp/index.jsp
```

3. Verifica los nuevos flags:

```bash
PID_MS1=$(jps -l | grep weblogic.Server | awk '{print $1}')
jcmd $PID_MS1 VM.flags | egrep \
  "UseG1GC|MaxGCPauseMillis|InitialHeapSize|MaxHeapSize|UseStringDeduplication"
```

4. Ejecuta la **misma prueba de carga** con idénticos parámetros:

```bash
$JMETER_HOME/bin/jmeter -n \
  -t $HOME/lab-resources/lab05-load-test.jmx \
  -l $DOMAIN_HOME/logs/gc/jmeter-results-g1-4g.jtl \
  -e -o $DOMAIN_HOME/logs/gc/jmeter-report-g1-4g \
  -Jthreads=50 \
  -Jrampup=30 \
  -Jduration=120 \
  -Jtarget_host=localhost \
  -Jtarget_port=8001
```

5. Extrae métricas y compara:

```bash
echo "=== G1GC 4g ===" > $DOMAIN_HOME/logs/gc/summary-g1-4g.txt
grep "GC pause" $DOMAIN_HOME/logs/gc/ms1-gc-g1-4g.log \
  >> $DOMAIN_HOME/logs/gc/summary-g1-4g.txt

# Comparativa rápida
echo "--- COMPARATIVA ---"
echo "G1GC 512m:"
grep "GC pause" $DOMAIN_HOME/logs/gc/ms1-gc-g1-512m.log | wc -l
echo "G1GC 4g:"
grep "GC pause" $DOMAIN_HOME/logs/gc/ms1-gc-g1-4g.log | wc -l
```

**Salida esperada (ejemplo orientativo):**
```
G1GC 512m: 47 pausas (media ~89ms, máx ~312ms)
G1GC 4g:   12 pausas (media ~45ms, máx ~180ms)
```
> Con más heap, G1GC recolecta con menor frecuencia y las pausas tienden a ser más cortas y predecibles.

**Verificación:**
```bash
diff $DOMAIN_HOME/logs/gc/summary-g1-512m.txt \
     $DOMAIN_HOME/logs/gc/summary-g1-4g.txt | head -30
```

---

### Paso 5 — Cambiar a ZGC y repetir las pruebas de carga

**Objetivo:** Sustituir G1GC por ZGC y observar el impacto en las pausas STW, especialmente en heaps de tamaño medio-grande.

#### Instrucciones

1. Modifica `setDomainEnv.sh` para la **Configuración C — ZGC 4g**:

```bash
vi $DOMAIN_HOME/bin/setDomainEnv.sh
```

```bash
# ============================================================
# LAB 05-00-01 - Configuración C: ZGC heap 4g
# ============================================================
USER_MEM_ARGS="-Xms4g -Xmx4g \
  -XX:+UseZGC \
  -XX:MaxMetaspaceSize=512m \
  -XX:+ExitOnOutOfMemoryError \
  -Xlog:gc*,safepoint,class+unload=info\
:file=${DOMAIN_HOME}/logs/gc/ms1-gc-zgc-4g.log\
:tags,uptime,time,level\
:filecount=5,filesize=20m"
export USER_MEM_ARGS
```

> **Nota:** ZGC no usa `-XX:MaxGCPauseMillis` de la misma forma que G1GC. ZGC está diseñado para mantener pausas STW por debajo de 1ms independientemente del tamaño del heap.

2. Reinicia el Managed Server:

```bash
$DOMAIN_HOME/bin/stopManagedWebLogic.sh ms1 \
  t3://localhost:7001 weblogic Welcome1
sleep 20
nohup $DOMAIN_HOME/bin/startManagedWebLogic.sh ms1 \
  t3://localhost:7001 > $DOMAIN_HOME/logs/ms1-start.log 2>&1 &
sleep 60

# Verificar arranque exitoso
curl -s -o /dev/null -w "%{http_code}" \
  http://localhost:8001/sampleapp/index.jsp
```

3. Confirma que ZGC está activo:

```bash
PID_MS1=$(jps -l | grep weblogic.Server | awk '{print $1}')
jcmd $PID_MS1 VM.flags | grep -E "UseZGC|UseG1GC"
# Debe mostrar: -XX:+UseZGC y -XX:-UseG1GC
```

4. Ejecuta la prueba de carga con los mismos parámetros:

```bash
$JMETER_HOME/bin/jmeter -n \
  -t $HOME/lab-resources/lab05-load-test.jmx \
  -l $DOMAIN_HOME/logs/gc/jmeter-results-zgc-4g.jtl \
  -e -o $DOMAIN_HOME/logs/gc/jmeter-report-zgc-4g \
  -Jthreads=50 \
  -Jrampup=30 \
  -Jduration=120 \
  -Jtarget_host=localhost \
  -Jtarget_port=8001
```

5. Analiza las pausas de ZGC:

```bash
# ZGC usa etiquetas diferentes en el log
echo "=== ZGC 4g ===" > $DOMAIN_HOME/logs/gc/summary-zgc-4g.txt
grep -E "Pause|GC\(" $DOMAIN_HOME/logs/gc/ms1-gc-zgc-4g.log \
  >> $DOMAIN_HOME/logs/gc/summary-zgc-4g.txt

# Comparativa final de los tres escenarios
echo "============================================"
echo "RESUMEN COMPARATIVO DE CONFIGURACIONES JVM"
echo "============================================"
for cfg in g1-512m g1-4g zgc-4g; do
  echo -n "[$cfg] Número de eventos GC: "
  grep -c "GC\(" $DOMAIN_HOME/logs/gc/ms1-gc-${cfg}.log 2>/dev/null || echo "0"
done
```

**Salida esperada (ejemplo orientativo):**
```
[g1-512m] Número de eventos GC: 47
[g1-4g]   Número de eventos GC: 12
[zgc-4g]  Número de eventos GC: 8  (con pausas STW < 5ms cada una)
```

**Verificación:**
```bash
# Confirmar que los tres logs de GC existen y tienen contenido
for f in ms1-gc-g1-512m.log ms1-gc-g1-4g.log ms1-gc-zgc-4g.log; do
  echo "$f: $(wc -l < $DOMAIN_HOME/logs/gc/$f) líneas"
done
```

---

### Paso 6 — Provocar un OutOfMemoryError controlado y capturar el heap dump

**Objetivo:** Desplegar la aplicación `leak-generator.war`, configurar la JVM para capturar automáticamente un heap dump ante OOM, y provocar el error de forma controlada.

#### Instrucciones

1. Añade los flags de heap dump automático a `setDomainEnv.sh`. Usa como base la **Configuración D — G1GC con OOM dump** (heap deliberadamente pequeño para facilitar el OOM):

```bash
vi $DOMAIN_HOME/bin/setDomainEnv.sh
```

```bash
# ============================================================
# LAB 05-00-01 - Configuración D: G1GC 1g + HeapDump on OOM
# ============================================================
USER_MEM_ARGS="-Xms1g -Xmx1g \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:MaxMetaspaceSize=256m \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=${DOMAIN_HOME}/logs/gc/heapdump-oom.hprof \
  -XX:+ExitOnOutOfMemoryError \
  -Xlog:gc*,safepoint=info\
:file=${DOMAIN_HOME}/logs/gc/ms1-gc-oom-test.log\
:tags,uptime,time,level\
:filecount=3,filesize=10m"
export USER_MEM_ARGS
```

2. Reinicia el Managed Server:

```bash
$DOMAIN_HOME/bin/stopManagedWebLogic.sh ms1 \
  t3://localhost:7001 weblogic Welcome1
sleep 20
nohup $DOMAIN_HOME/bin/startManagedWebLogic.sh ms1 \
  t3://localhost:7001 > $DOMAIN_HOME/logs/ms1-start.log 2>&1 &
sleep 60
```

3. Despliega la aplicación generadora de fugas:

```bash
# Copiar el WAR al directorio de autodeploy
cp $HOME/lab-resources/leak-generator.war \
   $DOMAIN_HOME/autodeploy/

# Verificar el despliegue (esperar ~30 segundos)
sleep 30
curl -s -o /dev/null -w "%{http_code}" \
  http://localhost:8001/leak-generator/status
# Resultado esperado: 200
```

4. Activa la fuga de memoria mediante la API REST de la aplicación:

```bash
# Iniciar la generación de objetos en memoria (fuga controlada)
# El endpoint /leak acepta el parámetro 'mb' para indicar MB a retener
curl -X POST \
  "http://localhost:8001/leak-generator/leak?mb=50&iterations=20" \
  -v

# Monitorear el heap mientras se genera la fuga
PID_MS1=$(jps -l | grep weblogic.Server | awk '{print $1}')
watch -n 3 "jcmd $PID_MS1 GC.heap_info 2>/dev/null | grep -E 'heap|used'"
```

5. Continúa activando la fuga hasta provocar el OOM (puede tardar 2–5 minutos):

```bash
# Bucle para acumular objetos hasta OOM
for i in $(seq 1 15); do
  echo "Iteración $i: acumulando 50MB más..."
  curl -s -X POST \
    "http://localhost:8001/leak-generator/leak?mb=50&iterations=5" \
    || echo "Servidor no respondió (posible OOM)"
  sleep 10
done
```

6. Verifica que el heap dump fue generado:

```bash
ls -lh $DOMAIN_HOME/logs/gc/heapdump-oom.hprof
# Debe mostrar un archivo de tamaño aproximado al heap configurado (~1GB)
```

**Salida esperada:**
```
-rw-r--r-- 1 oracle oracle 987M May 15 14:32 heapdump-oom.hprof
```

**Verificación:**
```bash
# Confirmar que el log de GC registró el OOM
grep -i "OutOfMemory\|OOM\|Heap Dump" \
  $DOMAIN_HOME/logs/gc/ms1-gc-oom-test.log | tail -5

# También verificar en el log del servidor
grep -i "OutOfMemoryError" \
  $DOMAIN_HOME/logs/ms1/ms1.log | tail -5
```

---

### Paso 7 — Analizar el heap dump con Eclipse MAT

**Objetivo:** Abrir el heap dump en Eclipse MAT, identificar los objetos dominantes, analizar el árbol de retención y generar el reporte de fugas.

#### Instrucciones

1. Configura Eclipse MAT con heap suficiente para analizar el dump de 1 GB:

```bash
# Editar el archivo de configuración de MAT
vi /opt/mat/MemoryAnalyzer.ini
```

Asegúrate de que contiene al menos:
```
-Xms1g
-Xmx4g
```

2. Abre Eclipse MAT:

```bash
/opt/mat/MemoryAnalyzer &
```

3. Carga el heap dump:
   - Ve a **File → Open Heap Dump...**
   - Navega a `$DOMAIN_HOME/logs/gc/heapdump-oom.hprof`
   - Selecciona el archivo y haz clic en **Finish**
   - MAT tardará varios minutos en parsear e indexar el dump (~1 GB)

4. Una vez cargado, ejecuta el **Leak Suspects Report**:
   - En la pantalla de bienvenida de MAT, haz clic en **"Leak Suspects"**
   - O desde el menú: **Reports → Leak Suspects**
   - MAT generará automáticamente un informe con los principales sospechosos de fuga

5. Analiza el **Dominator Tree** para identificar los objetos que retienen más memoria:
   - Ve a **Window → Heap Dump Details → Dominator Tree**
   - Ordena por **"Retained Heap"** de mayor a menor
   - Busca objetos de la clase `leak-generator` o colecciones (`ArrayList`, `HashMap`) con retención anormalmente alta

6. Usa la consulta OQL para buscar instancias de la clase de fuga:

```sql
-- En MAT: Window → OQL (Object Query Language)
SELECT * FROM java.util.ArrayList a WHERE a.size > 10000
```

7. Examina el **Histogram** para ver la distribución por clase:
   - Ve a **Window → Heap Dump Details → Class Histogram**
   - Busca las clases con mayor número de instancias y mayor heap retenido
   - Anota las 5 clases principales en tu cuaderno de laboratorio

8. Genera y guarda el reporte completo:

```bash
# MAT puede ejecutarse también en modo batch para generar reportes
/opt/mat/ParseHeapDump.sh \
  $DOMAIN_HOME/logs/gc/heapdump-oom.hprof \
  org.eclipse.mat.api:leakhunter \
  org.eclipse.mat.api:suspects \
  -vmargs -Xmx4g

# El reporte se genera en el mismo directorio del .hprof
ls $DOMAIN_HOME/logs/gc/ | grep -E "\.zip|\.html"
```

**Salida esperada en MAT (ejemplo):**
```
Problem Suspect 1:
  One instance of "com.lab.LeakGenerator$MemoryHolder"
  loaded by "WebApp ClassLoader" occupies 734 MB (73.4%)
  of the heap.

  Shortest paths to the accumulation point:
  LeakGenerator.retainedObjects -> ArrayList[50000 entries]
    -> byte[][] (average 15 KB each)
```

**Verificación:**
```bash
# El reporte en batch debe haber generado archivos de salida
ls -lh $DOMAIN_HOME/logs/gc/*.zip 2>/dev/null || \
  echo "Revisar directorio de MAT para el reporte HTML"
```

---

### Paso 8 — Documentar la configuración JVM óptima como baseline

**Objetivo:** Sintetizar los resultados de las pruebas y documentar la configuración JVM óptima para el entorno de laboratorio, tanto en `setDomainEnv.sh` como en formato WDT.

#### Instrucciones

1. Basándote en los resultados de los pasos 3, 4 y 5, establece la **Configuración Final Recomendada** en `setDomainEnv.sh`:

```bash
vi $DOMAIN_HOME/bin/setDomainEnv.sh
```

```bash
# ============================================================
# LAB 05-00-01 - Configuración FINAL BASELINE
# Justificación: G1GC con heap 4g ofrece el mejor equilibrio
# entre frecuencia de GC (12 eventos vs 47) y pausas medias
# (~45ms vs ~89ms) para la carga de laboratorio.
# ZGC es alternativa para SLAs con latencia < 10ms.
# ============================================================
USER_MEM_ARGS="-Xms4g -Xmx4g \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=150 \
  -XX:MaxMetaspaceSize=512m \
  -XX:+UseStringDeduplication \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=${DOMAIN_HOME}/logs/gc/ \
  -XX:+ExitOnOutOfMemoryError \
  -Xlog:gc*,safepoint,class+unload=info\
:file=${DOMAIN_HOME}/logs/gc/ms1-gc-baseline.log\
:tags,uptime,time,level\
:filecount=10,filesize=20m"
export USER_MEM_ARGS
```

2. Documenta la misma configuración en formato **WDT (YAML)**:

```bash
cat > $HOME/lab-resources/jvm-baseline-model.yaml << 'EOF'
# WDT Model - JVM Baseline Configuration
# Lab 05-00-01 - Configuración óptima documentada
domainInfo:
  AdminUserName: "weblogic"

topology:
  Name: "lab-domain"
  Servers:
    ms1:
      ListenPort: 8001
      ServerStart:
        Arguments: >
          -Xms4g -Xmx4g
          -XX:+UseG1GC
          -XX:MaxGCPauseMillis=150
          -XX:MaxMetaspaceSize=512m
          -XX:+UseStringDeduplication
          -XX:+HeapDumpOnOutOfMemoryError
          -XX:HeapDumpPath=/opt/oracle/domains/lab/logs/gc/
          -XX:+ExitOnOutOfMemoryError
          -Xlog:gc*,safepoint,class+unload=info:file=/opt/oracle/domains/lab/logs/gc/ms1-gc-baseline.log:tags,uptime,time,level:filecount=10,filesize=20m
EOF

echo "Modelo WDT generado:"
cat $HOME/lab-resources/jvm-baseline-model.yaml
```

3. Reinicia el Managed Server con la configuración final y verifica:

```bash
$DOMAIN_HOME/bin/stopManagedWebLogic.sh ms1 \
  t3://localhost:7001 weblogic Welcome1
sleep 20
nohup $DOMAIN_HOME/bin/startManagedWebLogic.sh ms1 \
  t3://localhost:7001 > $DOMAIN_HOME/logs/ms1-start.log 2>&1 &
sleep 60

PID_MS1=$(jps -l | grep weblogic.Server | awk '{print $1}')
jcmd $PID_MS1 VM.flags | egrep \
  "UseG1GC|MaxGCPauseMillis|InitialHeapSize|MaxHeapSize|\
MaxMetaspaceSize|HeapDumpOnOutOfMemoryError|UseStringDeduplication"
```

4. Crea el documento de baseline final:

```bash
cat > $DOMAIN_HOME/logs/gc/JVM_BASELINE_REPORT.txt << EOF
========================================
REPORTE DE BASELINE JVM - Lab 05-00-01
Fecha: $(date)
Servidor: ms1 (Managed Server)
WebLogic: 15.0 | Java: 21
========================================

RESULTADOS DE PRUEBAS DE CARGA (50 hilos, 120s, misma app):

| Configuración  | Heap  | GC    | Eventos GC | Pausa Media | Pausa Máx |
|----------------|-------|-------|------------|-------------|-----------|
| G1GC           | 512m  | G1    | ~47        | ~89ms       | ~312ms    |
| G1GC           | 4g    | G1    | ~12        | ~45ms       | ~180ms    |
| ZGC            | 4g    | ZGC   | ~8         | <5ms STW    | <10ms STW |

CONFIGURACIÓN SELECCIONADA: G1GC 4g
JUSTIFICACIÓN:
- Equilibrio óptimo entre frecuencia de GC y latencia de pausa.
- G1GC es el GC por defecto recomendado para cargas de servidor en Java 21.
- ZGC es preferible solo si el SLA exige latencias sub-10ms y el overhead
  de CPU concurrente es aceptable.
- -Xms=-Xmx (4g=4g) elimina redimensionado dinámico del heap en producción.
- UseStringDeduplication reduce huella de memoria en apps con muchos Strings.

FLAGS EFECTIVOS VERIFICADOS CON jcmd:
$(jcmd $PID_MS1 VM.flags 2>/dev/null)

HEAP INFO ACTUAL:
$(jcmd $PID_MS1 GC.heap_info 2>/dev/null)
========================================
EOF

cat $DOMAIN_HOME/logs/gc/JVM_BASELINE_REPORT.txt
```

**Verificación:**
```bash
# El reporte debe existir y contener la tabla comparativa
grep "CONFIGURACIÓN SELECCIONADA" \
  $DOMAIN_HOME/logs/gc/JVM_BASELINE_REPORT.txt

# El modelo WDT debe ser válido (verificación de sintaxis YAML)
python3 -c "import yaml; yaml.safe_load(open('$HOME/lab-resources/jvm-baseline-model.yaml'))" \
  && echo "YAML válido" || echo "ERROR: YAML inválido"
```

---

## 7. Validación y Pruebas Finales

Ejecuta los siguientes comandos de validación para confirmar que el laboratorio se ha completado correctamente:

```bash
#!/bin/bash
# Script de validación final - Lab 05-00-01
echo "=========================================="
echo "VALIDACIÓN FINAL - Lab 05-00-01"
echo "=========================================="

PASS=0
FAIL=0

check() {
  local desc="$1"
  local cmd="$2"
  if eval "$cmd" > /dev/null 2>&1; then
    echo "✓ PASS: $desc"
    ((PASS++))
  else
    echo "✗ FAIL: $desc"
    ((FAIL++))
  fi
}

# 1. Managed Server en estado RUNNING
check "Managed Server responde en puerto 8001" \
  "curl -sf http://localhost:8001/sampleapp/index.jsp"

# 2. Log de GC baseline existe y tiene contenido
check "Log de GC baseline generado" \
  "test -s $DOMAIN_HOME/logs/gc/ms1-gc-baseline.log"

# 3. Tres logs de prueba existen
check "Log G1GC 512m existe" \
  "test -f $DOMAIN_HOME/logs/gc/ms1-gc-g1-512m.log"
check "Log G1GC 4g existe" \
  "test -f $DOMAIN_HOME/logs/gc/ms1-gc-g1-4g.log"
check "Log ZGC 4g existe" \
  "test -f $DOMAIN_HOME/logs/gc/ms1-gc-zgc-4g.log"

# 4. Heap dump generado
check "Heap dump OOM generado" \
  "test -f $DOMAIN_HOME/logs/gc/heapdump-oom.hprof"

# 5. Flags JVM correctos en la configuración final
PID=$(jps -l 2>/dev/null | grep weblogic.Server | awk '{print $1}')
check "Flag UseG1GC activo" \
  "jcmd $PID VM.flags 2>/dev/null | grep -q '+UseG1GC'"
check "Heap configurado a 4g" \
  "jcmd $PID VM.flags 2>/dev/null | grep -q 'MaxHeapSize=4294967296'"
check "HeapDumpOnOutOfMemoryError activo" \
  "jcmd $PID VM.flags 2>/dev/null | grep -q '+HeapDumpOnOutOfMemoryError'"

# 6. Modelo WDT generado
check "Modelo WDT YAML existe" \
  "test -f $HOME/lab-resources/jvm-baseline-model.yaml"

# 7. Reporte de baseline existe
check "Reporte JVM_BASELINE_REPORT.txt existe" \
  "test -f $DOMAIN_HOME/logs/gc/JVM_BASELINE_REPORT.txt"

echo "------------------------------------------"
echo "Resultado: $PASS pasados, $FAIL fallidos"
echo "=========================================="
```

**Resultado esperado:** 10/10 checks en estado PASS. Si alguno falla, revisa el paso correspondiente según la descripción del check.

---

## 8. Resolución de Problemas

### Problema 1 — El Managed Server no arranca tras modificar `USER_MEM_ARGS`

**Síntomas:**
- El proceso Java del Managed Server termina inmediatamente tras el arranque.
- El log `ms1-start.log` muestra errores como `Error occurred during initialization of VM` o `Could not reserve enough space for object heap`.
- `jps -l` no muestra ningún proceso `weblogic.Server` activo.

**Causa:**
La máquina no dispone de memoria física suficiente para satisfacer el valor de `-Xms` configurado. Si se especifica `-Xms4g` pero el sistema solo tiene 4 GB de RAM total (con otros procesos activos), la JVM no puede reservar el heap al inicio. También puede ocurrir si hay un error tipográfico en los flags (ej. `-Xms4G` en lugar de `-Xms4g` en ciertos contextos, o un flag mal formado que la JVM no reconoce).

**Solución:**

```bash
# 1. Verificar memoria disponible en el sistema
free -h
# Si el disponible es < 4g, reducir los valores

# 2. Revisar el log de arranque para el error exacto
cat $DOMAIN_HOME/logs/ms1-start.log | grep -E "Error|Exception|Could not"

# 3. Restaurar la configuración de backup y reducir el heap
cp $DOMAIN_HOME/bin/setDomainEnv.sh.bak_original \
   $DOMAIN_HOME/bin/setDomainEnv.sh

# 4. Editar con valores más conservadores (ej. 2g en lugar de 4g)
vi $DOMAIN_HOME/bin/setDomainEnv.sh
# Cambiar -Xms4g -Xmx4g por -Xms2g -Xmx2g

# 5. Verificar la sintaxis del flag antes de reiniciar
java -Xms2g -Xmx2g -XX:+UseG1GC -version
# Si este comando funciona, los flags son válidos

# 6. Reintentar el arranque
nohup $DOMAIN_HOME/bin/startManagedWebLogic.sh ms1 \
  t3://localhost:7001 > $DOMAIN_HOME/logs/ms1-start.log 2>&1 &
```

---

### Problema 2 — Eclipse MAT falla al abrir el heap dump con `OutOfMemoryError`

**Síntomas:**
- Eclipse MAT se cierra inesperadamente al intentar parsear el archivo `.hprof`.
- Aparece un diálogo de error: `An internal error occurred. Java heap space` o `GC overhead limit exceeded`.
- El análisis se detiene al 30–50 % del proceso de indexación.

**Causa:**
El heap configurado para Eclipse MAT es insuficiente para analizar un dump de ~1 GB. Por defecto, MAT puede estar configurado con `-Xmx1g` o `-Xmx2g`, lo que es insuficiente para procesar un heap dump de tamaño similar o mayor (MAT necesita aproximadamente 1.5–2x el tamaño del dump para el análisis completo).

**Solución:**

```bash
# 1. Localizar y editar el archivo de configuración de MAT
cat /opt/mat/MemoryAnalyzer.ini

# 2. Aumentar el heap de MAT a al menos 4g (o más si el dump es > 2g)
vi /opt/mat/MemoryAnalyzer.ini
```

El archivo debe contener:
```
-startup
plugins/org.eclipse.equinox.launcher_*.jar
--launcher.library
plugins/org.eclipse.equinox.launcher.gtk.linux.x86_64_*/eclipse_*.so
-vmargs
-Xms1g
-Xmx6g
-XX:+UseG1GC
```

```bash
# 3. Alternativa: usar MAT en modo batch desde línea de comandos
#    (más eficiente en memoria que la GUI)
/opt/mat/ParseHeapDump.sh \
  $DOMAIN_HOME/logs/gc/heapdump-oom.hprof \
  org.eclipse.mat.api:leakhunter \
  -vmargs -Xmx6g -XX:+UseG1GC

# 4. Si la máquina tiene menos de 8g RAM total, generar primero
#    un dump más pequeño para el análisis (heap de 512m):
#    Repite el Paso 6 con -Xmx512m en lugar de -Xmx1g para
#    obtener un .hprof de ~500MB más manejable.

# 5. Verificar que MAT arranca con el nuevo heap
/opt/mat/MemoryAnalyzer &
# En Help → About → Installation Details → Configuration
# verificar que -Xmx6g aparece en los argumentos JVM
```

---

## 9. Limpieza del Entorno

Ejecuta los siguientes pasos para dejar el entorno en un estado consistente para futuros laboratorios:

```bash
# 1. Restaurar la configuración JVM a la configuración final del baseline
#    (ya establecida en el Paso 8 - no requiere cambios adicionales)
echo "Configuración JVM final del baseline activa en setDomainEnv.sh"

# 2. Eliminar la aplicación generadora de fugas
rm -f $DOMAIN_HOME/autodeploy/leak-generator.war
# Esperar a que WebLogic detecte el undeploy automático (~30s)
sleep 30
curl -s -o /dev/null -w "Estado leak-generator: %{http_code}\n" \
  http://localhost:8001/leak-generator/status
# Debe retornar 404 (aplicación eliminada)

# 3. Archivar los logs de GC de las pruebas (no eliminar, son referencia)
mkdir -p $DOMAIN_HOME/logs/gc/archive-lab05
mv $DOMAIN_HOME/logs/gc/ms1-gc-g1-512m.log \
   $DOMAIN_HOME/logs/gc/ms1-gc-g1-4g.log \
   $DOMAIN_HOME/logs/gc/ms1-gc-zgc-4g.log \
   $DOMAIN_HOME/logs/gc/ms1-gc-oom-test.log \
   $DOMAIN_HOME/logs/gc/archive-lab05/ 2>/dev/null
echo "Logs archivados en: $DOMAIN_HOME/logs/gc/archive-lab05/"

# 4. El heap dump puede ocupar mucho espacio; comprimir o eliminar
#    según disponibilidad de disco
ls -lh $DOMAIN_HOME/logs/gc/heapdump-oom.hprof
# Opción A: Comprimir para conservar
gzip $DOMAIN_HOME/logs/gc/heapdump-oom.hprof
# Opción B: Eliminar si no se necesita
# rm $DOMAIN_HOME/logs/gc/heapdump-oom.hprof

# 5. Verificar que el Managed Server está activo con la config final
curl -s -o /dev/null -w "Estado ms1: %{http_code}\n" \
  http://localhost:8001/sampleapp/index.jsp

# 6. Verificar espacio en disco disponible
df -h $DOMAIN_HOME

echo "Limpieza completada. Entorno listo para el siguiente laboratorio."
```

---

## 10. Resumen

En este laboratorio has recorrido el ciclo completo de optimización JVM para WebLogic 15 sobre Java 21:

| Actividad realizada | Resultado clave |
|---|---|
| Baseline de flags ergonómicos (jcmd) | Comprensión de los valores por defecto que aplica Java 21 automáticamente |
| G1GC con heap 512m bajo carga | Alta frecuencia de GC (~47 eventos), pausas irregulares hasta ~312ms |
| G1GC con heap 4g bajo carga | Reducción del 75 % en eventos GC, pausas medianas < 50ms |
| ZGC con heap 4g bajo carga | Pausas STW < 10ms, mayor overhead de CPU concurrente |
| Provocación de OOM y heap dump | Captura automática con `-XX:+HeapDumpOnOutOfMemoryError` |
| Análisis con Eclipse MAT | Identificación de objetos dominantes y árbol de retención |
| Documentación del baseline | Configuración reproducible en `setDomainEnv.sh` y modelo WDT YAML |

**Conclusión técnica:** Para la mayoría de cargas empresariales en WebLogic 15, **G1GC con heap fijo (`-Xms=-Xmx`) y una meta de pausa de 150–200ms** ofrece el mejor equilibrio. ZGC debe reservarse para SLAs con latencias de pausa sub-10ms donde el overhead de CPU adicional sea aceptable. La ergonomía de Java 21 es un punto de partida sólido, pero los entornos de producción siempre deben tener flags explícitos, logging de GC activo y dumps automáticos ante OOM.

### Recursos adicionales

- [Java 21 GC Tuning Guide — Oracle](https://docs.oracle.com/en/java/javase/21/gctuning/)
- [WebLogic 15 Performance and Tuning Guide — Oracle Docs](https://docs.oracle.com/en/middleware/standalone/weblogic-server/15/)
- [Eclipse MAT Documentation](https://help.eclipse.org/latest/index.jsp?topic=/org.eclipse.mat.ui.help/welcome.html)
- [JVM Unified Logging Reference (-Xlog)](https://docs.oracle.com/en/java/javase/21/docs/specs/man/java.html)
- [ZGC — The Z Garbage Collector (JEP 333/439)](https://openjdk.org/jeps/439)

---
