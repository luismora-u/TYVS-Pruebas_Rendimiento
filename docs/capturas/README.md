# Evidencia visual — Capturas de pantalla

Coloca aquí las capturas del proceso de ejecución y de los dashboards de JMeter.

## Capturas sugeridas

1. **Ejecución en consola** — la salida `summary =` de los tres escenarios (smoke/carga/estrés).
2. **Dashboard — Carga (50 usuarios):** abrir `results/02-carga-report/index.html` y capturar:
   - *APDEX / Statistics* (tabla resumen, 0 % de error).
   - *Response Times Over Time* (gráfica de latencia en el tiempo).
   - *Transactions per Second* (throughput).
3. **Dashboard — Estrés (250 usuarios):** abrir `results/03-estres-report/index.html` y capturar
   la tabla de estadísticas mostrando el ~66.9 % de error (`BindException`).
4. **Comparación de percentiles** entre escenarios (gráfica *Response Time Percentiles*).

## Cómo abrir un dashboard

```powershell
Start-Process "results\02-carga-report\index.html"
```

Nombrar las capturas de forma descriptiva, por ejemplo:
`01-consola-ejecucion.png`, `02-carga-statistics.png`, `02-carga-response-time.png`,
`03-estres-statistics.png`.
