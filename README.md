# Analysis-of-Intracellular-Neural-Recordings-under-CW-NIR-980-nm-Laser-Illumination
Python code for analysis of patch clamp recordings from mouse brain slices under continuous-wave near-infrared (CW-NIR) laser stimulation at 976 nm.

**TFG — Grado en Ingeniería Biomédica**  
Escuela Politécnica Superior, Universidad Autónoma de Madrid  
Autora: Claudia Riesco Sánchez   
Mayo 2026

## Descripción del proyecto

Este repositorio contiene el código de análisis del TFG sobre neuromodulación con láser infrarrojo de onda continua (CW-NIR) en rodajas de cerebro de ratón. Se estudia cómo la iluminación con un láser CW-NIR de **976 nm** (Thorlabs BL976-PAG900) modifica la actividad eléctrica de neuronas piramidales de la **corteza somatosensorial primaria (barrel cortex)**, registradas mediante patch-clamp whole-cell en modo fijación de corriente.

Cada registro contiene una señal de Voltaje vs. Tiempo que sigue la estructura:
```
[0 – ~150 s] Control  →  [~150 – ~300 s] Láser  →  [~300 – 480 s] Recuperación
```

El inicio y fin de la fase de láser se detectan automáticamente a partir del canal TTL sincronizado. Para garantizar comparabilidad entre archivos, todas las métricas se calculan sobre ventanas estandarizadas de **150 s por fase**.

### Archivos analizados, correspondientes a cada registro

| Archivo | Intensidad láser |
|---------|-----------------|
| 8       | 500 mA          |
| 9       | 700 mA          |
| 10      | 900 mA          |
| 11      | 1000 mA         |
| 12      | 1100 mA         |
| 13      | 1200 mA         |
| 14      | 1400 mA         |

> Los archivos `.abf` originales no están incluídos en el repositorio, por lo que, si se quisiera usar estos códigos, se tendría que cambiar la ruta de mi directorio.

## Métodos implementados

### 1. Segmentación de fases
- Detección automática de flancos del canal TTL (subida = inicio láser; bajada = fin).
- Ventana estandarizada de **150 s por fase** a partir del momento de transición.
- Resultado: diccionario `archivos_info` con `(t_inicio, t_fin)` para cada fase y archivo.

### 2. Detección de pulsos de corriente
- Umbral = `mediana(Vm) + 10 mV`. La mediana es robusta para detectar el potencial de reposo frente a los picos de AP porque la neurona pasa ~70% del tiempo en reposo.
- Cruce hacia arriba → inicio del pulso; cruce hacia abajo → fin.
- Resultado: entre 44 y 45 pulsos por fase en cada archivo.

### 3. Detección de potenciales de acción
- Uso de find_peaks de SciPy con umbral en 0 mV y distancia mínima de 50 ms entre picos (período refractario absoluto).
- Permite capturar ráfagas de hasta 4 APs por pulso sin dobles detecciones.

### 4. Forma del AP (waveform promedio)
- Ventanas: **3 ms antes** y **7 ms después** del pico.
- Alineación al **pico** (`np.argmax` dentro del recorte) para corregir desplazamientos de 1–2 muestras.
- Corrección de baseline: se resta la media del voltaje en el primer milisegundo del recorte (todos los APs quedan referenciados a 0 mV).
- Se calcula la media ± SD punto a punto de todos los APs de cada fase.

### 5. Cinética del potencial de acción — pendientes dV/dt
- La derivada dV/dt (mV/ms) se calcula con np.diff.
- El reposo local de cada AP se estima como la media del voltaje en la ventana [−8 ms, −5 ms] antes del pico.
- La semiamplitud se calcula de forma local para cada AP a partir de ese reposo y el voltaje pico.

Pendiente de despolarización: máximo de dV/dt en la ventana de 10 ms antes del pico, evaluado en el cruce de la semiamplitud hacia arriba.
Pendiente de repolarización: mínimo de dV/dt en la ventana de 3 ms después del pico, evaluado en el cruce de la semiamplitud hacia abajo.
> Las ventanas son asimétricas porque la despolarización es más lenta que la repolarización, que ocurre en los primeros milisegundos tras el pico.

### 6. Amplitud del AP
```
A = V_pico − V_rep_local
```
donde `V_rep_local` es la media del voltaje en la misma ventana [−8 ms, −5 ms] usada para las pendientes.

### 7. Hiperpolarización post-AP (AHP)
```
AHP = V_min − V_rep_local
```
- `V_min`: mínimo de voltaje en ventana de **50 ms tras el pico**, excluyendo los primeros 3 ms (para evitar la repolarización rápida).
- Solo se analizan los APs que caen entre el **2º y el 5º sexto** de la duración de cada pulso, excluyendo los del inicio y del final (donde la membrana aún está en tránsito por la inyección de corriente).

### 8. Latencia al primer AP
Dado que el láser en muchos casos bloquea el primer disparo y lo desplaza a un segundo intento dentro del mismo pulso, la latencia se analiza clasificando cada pulso en tres categorías:

 Slot 1 --> Latencia < 100 ms (disparo en el primer intento) |
 Slot 2 --> Latencia ≥ 100 ms (disparo retrasado al segundo intento) |
 Silente --> Sin potencial de acción en todo el pulso |

Solo se incluyen en el análisis los pulsos que generan al menos un AP.

### 9. Entropía — ExSEnt
Basado en Kamali, Baroni & Varona (2025) — [arXiv:2509.07751](https://arxiv.org/abs/2509.07751).

La señal se segmenta en intervalos delimitados por extremos locales. De cada segmento se extraen:
- **D_i**: duración temporal del segmento
- **A_i**: cambio neto de amplitud entre extremos consecutivos

Umbral de tolerancia para evitar segmentación por ruido:
```
θ = λ · IQR({Δx_k})
```

Tres métricas de entropía calculadas con Sample Entropy:
- `H_D` = SampEnt sobre {D_i} — irregularidad temporal
- `H_A` = SampEnt sobre {A_i} — variabilidad de amplitud
- `H_DA` = SampEnt sobre {(D_i, A_i)} — complejidad conjunta


## Instalación

```bash
pip install pyabf numpy scipy matplotlib pandas
```

Para ExSEnt, clonar el repositorio original y seguir sus instrucciones de instalación.



> Claudia Riesco Sánchez. *Automatización de la estimulación óptica y del análisis de su efecto en registros de rodajas de cerebro de ratones.* TFG, Grado en Ingeniería Biomédica, Escuela Politécnica Superior, Universidad Autónoma de Madrid, mayo 2026.
