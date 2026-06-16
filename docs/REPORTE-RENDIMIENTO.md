# Reporte de Ejecución y Análisis — Pruebas de Rendimiento y Carga

**Asignatura:** Testing y Validación de Software — Unidad 5 (Validación y verificación de resultados)
**Programa:** Maestría en Ingeniería de Software — Universidad de La Sabana
**Profesor:** César Augusto Vega Fernández
**Estudiante:** Luis Mora — luismorlo@unisabana.edu.co
**Herramienta:** Apache JMeter 5.6.3 (modo no-GUI / CLI)
**Sistema bajo prueba (SUT):** Servicio *Registraduría* (Spring Boot 2.7.18, `POST /register`)
**Fecha de ejecución:** 16 de junio de 2026

---

## 1. Objetivo y alcance

Configurar escenarios de carga con JMeter, ejecutarlos de forma automatizada (sin interfaz
gráfica) e **interpretar las métricas de rendimiento** (tiempo de respuesta, throughput,
tasa de errores) del endpoint `POST /register` del servicio Registraduría, identificando
cuellos de botella y oportunidades de mejora.

El alcance se limita al único endpoint expuesto por el servicio (`POST /register`), que
aplica las reglas de negocio del registro de votantes y persiste en una base de datos
**H2 en memoria**.

---

## 2. Entorno de pruebas

| Componente | Detalle |
|------------|---------|
| Equipo | Laptop Intel® Core™ i5-7200U @ 2.50 GHz (2 núcleos / **4 hilos lógicos**) |
| Memoria RAM | 8 GB |
| Sistema operativo | Windows 10 Pro (64 bits) |
| JDK | OpenJDK 17.0.12 LTS |
| Generador de carga | Apache JMeter 5.6.3 (HTTP Request, HttpClient4, keep-alive) |
| SUT | Spring Boot 2.7.18 + Tomcat embebido + H2 en memoria |
| Topología | **SUT y generador de carga en el mismo equipo** (localhost) |

> ⚠️ **Caveat importante:** el generador de carga (JMeter) y la aplicación bajo prueba se
> ejecutan en la **misma máquina** de 4 hilos lógicos. Ambos procesos compiten por CPU y por
> el stack de red del sistema operativo. Esto es relevante para interpretar el escenario de
> estrés (sección 7).

---

## 3. Sistema bajo prueba (SUT)

- **Endpoint:** `POST /register`
- **Content-Type:** `application/json`
- **Cuerpo (ejemplo):**
  ```json
  { "name": "Votante_1_482910", "id": 73214, "age": 34, "gender": "FEMALE", "alive": true }
  ```
- **Respuestas posibles (texto plano, HTTP 200):** `VALID`, `DUPLICATED`, `UNDERAGE`, `DEAD`, `INVALID`
- **Entrada inválida (género no reconocido o JSON mal formado):** HTTP 400 (manejado por `ApiExceptionHandler`, **nunca 500**)

### Generación de datos de prueba (variabilidad realista)

El cuerpo de cada petición se genera dinámicamente en JMeter para simular tráfico heterogéneo:

| Campo | Generación | Efecto buscado |
|-------|------------|----------------|
| `name` | `Votante_${__threadNum}_${__Random(1,1000000)}` | Nombre único por hilo |
| `id`   | `${__Random(1,100000)}` | Rango acotado → genera **colisiones** (respuestas `DUPLICATED`) bajo carga |
| `age`  | `${__Random(15,90)}` | Mezcla de mayores (`VALID`) y menores (`UNDERAGE`) |
| `gender` | `${__groovy(...)}` → `FEMALE`/`MALE` | Género siempre válido |
| `alive` | `true` | Constante |

---

## 4. Diseño de escenarios

Se utilizó **un único plan de pruebas** (`jmeter/registraduria-loadtest.jmx`) parametrizado
por propiedades JMeter (`-Jthreads`, `-Jrampup`, `-Jduration`), ejecutado tres veces para
representar una progresión de carga creciente:

| # | Escenario | Hilos (usuarios) | Ramp-up | Duración | Propósito |
|---|-----------|------------------|---------|----------|-----------|
| 1 | **Smoke** | 1 | 1 s | 30 s | Línea base de latencia con un único usuario |
| 2 | **Carga** | 50 | 10 s | 60 s | Concurrencia esperada en operación normal |
| 3 | **Estrés** | 250 | 25 s | 60 s | Llevar el sistema (y el generador) más allá de lo esperado |

### Parámetros comunes a todos los escenarios

| Parámetro | Valor | Justificación |
|-----------|-------|---------------|
| **Think time** (Uniform Random Timer) | 200–500 ms entre peticiones | Simula el tiempo de reflexión de un usuario real y evita saturar el stack de red del cliente |
| Keep-Alive | Activado | Reutilización de conexiones TCP por hilo |
| Connect timeout | 5 000 ms | Falla rápida si no hay conexión |
| Response timeout | 15 000 ms | Marca como error las respuestas excesivamente lentas |
| Aserción de respuesta | Código HTTP ∈ {200, 400} | Cualquier 5xx o error de transporte cuenta como fallo |
| Scheduler | Por duración (no por iteraciones) | Carga sostenida durante el tiempo fijado |

