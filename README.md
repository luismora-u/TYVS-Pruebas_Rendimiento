# TYVS-Pruebas_Rendimiento — Pruebas de Rendimiento y Carga (JMeter)

Pruebas de **rendimiento y carga** con **Apache JMeter** sobre el servicio **Registraduría**
(Spring Boot, endpoint `POST /register`), correspondientes a la **Unidad 5 — Validación y
verificación de resultados** de la asignatura *Testing y Validación de Software*
(Maestría en Ingeniería de Software, Universidad de La Sabana).

> **Autor (entrega individual):** ver [`integrantes.txt`](integrantes.txt).

---

## 1. Contenido del repositorio

```
TYVS-Pruebas_Rendimiento/
├── jmeter/
│   └── registraduria-loadtest.jmx     # Plan de pruebas JMeter (parametrizable)
├── results/
│   ├── 01-smoke.jtl   + 01-smoke-report/    # resultados crudos + dashboard HTML
│   ├── 02-carga.jtl   + 02-carga-report/
│   └── 03-estres.jtl  + 03-estres-report/
├── docs/
│   ├── REPORTE-RENDIMIENTO.md          # Reporte de ejecución y análisis (principal)
│   └── capturas/                       # Evidencia visual
├── integrantes.txt
└── README.md
```

---

## 2. El plan de pruebas (`.jmx`)

Un único plan parametrizable por **propiedades JMeter**, reutilizado en los tres escenarios:

| Elemento | Detalle |
|----------|---------|
| HTTP Request Defaults | `http://${host}:${port}` (por defecto `localhost:8080`) |
| Header Manager | `Content-Type: application/json` |
| Thread Group | `threads`, `rampup`, `duration` configurables por `-J` |
| Think time | Uniform Random Timer 200–500 ms (simula usuario real) |
| HTTP Sampler | `POST /register` con cuerpo JSON generado dinámicamente |
| Aserción | Código HTTP ∈ {200, 400} (cualquier 5xx/error de transporte = fallo) |

Propiedades disponibles: `-Jthreads`, `-Jrampup`, `-Jduration`, `-Jhost`, `-Jport`,
`-Jthinkbase`, `-Jthinkrange`.

---

## 3. Escenarios ejecutados

| Escenario | Usuarios | Ramp-up | Duración | % Error | Mediana | Throughput |
|-----------|---------:|--------:|---------:|--------:|--------:|-----------:|
| Smoke  | 1 | 1 s | 30 s | 0.00 % | 5 ms | 2.4 req/s |
| Carga  | 50 | 10 s | 60 s | 0.00 % | 3 ms | 98.5 req/s |
| Estrés | 250 | 25 s | 60 s | 66.88 %\* | 13 ms | 506 req/s\* |

\* El error en estrés es **`BindException` del cliente** (agotamiento de puertos efímeros de
Windows), **no un fallo de la aplicación**. Ver el análisis completo en
[`docs/REPORTE-RENDIMIENTO.md`](docs/REPORTE-RENDIMIENTO.md).

**Resultado clave: 0 errores HTTP 500 en más de 86 000 peticiones** → la aplicación nunca
falló del lado del servidor.

---

## 4. Cómo ejecutar

Requisitos: **JDK 17**, **Apache JMeter 5.6.3** y la app Registraduría corriendo en
`localhost:8080` (proyecto `TYVS-Pruebas_Integracion`, `mvn spring-boot:run`).

```powershell
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

Los dashboards quedan en `results/<escenario>-report/index.html` (abrir en navegador).

---

## 5. Reporte de análisis

El documento principal del entregable es **[`docs/REPORTE-RENDIMIENTO.md`](docs/REPORTE-RENDIMIENTO.md)**,
que incluye: descripción de escenarios, parámetros, métricas reales, interpretación
(tiempo de respuesta, throughput, tasa de errores), identificación de cuellos de botella
y propuestas de mejora.

---

## Créditos

Dominio Registraduría basado en el material del profesor **César Augusto Vega Fernández** —
Universidad de La Sabana. Solución de pruebas de rendimiento desarrollada por el estudiante.
