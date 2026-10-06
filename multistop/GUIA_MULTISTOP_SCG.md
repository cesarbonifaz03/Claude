# Guía: Multi-Stop Estimation en Supply Chain Guru

Configuración probada el 10/06/2026 con el CEDIS GDL (121 tiendas, abril 2026). Resultado: estimación sin fallas, 429 filas en la matriz de costos, transporte de $1.24 M/mes, $2.31–7.23 por bulto y 6.9 camiones equivalentes (real: 8).

Archivos en esta carpeta:

| Archivo | Para qué |
|---|---|
| `plantilla_multistop_SCG.xlsx` | Plantilla con el ejemplo funcional + hojas LEEME, CHECKLIST y CALIBRACION |
| `calibracion_A_dias_programar_0.5.xlsx` | Prueba de calibración: rutas menos eficientes (más distancia entre paradas) |
| `calibracion_B_ventana_entrega_5h.xlsx` | Prueba de calibración: ventana de entrega de 5 h en tienda |

> Antes de importar la plantilla, borra las hojas LEEME, CHECKLIST y CALIBRACION (o no las selecciones al importar).

## 1. Por qué fallaba (y cómo no repetirlo)

1. **Porcentajes en escala 0–100.** `Capacity Utilization`, `Time Utilization` y `Vehicle Speed Factor` de `AC_PseudoRoutingVehicleCosts` son porcentajes. Con 1, el camión tenía 1 % de capacidad, tiempo y velocidad, y su alcance (`Vehicle Range`) era casi 0.
2. **IDs de unidades de medida movidos.** Coupa: *"Multi-Stop Estimation… expects the row ID values to match those defined when the model was first created (EA has ID = 1)… This can occur if you export the table to Excel, then re-import it."* Reimportar `UnitOfMeasure` cambió EA a ID 61 → "No matrix demand data".

Fórmula de `Vehicle Range` (comprobada contra 3 corridas):
`(Horas turno − descanso − descarga fija) × TimeUtil% × SpeedFactor% × 80 km/h ÷ 2`

## 2. Configuración tabla por tabla

| Tabla | Campo | Valor |
|---|---|---|
| AC_TransportRegion | Name | Nombre propio (no "Default Region"); sin overrides |
| Customers | Transportation Region | Región de cada tienda; lat/long válidas |
| CustomerDemand | Quantity | Bultos **por entrega** |
| | Occurrences | Entregas en el periodo |
| | Order Frequency | Obligatorio si Occurrences > 1 (ej. `3 DAY`) |
| AC_PseudoRoutingVehicleCosts | Capacity / Time Utilization, Speed Factor | 100 (ajustar en escala 0–100) |
| | Capacity (Quantity) | Capacidad real del camión |
| | Is Owned Vehicle | Sí → usa costo/día, chofer, combustible, mantenimiento; No → costo por ruta, por parada y por km |
| Modes | Fixed Shipment Cost | Nombre del vehículo de la tabla anterior |
| | Shipment Rule | `Treat Shipment Cost as Fixed` |
| | Allowable Products | Producto(s) |
| TG_Policies | Mode | El Mode de Multi-Stop |
| | Shipment Size | = Quantity por entrega de esa tienda |
| | Fixed / Variable Cost | **Vacíos** |

Opciones de Network Optimization → Multi-Stop: *Run Multi-Stop Estimation with NO* encendido, overrides de región apagados, horizonte = periodo, correr **local** (no Cloud Solve).

## 3. Calibración

Objetivo: que camiones y km del modelo se parezcan a la operación real.

| Palanca | Efecto documentado |
|---|---|
| Number Of Days To Schedule Transportation | Más alto → menor distancia entre paradas → rutas más eficientes |
| Delivery Time Window | Limita horas entre primera y última parada → menos paradas por ruta |
| Stem Distance Adjustment Percentage | Ajusta la distancia CEDIS–zona (semántica exacta por confirmar con Coupa) |
| Inter Drop Distance (vehículo o región) | Fija la distancia entre paradas directamente |

Base: 6.9 camiones y 27,000 km/mes contra ~8 camiones y ~34,600 km/mes reales (estimado de 1,000 km/semana/camión). El modelo sale ~20 % más eficiente que la realidad; las corridas A y B prueban dos formas de acercarlo. Anota los resultados en la hoja CALIBRACION.

Cómo medir en la salida:
- Camiones = suma de `Number Of Vehicles` en `OO_PseudoRoutingSummary`.
- Km/mes = suma de `Transport Distance × Number Of Movements`.

## 4. Escalar al proyecto real (varios CEDIS)

Recomendaciones (inferidas de la documentación; validar en la primera corrida):
- Una **región por zona geográfica** de tiendas; la documentación calcula distancia entre paradas, área y costos por región, Mode y periodo.
- Si cada CEDIS tiene flota distinta, crea un **vehículo y un Mode por CEDIS** y asígnalos en sus políticas.
- Cada política CEDIS→tienda busca en la matriz usando la distancia del carril, así que la optimización puede comparar CEDIS con costos de varias paradas.
- Con varios periodos, la demanda por entrega se define por periodo (Quantity × Occurrences en cada uno).
- Mantén la regla de oro: modelo nuevo, sin reimportar tablas del sistema.

## 5. Pendientes de datos del modelo GDL

- 11 tiendas con coordenadas repetidas (marcadas en Customers.Notes).
- 10 distancias en línea recta (haversine) en TG_Policies.
- Confirmar la frecuencia real de entrega (se usó lunes y jueves = 9 entregas/mes).
- Revisar si la flota real es 8 o 12 y los km reales por camión.