---

## 5. Parámetros utilizados (resumen de ejecución)

```bash
# Comando base (repetido por escenario con distintas propiedades -J)
jmeter -n -t jmeter/registraduria-loadtest.jmx \
       -l results/<escenario>.jtl \
       -e -o results/<escenario>-report \
       -Jthreads=<N> -Jrampup=<R> -Jduration=<D>
```

- `-n` modo no-GUI (apropiado para ejecución automatizada / CI).
- `-l` archivo de resultados crudos (`.jtl`).
- `-e -o` genera el **dashboard HTML** con gráficas a partir del `.jtl`.

---

## 6. Métricas obtenidas (resultados reales)

> Fuente: `results/<escenario>-report/statistics.json` generado por JMeter.
> Total de muestras agregadas en los tres escenarios: **36 003** (sin contar la corrida
> de calibración descrita en la sección 7).

| Métrica | Smoke (1 usr) | Carga (50 usr) | Estrés (250 usr) |
|---------|--------------:|---------------:|-----------------:|
| Muestras (samples) | 69 | 5 757 | 30 168 |
| Errores | 0 | 0 | 20 177 |
| **% Error** | **0.00 %** | **0.00 %** | **66.88 %** |
| Tiempo medio (ms) | 69.9 | 121.4 | 19.1 \* |
| Mediana (ms) | 5 | 3 | 13 |
| Percentil 90 (ms) | 17 | 9 | 37 |
| Percentil 95 (ms) | 98 | 60 | 55 |
| Percentil 99 (ms) | 3 383 | 2 221 | 179 |
| Máximo (ms) | 3 383 | 13 495 | 3 157 |
| **Throughput (req/s)** | 2.4 | **98.5** | 506.4 \* |
| KB/s recibidos | 0.4 | 16.2 | 815.4 |

> \* En el escenario de estrés, las métricas de tiempo y throughput están **contaminadas**
> por los errores de transporte (ver sección 7): los fallos `BindException` retornan de
> forma casi instantánea, lo que **reduce artificialmente** la media y **infla** el
> throughput. No deben interpretarse como rendimiento real del SUT.

---

## 7. Interpretación de resultados

### 7.1 Tiempo de respuesta

- **Estado estable excelente.** La **mediana** se mantiene entre **3 y 13 ms** en todos los
  escenarios: la lógica de negocio (validaciones + inserción en H2 en memoria) es muy ligera.
- **Cold start / calentamiento de la JVM.** Los **máximos** elevados (3.4 s en smoke,
  13.5 s en carga) y el **percentil 99** alto corresponden a las **primeras peticiones**:
  arranque en frío de la JVM, compilación JIT, primera apertura del pool de conexiones y
  garbage collection inicial. Tras el calentamiento, los tiempos caen a milisegundos
  (la mediana de 3 ms en carga lo confirma).
- **Lección de medición:** la **media aritmética es engañosa** ante outliers; la **mediana y
  los percentiles** describen mucho mejor la experiencia típica del usuario.

### 7.2 Throughput

- Escala de forma coherente con la concurrencia útil: **2.4 → 98.5 req/s** entre smoke y
  carga. Con think-time de 200–500 ms, 50 usuarios producen ~98 req/s, valor consistente con
  la teoría (50 usuarios ÷ ~0.35 s por ciclo + latencia ≈ ~100 req/s).
- El throughput "aparente" del estrés (506 req/s) **no es real**: incluye 20 177 fallos de
  conexión instantáneos.

### 7.3 Tasa de errores

- **Smoke y Carga: 0 % de errores.** Ni un solo fallo en **5 826** peticiones reales.
- **No se registró ningún HTTP 500 en ninguno de los tres escenarios** (86 000+ peticiones
  acumuladas entre todas las corridas). **La aplicación nunca falló del lado del servidor:**
  manejó correctamente registros válidos, duplicados, menores de edad y entradas inválidas.

### 7.4 Identificación de cuellos de botella

El cuello de botella **no está en la aplicación**, sino en el **generador de carga / stack de
red del sistema operativo**, y se manifestó en dos fases:

**Fase A — Calibración fallida (sin think-time).**
La primera ejecución se hizo **sin tiempo de espera** entre peticiones. Cada hilo disparaba
miles de peticiones por segundo abriendo y cerrando sockets a gran velocidad. Resultado:

```
Non HTTP response code: java.net.BindException:
Address already in use: no further information   → 88–94 % de errores
```

