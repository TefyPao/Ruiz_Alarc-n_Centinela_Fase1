# Propuesta de Datos — Proyecto Centinela, Fase 1

**Autora:** Estefanny Ruíz González_Miguel Alarcón Ojeda

**Curso:** Profundización I — Redes Neuronales / Deep Learning 

**Maestría en Ciencia de Datos — Universidad Santo Tomás**

---

## 1. Problema y cliente

El **poder adquisitivo real** del salario mínimo legal vigente (SMLV) en Colombia se erosiona cuando la inflación crece más rápido que los ajustes nominales del salario. Esta pérdida no siempre es visible de inmediato para quienes dependen del SMLV, y suele reconocerse solo después de haber afectado el consumo de los hogares.

Este proyecto propone un **sistema de alerta temprana ("Centinela")** que prediga si el poder adquisitivo del SMLV está **en riesgo de caer** en un periodo dado, a partir de variables macroeconómicas observables con antelación (inflación, tasa de cambio, salario nominal).

**Posibles Cliente:** un analista de política pública o de un sindicato/gremio laboral que necesita anticipar —no solo describir después del hecho— los periodos en que el salario mínimo perderá poder de compra, para sustentar decisiones de ajuste salarial o de política económica.

Este problema es la base del proyecto de grado de los autores (línea de investigación en poder adquisitivo del SMLV), aquí adaptado al formato de predicción de evento de riesgo que exige esta fase.

## 2. Las dos modalidades previstas para el sistema completo

- **Modalidad aplanada en esta Fase 1 — Serie temporal económica:** variables macroeconómicas mensuales (IPC de Bogotá, TRM, salario nominal, rezagos de estas series) tratadas como vector de características tabulares por periodo.
- **Modalidad prevista para la Fase 2 — Serie temporal nativa:** las mismas variables, sin aplanar, procesadas con una arquitectura recurrente (RNN/LSTM/GRU) que explote la dependencia secuencial mes a mes.

*Nota:* a diferencia del escenario de referencia (imagen + clima), este proyecto usa **dos representaciones de la misma fuente de datos** (tabular vs. secuencial), no dos fuentes distintas. Esta decisión está en consulta con el docente; en caso de que se requiera una segunda modalidad independiente (p. ej., imágenes satelitales de luminosidad nocturna como proxy de actividad económica regional), se ajustará en la versión final de esta propuesta.

## 3. Fuentes de datos

| Fuente | Variable | Enlace | Licencia / acceso |
|---|---|---|---|
| DANE | IPC Bogotá, variación anual | https://www.dane.gov.co | Datos abiertos, uso público |
| Banco de la República | TRM (tasa de cambio representativa del mercado) | https://www.banrep.gov.co | Datos abiertos, uso público |
| Ministerio de Trabajo / DANE | Salario mínimo legal vigente (histórico anual) | https://www.dane.gov.co | Datos abiertos, uso público |

Estos datos no provienen de Kaggle ni de ningún *mirror* de Kaggle. La base fue **consolidada manualmente por los autores** a partir de las fuentes oficiales citadas, como parte de su proyecto de grado.

## 4. Por qué no es un reto ya resuelto

No existe un dataset preempaquetado ni una competencia pública que combine estas series ya etiquetadas con un evento de riesgo del poder adquisitivo del SMLV colombiano. La consolidación mensual de IPC, TRM y salario nominal en una sola tabla, así como la definición del evento de riesgo, es trabajo original de los autores, desarrollado en el marco de su proyecto de grado y anterior a este curso.

## 5. Descripción del conjunto de datos (EDA preliminar)

- **Periodo:** enero de 2000 a diciembre de 2026 (frecuencia mensual). *Los últimos meses de 2026 corresponden a valores proyectados, no observados; se declarará y tratará esta limitación en el notebook.*
- **Tamaño:** 324 registros mensuales, 8 variables.
- **Valores faltantes:** mínimos — 7 en TRM y 12 en las variables de variación anual (esperado, por ausencia de año base de comparación en los primeros 12 meses).
- **Variable objetivo propuesta:** `riesgo = 1` si la variación anual del salario mínimo real (`smlv_real_var_anual`) es negativa; `riesgo = 0` en caso contrario.
- **Balance de clases:** **fuertemente desbalanceado** — 30 de 324 registros (≈9.3 %) corresponden a la clase de riesgo. Este hallazgo del EDA condiciona la evaluación del modelo: la exactitud global no será la métrica principal; se priorizarán F1 y *recall* de la clase de riesgo, y se considerará ponderar la pérdida o aplicar remuestreo.
- **Variables de entrada (decididas por el EDA):** el análisis mostró que las variables en su **nivel crudo** (IPC, TRM, salario nominal) tienen correlación casi nula con la etiqueta (±0.06), pero sus features de **dinámica** sí aportan señal: la variación mensual y trimestral del IPC (`ipc_mom`, `ipc_mom3`) y la **brecha entre el crecimiento nominal del salario y la inflación** (`brecha_yoy = var. anual del salario nominal − var. anual del IPC`) alcanzan correlaciones de hasta ±0.37. Se selecciona este conjunto por tener señal y fundamento económico directo (el salario real cae cuando el nominal crece menos que la inflación). La TRM se descarta por baja señal (~0.02).
- **Exclusión por fuga de datos (*data leakage*):** se excluyen explícitamente `smlv_real`, `smlv_real_var_anual` **y `poder_adq_index`** como variables de entrada. Las dos primeras son la base de la etiqueta; la tercera es `smlv_real` reescalado a base 100 (verificado en el EDA), por lo que su inclusión filtraría la respuesta al modelo.
- **Partición:** por tratarse de una serie temporal, la partición train/val/test será **cronológica 70 / 15 / 15** (primeros años para entrenamiento, tramo intermedio para validación, años recientes para prueba), nunca aleatoria, para evitar que el modelo "vea" información futura. El EDA reveló que los casos de riesgo se concentran en periodos históricos, dejando el tramo de prueba más reciente **sin casos positivos**; por ello las métricas de detección (matriz de confusión, *recall* de la clase de riesgo) se reportarán sobre el tramo de **validación**, que sí contiene positivos, conservando el orden cronológico.

## 6. Riesgos éticos

- **Costo de los falsos negativos:** no anticipar una caída real del poder adquisitivo dejaría a los tomadores de decisión sin tiempo de reacción, perpetuando la pérdida de bienestar de los hogares que dependen del SMLV.
- **Costo de los falsos positivos:** alertar sobre una caída que no ocurre podría presionar decisiones de ajuste salarial innecesarias o generar alarma social/sindical sin fundamento.
- **Representatividad:** el piloto usa solo Bogotá; los resultados no son directamente generalizables a las 13 áreas metropolitanas del DANE ni al ámbito rural, donde la dinámica de precios puede diferir.
- **Uso indebido:** el modelo no debe usarse como única base para decisiones de política salarial; es una herramienta de apoyo, no un sustituto del análisis experto, dado el tamaño reducido de la muestra (~290 registros útiles) y el fuerte desbalance de clases.

---

*Esta propuesta está pendiente de aprobación por el docente. Una vez aprobada, se referenciará como anexo del Notebook, el Informe técnico-ético y la Bitácora de IA.*