Es el clásico **agotamiento de puertos efímeros en Windows**: los sockets quedan en estado
`TIME_WAIT` y el cliente no consigue asignar nuevos puertos locales. **Esto mide el límite
del sistema operativo, no el de la aplicación.**

**Fase B — Corrección (con think-time 200–500 ms).**
Al añadir un *Uniform Random Timer* que simula usuarios reales, la reutilización de
conexiones keep-alive se volvió efectiva y los errores **desaparecieron** en smoke y carga.
En estrés (250 usuarios concurrentes **en el mismo laptop de 4 hilos** que además ejecuta el
SUT) el agotamiento de puertos reaparece parcialmente (66.9 % de `BindException`), porque la
tasa de apertura de conexiones vuelve a superar la capacidad del stack de red local.

**Conclusión sobre el cuello de botella:** con la topología actual (cliente y servidor
co-ubicados en una máquina modesta) el factor limitante es la **capacidad de red/CPU del
equipo de pruebas**, no el SUT. El rendimiento real de la aplicación bajo 250 usuarios **no
pudo determinarse** porque el generador de carga se saturó antes.

---

## 8. Posibles mejoras

### Sobre el proceso de pruebas (para medir el límite real del SUT)
1. **Separar generador y SUT** en máquinas distintas (o contenedores con red dedicada).
2. **JMeter distribuido** (varios *workers*) para repartir la apertura de conexiones.
3. **Ejecutar el generador en Linux**, donde el rango de puertos efímeros es mayor y el
   `TIME_WAIT` se recicla más rápido; o ajustar en Windows `MaxUserPort` y
   `TcpTimedWaitDelay`.
4. Añadir un **escenario de resistencia (soak test)** de mayor duración para detectar fugas
   de memoria/conexiones, y un **spike test** para picos súbitos.
5. **Calentar la JVM** (peticiones previas no medidas) para que los percentiles altos no
   reflejen el arranque en frío.

### Sobre la aplicación (para producción)
6. Sustituir **H2 en memoria** por una base de datos real (PostgreSQL) y medir con
   persistencia realista; configurar un **pool de conexiones** (HikariCP) dimensionado.
7. Ajustar el **pool de hilos de Tomcat** (`server.tomcat.max-threads`) según la concurrencia
   objetivo.
8. Exponer métricas con **Actuator + Prometheus/Grafana** para correlacionar latencia con
   CPU, memoria y GC durante la prueba.
9. Definir y validar un **SLA/SLO** explícito (p. ej., p95 < 200 ms con 0 % de errores hasta
   N usuarios).

---

## 9. Cómo reproducir

Requisitos: JDK 17, Apache JMeter 5.6.3 y la aplicación Registraduría corriendo en
`localhost:8080`.

```powershell
# 1) Levantar el SUT (desde el proyecto TYVS-Pruebas_Integracion)
mvn spring-boot:run

# 2) Ejecutar los tres escenarios (desde este proyecto)
$JM="C:\tools\apache-jmeter-5.6.3\bin\jmeter.bat"
$scenarios = @(
  @{name="01-smoke";  threads=1;   rampup=1;  duration=30},
  @{name="02-carga";  threads=50;  rampup=10; duration=60},
  @{name="03-estres"; threads=250; rampup=25; duration=60}
)
foreach($s in $scenarios){
  $jArgs = @("-n","-t","jmeter\registraduria-loadtest.jmx",
             "-l","results\$($s.name).jtl","-e","-o","results\$($s.name)-report",
             "-Jthreads=$($s.threads)","-Jrampup=$($s.rampup)","-Jduration=$($s.duration)")
  & $JM @jArgs
}
```

---

## 10. Evidencia visual

Los **dashboards HTML interactivos** generados por JMeter (con gráficas de tiempo de
respuesta a lo largo del tiempo, distribución de percentiles, throughput, etc.) están
versionados en:

- `results/01-smoke-report/index.html`
- `results/02-carga-report/index.html`
- `results/03-estres-report/index.html`

Para verlos, abrir cualquiera de esos `index.html` en un navegador. Las capturas de pantalla
del proceso de ejecución y de los dashboards se encuentran en
[`docs/capturas/`](capturas/) (ver `docs/capturas/README.md` para la guía de capturas).

---

## 11. Conclusión

La aplicación Registraduría demostró ser **funcionalmente robusta y de muy baja latencia**:
**0 errores de servidor** en más de 86 000 peticiones y tiempos de respuesta típicos de
**3–13 ms**. El ejercicio también dejó una lección metodológica central de las pruebas de
rendimiento: **distinguir el cuello de botella del sistema bajo prueba del cuello de botella
del entorno de pruebas**. Los errores observados bajo alta concurrencia provienen del
agotamiento de puertos del cliente (Windows) y de la co-ubicación de generador y SUT en un
equipo modesto, no de la aplicación. Para medir el límite real del servicio se requiere
separar el generador de carga del SUT, según las mejoras propuestas.
