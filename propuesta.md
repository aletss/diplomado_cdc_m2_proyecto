# Predicción de Probabilidad de Retraso en Vuelos
## Propuesta de Proyecto de Ciencia de Datos

> **Fuente de datos:** Bureau of Transportation Statistics — Marketing Carrier On-Time Performance  
> **Dataset:** `data.csv`  
> **Última actualización del documento:** 2025/01/31/

---

## Tabla de Contenidos

1. [Resumen Ejecutivo](#1-resumen-ejecutivo)
2. [Definición del Problema](#2-definición-del-problema)
3. [Diccionario de Datos](#3-diccionario-de-datos)
4. [Variables Predictoras Propuestas](#4-variables-predictoras-propuestas)
5. [Variable Objetivo](#5-variable-objetivo)
6. [Arquitectura del Modelo](#6-arquitectura-del-modelo)
7. [Estrategia de Validación](#7-estrategia-de-validación)
8. [Uso Operacional](#8-uso-operacional)
9. [Glosario Operacional](#9-glosario-operacional)

---

## 1. Resumen Ejecutivo

Este proyecto tiene como objetivo construir un sistema de predicción de retrasos de vuelos con horizonte de un mes, utilizando información histórica operacional de aeropuertos, aerolíneas y rutas. El modelo busca transformar un problema reactivo —atender pasajeros ya afectados por retrasos— en uno **proactivo**, anticipando qué vuelos del mes siguiente tienen mayor probabilidad de sufrir retrasos y permitiendo preparar estrategias de comunicación, asignación de personal y configuración de mensajes en aplicaciones antes de que ocurran.

---

## 2. Definición del Problema

### 2.1 Escenario de Predicción

Dado el calendario de vuelos programados del **mes siguiente** (con horarios, rutas y aerolíneas ya confirmados), determinar para cada vuelo:

> **¿Cuál es la probabilidad de que este vuelo tenga un retraso en llegada superior a 15 minutos?**

### 2.2 Variable Objetivo

| Tipo | Nombre | Definición |
|------|--------|------------|
| Binaria (clasificación) | `TARGET_DELAYED_15` | 1 si `ARR_DELAY > 15` min, 0 en caso contrario |
| Continua (regresión) | `TARGET_ARR_DELAY` | Valor en minutos de `ARR_DELAY` (puede ser negativo = llegada temprana) |

> El umbral de **15 minutos** es el estándar oficial del Departamento de Transporte de EE.UU. (DOT) para definir un vuelo como retrasado.

### 2.3 Restricción Temporal Crítica

Las variables predictoras deben construirse **exclusivamente con datos históricos anteriores al mes de predicción**. No se puede usar ninguna variable que requiera conocer lo que ocurrió durante el vuelo mismo.

**Información disponible al momento de predicción:**
- Horario programado completo (CRS_DEP_TIME, CRS_ARR_TIME, CRS_ELAPSED_TIME)
- Histórico de comportamiento de aerolínea, ruta y aeropuerto hasta el mes anterior
- Temporalidad conocida del vuelo futuro (día, hora, mes, festivos)

**Información NO disponible al momento de predicción:**
- DEP_TIME, ARR_TIME, DEP_DELAY (ocurren durante el vuelo)
- TAXI_OUT, TAXI_IN, AIR_TIME (tiempo real de operación)
- Causas de retraso (CARRIER_DELAY, WEATHER_DELAY, etc.)

### 2.4 Propósito del Modelo

```
Entrada:   Horario de vuelos del mes siguiente
           + Histórico operacional hasta el mes anterior

Salida:    Score de probabilidad de retraso por vuelo (0.0 – 1.0)
           + Categoría de riesgo: BAJO / MEDIO / ALTO / CRÍTICO

Uso:       Priorizar acciones preventivas de comunicación,
           asignación de personal y configuración de mensajes
           en aplicaciones de usuarios
```

---

## 3. Diccionario de Datos

### 3.1 Convenciones

| Símbolo | Significado |
|---------|-------------|
| PREDICTOR | Disponible antes del vuelo — puede usarse como predictor |
| TIEMPO REAL | Disponible durante o justo después del despegue |
| POST-HOC | Solo disponible después del vuelo — no usar como predictor |
| LLAVE | Variable de identificación o llave |
| OBJETIVO | Candidata a variable objetivo |

---

### 3.2 Variables de Identificación y Tiempo

| Campo | Nombre Completo | Descripción | Tipo de Dato | Disponibilidad |
|-------|-----------------|-------------|--------------|----------------|
| `FL_DATE` | Flight Date | Fecha del vuelo en formato `YYYY-MM-DD` | Fecha | LLAVE temporal |
| `MKT_UNIQUE_CARRIER` | Marketing Airline Network | Código único de la aerolínea de marketing. Cuando el mismo código fue usado por múltiples aerolíneas, se añade sufijo numérico para versiones anteriores (ej: `PA`, `PA(1)`). Recomendado para análisis multi-año. | Texto (2–3 chars) | PREDICTOR |
| `MKT_CARRIER_FL_NUM` | Flight Number (Marketing) | Número de vuelo asignado por la aerolínea de marketing | Entero | LLAVE |
| `OP_UNIQUE_CARRIER` | Operating Airline | Código único de la aerolínea operadora. Puede diferir de `MKT_UNIQUE_CARRIER` en vuelos codeshare. | Texto (2–3 chars) | PREDICTOR |
| `TAIL_NUM` | Tail Number | Matrícula física de la aeronave. Permite rastrear el historial operacional del avión y detectar el vuelo inbound previo para modelar el efecto cascada. | Texto | PREDICTOR (llave para features de cascada) |

---

### 3.3 Variables de Aeropuerto de Origen

| Campo | Nombre Completo | Descripción | Tipo de Dato | Disponibilidad |
|-------|-----------------|-------------|--------------|----------------|
| `ORIGIN_AIRPORT_ID` | Origin Airport ID | Identificador único DOT del aeropuerto de origen. Estable en el tiempo aunque cambie el código IATA. Recomendado para análisis multi-año. | Entero | LLAVE |
| `ORIGIN_AIRPORT_SEQ_ID` | Origin Airport Sequence ID | Identificador único DOT del aeropuerto de origen en un punto específico del tiempo. Captura cambios en atributos como nombre o coordenadas. | Entero | LLAVE temporal |
| `ORIGIN_CITY_MARKET_ID` | Origin City Market ID | Identificador de mercado de ciudad DOT. Agrupa múltiples aeropuertos que sirven a la misma ciudad (ej: LAX + BUR = Los Ángeles). | Entero | PREDICTOR |
| `ORIGIN` | Origin Airport Code | Código IATA del aeropuerto de origen (ej: `DFW`, `JFK`, `ORD`) | Texto (3 chars) | PREDICTOR clave |
| `ORIGIN_CITY_NAME` | Origin City Name | Nombre de la ciudad del aeropuerto de origen | Texto | Descriptivo |

---

### 3.4 Variables de Aeropuerto de Destino

| Campo | Nombre Completo | Descripción | Tipo de Dato | Disponibilidad |
|-------|-----------------|-------------|--------------|----------------|
| `DEST_AIRPORT_ID` | Destination Airport ID | Identificador único DOT del aeropuerto de destino. Análogo a `ORIGIN_AIRPORT_ID`. | Entero | LLAVE |
| `DEST_AIRPORT_SEQ_ID` | Destination Airport Sequence ID | Identificador único DOT del aeropuerto de destino en un punto específico del tiempo. | Entero | LLAVE temporal |
| `DEST_CITY_MARKET_ID` | Destination City Market ID | Identificador de mercado de ciudad DOT para el destino. Permite consolidar aeropuertos del mismo mercado. | Entero | PREDICTOR |
| `DEST` | Destination Airport Code | Código IATA del aeropuerto de destino | Texto (3 chars) | PREDICTOR clave |
| `DEST_CITY_NAME` | Destination City Name | Nombre de la ciudad del aeropuerto de destino | Texto | Descriptivo |

---

### 3.5 Variables de Desempeño en Salida

| Campo | Nombre Completo | Descripción | Tipo de Dato | Rango | Disponibilidad |
|-------|-----------------|-------------|--------------|-------|----------------|
| `CRS_DEP_TIME` | CRS Departure Time | Hora programada de salida en hora local, formato `hhmm`. Ejemplo: `1430` = 2:30 PM | Entero | 0 – 2359 | PREDICTOR — captura efecto de hora del día |
| `DEP_TIME` | Actual Departure Time | Hora real en que el avión se aleja de la puerta en hora local (hhmm). Nulo si vuelo cancelado. | Entero | 0 – 2359 | POST-HOC |
| `DEP_DELAY` | Departure Delay | Diferencia en minutos entre hora real y programada de salida. Valores negativos = salida temprana. Nulo si cancelado. | Float | Sin restricción | POST-HOC |
| `TAXI_OUT` | Taxi Out Time | Tiempo en minutos desde que el avión se aleja de la puerta hasta el despegue (`WHEELS_OFF – DEP_TIME`). Nulo si cancelado. | Float | >= 0 | POST-HOC (media histórica usable como PREDICTOR) |
| `WHEELS_OFF` | Wheels Off Time | Hora exacta en que las ruedas dejan el suelo (despegue), en hora local (hhmm). | Entero | 0 – 2359 | POST-HOC |

---

### 3.6 Variables de Desempeño en Llegada

| Campo | Nombre Completo | Descripción | Tipo de Dato | Rango | Disponibilidad |
|-------|-----------------|-------------|--------------|-------|----------------|
| `WHEELS_ON` | Wheels On Time | Hora exacta en que las ruedas tocan la pista (aterrizaje), en hora local (hhmm). | Entero | 0 – 2359 | POST-HOC |
| `TAXI_IN` | Taxi In Time | Tiempo en minutos desde el aterrizaje hasta que el avión llega a la puerta. | Float | >= 0 | POST-HOC (media histórica usable como PREDICTOR) |
| `CRS_ARR_TIME` | CRS Arrival Time | Hora programada de llegada en hora local (hhmm). | Entero | 0 – 2359 | PREDICTOR — captura efecto de hora de llegada |
| `ARR_TIME` | Actual Arrival Time | Hora real de llegada a la puerta en hora local (hhmm). Nulo si cancelado. | Entero | 0 – 2359 | POST-HOC |
| `ARR_DELAY` | Arrival Delay | **Variable objetivo principal.** Diferencia en minutos entre hora real y programada de llegada. Negativo = llegada temprana. Nulo si cancelado o desviado. | Float | Sin restricción | OBJETIVO |

---

### 3.7 Variables de Cancelación y Desvío

| Campo | Nombre Completo | Descripción | Valores posibles | Disponibilidad |
|-------|-----------------|-------------|------------------|----------------|
| `CANCELLED` | Cancelled Flight Indicator | Indicador binario de vuelo cancelado | `1` = Cancelado, `0` = Operado | OBJETIVO binario alternativo |
| `CANCELLATION_CODE` | Cancellation Code | Razón de la cancelación. Solo tiene valor cuando `CANCELLED = 1`. | `A` = Aerolínea, `B` = Clima, `C` = NAS, `D` = Seguridad | POST-HOC |
| `DIVERTED` | Diverted Flight Indicator | Indicador binario de vuelo desviado a aeropuerto alternativo | `1` = Desviado, `0` = No desviado | OBJETIVO binario alternativo |

---

### 3.8 Variables de Resumen de Vuelo

| Campo | Nombre Completo | Descripción | Tipo de Dato | Rango | Disponibilidad |
|-------|-----------------|-------------|--------------|-------|----------------|
| `CRS_ELAPSED_TIME` | CRS Elapsed Time | Duración total programada del vuelo en minutos (puerta a puerta), calculada como `CRS_ARR_TIME – CRS_DEP_TIME`. | Float | >= 0 | PREDICTOR — captura holgura del itinerario |
| `ACTUAL_ELAPSED_TIME` | Actual Elapsed Time | Duración total real del vuelo en minutos. Nulo si cancelado. | Float | >= 0 | POST-HOC |
| `AIR_TIME` | Air Time | Tiempo en minutos que el avión pasó efectivamente en el aire (`WHEELS_ON – WHEELS_OFF`). No incluye rodaje. | Float | >= 0 | POST-HOC (media histórica usable como PREDICTOR) |
| `FLIGHTS` | Number of Flights | Número de vuelos representados en el registro. Normalmente `1`. En registros agregados puede ser mayor. | Entero | >= 1 | Metadata |
| `DISTANCE` | Distance | Distancia entre aeropuerto de origen y destino en millas. | Float | >= 0 | PREDICTOR — proxy de duración y tipo de ruta |

---

### 3.9 Variables de Causa de Retraso

> Disponibles desde **junio de 2003**. Solo tienen valor cuando el vuelo llegó con retraso (`ARR_DELAY > 0`). La suma de los cinco componentes puede no igualar el `ARR_DELAY` total por diferencias de cálculo en los procedimientos de reporte.

| Campo | Nombre Completo | Descripción | Tipo de Dato | Disponibilidad |
|-------|-----------------|-------------|--------------|----------------|
| `CARRIER_DELAY` | Carrier Delay | Retraso en minutos atribuido a la aerolínea (mantenimiento, limpieza, boarding tardío, problemas de tripulación). | Float | POST-HOC |
| `WEATHER_DELAY` | Weather Delay | Retraso en minutos atribuido a condiciones meteorológicas extremas que impiden la operación segura. | Float | POST-HOC |
| `NAS_DELAY` | NAS Delay | Retraso en minutos atribuido al Sistema Aéreo Nacional: control de tráfico, aeropuerto congestionado, operaciones de desvío no meteorológicas. | Float | POST-HOC |
| `SECURITY_DELAY` | Security Delay | Retraso en minutos por evacuaciones, re-examinación de pasajeros o esperas de autoridades de seguridad. | Float | POST-HOC |
| `LATE_AIRCRAFT_DELAY` | Late Aircraft Delay | Retraso en minutos por llegada tardía del avión en su vuelo anterior. Es la causa de retraso más frecuente en EE.UU. Señal directa del efecto cascada. | Float | POST-HOC (media histórica usable como PREDICTOR de riesgo de cascada) |

---

### 3.10 Variables de Tiempo en Puerta (Gate Return)

> Disponibles desde **octubre de 2008**. Aplican a vuelos que regresaron a la puerta después de alejarse de ella o que fueron cancelados tras el pushback.

| Campo | Nombre Completo | Descripción | Tipo de Dato | Disponibilidad |
|-------|-----------------|-------------|--------------|----------------|
| `FIRST_DEP_TIME` | First Gate Departure Time | Hora del primer intento de salida de puerta. Difiere de `DEP_TIME` cuando el vuelo regresó a la puerta tras el pushback. | Entero (hhmm) | POST-HOC |
| `TOTAL_ADD_GTIME` | Total Additional Ground Time | Tiempo total en minutos que el avión permaneció en tierra fuera de la puerta durante retenciones o cancelaciones post-pushback. | Float | POST-HOC |
| `LONGEST_ADD_GTIME` | Longest Additional Ground Time | Mayor tiempo individual fuera de la puerta durante el mismo evento de retención. Identifica incidentes severos de espera en pista. | Float | POST-HOC |

---

### 3.11 Variables de Vuelos Desviados

> Disponibles desde **octubre de 2008**. Solo tienen valor cuando `DIVERTED = 1`.

| Campo | Nombre Completo | Descripción | Tipo de Dato | Disponibilidad |
|-------|-----------------|-------------|--------------|----------------|
| `DIV_AIRPORT_LANDINGS` | Diverted Airport Landings | Número de aterrizajes en aeropuertos alternativos durante el desvío (máximo 5 en el dataset). | Entero | POST-HOC |
| `DIV_REACHED_DEST` | Diverted Flight Reaching Destination | Indica si el vuelo desviado eventualmente llegó a su destino programado original. | `1` = Sí llegó, `0` = No llegó | POST-HOC |
| `DIV_ACTUAL_ELAPSED_TIME` | Diverted Flight Elapsed Time | Tiempo real en minutos del vuelo desviado que alcanzó su destino programado. El campo `ACTUAL_ELAPSED_TIME` queda nulo para vuelos desviados. | Float | POST-HOC |
| `DIV_ARR_DELAY` | Diverted Flight Arrival Delay | Diferencia en minutos entre la hora programada y real de llegada para vuelos desviados que eventualmente alcanzaron su destino. El campo `ARR_DELAY` queda nulo para todos los desviados. | Float | OBJETIVO alternativo para vuelos desviados |
| `DIV_DISTANCE` | Diverted Distance | Distancia en millas entre el destino programado y el aeropuerto de desvío final. Valor `0` si el vuelo eventualmente llegó a su destino. | Float | POST-HOC |
| `DIV1_AIRPORT` | Diverted Airport Code 1 | Código IATA del primer aeropuerto alternativo al que aterrizó el vuelo desviado. | Texto (3 chars) | POST-HOC |
| `DIV1_AIRPORT_ID` | Diverted Airport ID 1 | Identificador único DOT del primer aeropuerto de desvío. Clave estable para joins con tablas de aeropuertos. | Entero | POST-HOC |

---

## 4. Variables Predictoras Propuestas

### 4.1 Variables Disponibles en el Dataset (Sin Ingeniería Adicional)

| Variable | Justificación para el modelo |
|----------|------------------------------|
| `MKT_UNIQUE_CARRIER` | Las aerolíneas tienen perfiles de puntualidad significativamente distintos |
| `ORIGIN` / `DEST` | Los aeropuertos tienen niveles de congestión y patrones climáticos característicos |
| `ORIGIN_CITY_MARKET_ID` / `DEST_CITY_MARKET_ID` | Captura mercados servidos por múltiples aeropuertos |
| `CRS_DEP_TIME` | La hora del día determina el nivel de congestión acumulada en el sistema |
| `CRS_ARR_TIME` | Captura si el vuelo llega en hora pico o en horario nocturno de menor tráfico |
| `CRS_ELAPSED_TIME` | La holgura del itinerario respecto al histórico de la ruta es señal de riesgo |
| `DISTANCE` | Proxy de duración y tipo de ruta: rutas cortas son más vulnerables a retrasos relativos |
| `TAIL_NUM` | Permite cruzar con el vuelo inbound del mismo avión para detectar efecto cascada |

### 4.2 Features Derivadas de Ingeniería de Características

#### Temporales

| Feature | Cálculo | Señal esperada |
|---------|---------|----------------|
| `DEP_HOUR` | `floor(CRS_DEP_TIME / 100)` | Vuelos de madrugada tienen menos retrasos; los de tarde-noche acumulan mayor congestión |
| `DEP_PERIOD` | Categórica: Madrugada / Mañana / Tarde / Tarde-noche / Noche | Simplifica `DEP_HOUR` para modelos de árbol |
| `DOW` | Día de la semana numérico (0 = Lunes) | Viernes y domingos concentran mayor tráfico y retrasos |
| `IS_WEEKEND` | `1` si DOW >= 5 | Booleano de bajo costo con señal operacional alta |
| `MONTH` | Mes del año del vuelo | Captura estacionalidad anual completa |
| `SEASON` | Invierno / Primavera / Verano / Otoño | Agrupa meses con patrones climáticos similares |
| `IS_PEAK_TRAVEL` | `1` si mes pertenece a {6, 7, 8, 11, 12} | Temporada alta en EE.UU.: verano y navidad |
| `IS_HOLIDAY_WEEK` | `1` si la semana contiene un día festivo federal | Semanas festivas concentran los retrasos más extremos del año |
| `BLOCK_PADDING_PCT` | `(CRS_ELAPSED_TIME – min_hist_ruta) / min_hist_ruta × 100` | Rutas con más padding llegan a tiempo más frecuentemente |

#### Históricas de Aerolínea (calcular con datos de M-12 a M-2)

| Feature | Definición | Ventana recomendada |
|---------|------------|---------------------|
| `CARRIER_AVG_DELAY_HIST` | Retraso promedio de llegada de la aerolínea en el mismo mes del año anterior | 12 meses anteriores, mismo período |
| `CARRIER_PCT_ONTIME_HIST` | Porcentaje de vuelos con `ARR_DELAY` <= 15 min de la aerolínea | 6–12 meses anteriores |
| `CARRIER_CANCEL_RATE_HIST` | Tasa de cancelación histórica de la aerolínea | 12 meses anteriores |
| `CARRIER_LATE_AIRCRAFT_RATE_HIST` | Porcentaje de retrasos tipo LATE_AIRCRAFT de la aerolínea | 6 meses anteriores |

#### Históricas de Ruta (calcular con datos de M-12 a M-2)

| Feature | Definición | Ventana recomendada |
|---------|------------|---------------------|
| `ROUTE_AVG_DELAY_HIST` | Retraso promedio de llegada en la ruta ORIGIN–DEST | 12 meses anteriores |
| `ROUTE_PCT_DELAYED_HIST` | Porcentaje de vuelos retrasados >15 min en la ruta | 12 meses anteriores |
| `ROUTE_STD_DELAY_HIST` | Desviación estándar del retraso en la ruta (variabilidad = menor predictibilidad) | 12 meses anteriores |
| `ROUTE_AVG_TAXI_OUT_HIST` | Tiempo promedio de TAXI_OUT en la ruta para la misma franja horaria | 6 meses anteriores |
| `ROUTE_AVG_TAXI_IN_HIST` | Tiempo promedio de TAXI_IN en la ruta | 6 meses anteriores |
| `ROUTE_AVG_AIR_TIME_HIST` | Tiempo promedio de AIR_TIME en la ruta (comparar con `CRS_ELAPSED_TIME`) | 6 meses anteriores |

#### De Tráfico de Pista (Congestión Aeroportuaria)

| Feature | Definición | Señal esperada |
|---------|------------|----------------|
| `ORIGIN_AVG_TAXI_OUT_BY_HOUR_HIST` | TAXI_OUT promedio histórico en ORIGIN para la misma franja horaria | Mayor taxi-out histórico = mayor congestión esperada |
| `ORIGIN_DEP_VOLUME_BY_HOUR_HIST` | Número promedio de vuelos que salen de ORIGIN en la misma hora del día | Proxy de saturación de pistas en ese horario |
| `ORIGIN_PCT_DELAYED_BY_HOUR_HIST` | Porcentaje de vuelos retrasados en ORIGIN para la misma hora y mes | Señal directa de riesgo por combinación hora–aeropuerto |
| `DEST_ARR_VOLUME_BY_HOUR_HIST` | Número promedio de llegadas a DEST en la franja de arribo esperado | Congestión en destino puede generar esperas de holding |
| `ORIGIN_SEASONAL_DELAY_INDEX` | Ratio del retraso promedio del aeropuerto en el mismo mes vs. su promedio anual | Captura si el aeropuerto es especialmente propenso a retrasos en cierta temporada |

#### De Vuelos Conectados (Efecto Cascada)

| Feature | Definición | Disponibilidad |
|---------|------------|----------------|
| `INBOUND_ROUTE_AVG_DELAY_HIST` | Retraso promedio histórico de la ruta que alimenta el vuelo (trae el avión a ORIGIN) | Requiere `TAIL_NUM` + ruteo previo |
| `TAIL_INBOUND_LATE_RATE_HIST` | Porcentaje de veces que el avión llegó tarde en su vuelo inbound históricamente | Requiere `TAIL_NUM` |
| `TURNAROUND_TIME_SCHEDULED` | `CRS_DEP_TIME – inbound_CRS_ARR_TIME` en minutos | Requiere ruteo de `TAIL_NUM` |
| `FLAG_TIGHT_TURNAROUND` | `1` si `TURNAROUND_TIME_SCHEDULED` < 45 minutos | Requiere ruteo de `TAIL_NUM` |
| `ORIGIN_INBOUND_AVG_DELAY_HIST` | Retraso promedio histórico de vuelos que LLEGAN a ORIGIN en la hora previa al vuelo analizado | Proxy de congestión inbound en el aeropuerto de origen |
| `LATE_AIRCRAFT_EXPOSURE_HIST` | Porcentaje histórico de vuelos de la aerolínea afectados por retrasos tipo LATE_AIRCRAFT en ORIGIN | Captura vulnerabilidad estructural de la aerolínea en ese aeropuerto |

---

## 5. Variable Objetivo

### 5.1 Definición Recomendada

```
TARGET_DELAYED_15 = 1   si   ARR_DELAY > 15 minutos
TARGET_DELAYED_15 = 0   si   ARR_DELAY <= 15 minutos o vuelo adelantado
```

Los vuelos cancelados pueden incluirse como clase positiva (`TARGET = 1`) dado que representan la peor experiencia para el pasajero, o tratarse por separado en un modelo dedicado de cancelaciones.

### 5.2 Categorías de Riesgo para Uso Operacional

El score continuo del modelo (probabilidad 0–1) se traduce en categorías accionables:

| Score | Categoría | Acción Sugerida |
|-------|-----------|-----------------|
| 0.00 – 0.20 | BAJO | Sin acción especial, operación estándar |
| 0.20 – 0.45 | MEDIO | Monitoreo preventivo, mensaje informativo en app |
| 0.45 – 0.70 | ALTO | Alerta proactiva al pasajero, preparar agentes de atención |
| 0.70 – 1.00 | CRÍTICO | Comunicación inmediata, plan de contingencia activado |

### 5.3 Variables Objetivo Alternativas

| Variable | Tipo de Modelo | Caso de Uso |
|----------|----------------|-------------|
| `CANCELLED` | Clasificación binaria | Modelo dedicado de predicción de cancelaciones |
| `ARR_DELAY` (valor continuo) | Regresión | Estimar minutos exactos de retraso esperado |
| `ARR_DELAY > 60` | Clasificación binaria | Retrasos severos sujetos a compensación |
| `DIV_ARR_DELAY` | Regresión | Análisis dedicado de vuelos desviados |

---

## 6. Arquitectura del Modelo

### 6.1 Pipeline Propuesto

```
Datos Históricos (hasta mes M-1)
        |
        v
[1. EXTRACCIÓN Y LIMPIEZA]
   - Filtrar vuelos cancelados según objetivo
   - Imputar nulos operacionales
   - Validar integridad temporal
        |
        v
[2. FEATURE ENGINEERING]
   - Variables temporales
   - Agregados históricos por aerolínea, ruta, aeropuerto
   - Flags de temporada alta y días festivos
   - Features de congestión de pista (rolling por hora)
   - Features de cascada (TAIL_NUM + inbound routing)
        |
        v
[3. ENTRENAMIENTO]
   - Datos: M-12 a M-2
   - Walk-forward cross-validation
   - Baseline: Regresión Logística
   - Modelo: Random Forest / Gradient Boosting (XGBoost/LightGBM)
        |
        v
[4. VALIDACIÓN]
   - Período de validación: M-1
   - Métricas: AUC-ROC, Precision, Recall, Brier Score
   - Evaluación por aerolínea y aeropuerto
   - Análisis de calibración de probabilidad
        |
        v
[5. PREDICCIÓN]
   - Input: horario programado de mes M+1
   - Output: score de probabilidad + categoría por vuelo
   - Exportar a dashboard operacional
```

### 6.2 Métricas de Evaluación

| Métrica | Justificación |
|---------|---------------|
| AUC-ROC | Mide discriminación global sin depender del umbral de decisión |
| Precision @ umbral ALTO (0.45) | Evitar falsas alarmas que generen fatiga operacional |
| Recall @ umbral MEDIO (0.20) | Capturar la mayor proporción posible de retrasos reales |
| Brier Score | Evalúa la calibración de la probabilidad (confianza del modelo) |
| F1 por aerolínea y aeropuerto | Detectar segmentos donde el modelo tiene bajo desempeño |

---

## 7. Estrategia de Validación

### 7.1 Walk-Forward Temporal

Para evitar data leakage, se utiliza validación temporal en ventana rodante:

```
Fold 1:  TRAIN [Ene – Jun]  -->  TEST [Jul]
Fold 2:  TRAIN [Ene – Jul]  -->  TEST [Ago]
Fold 3:  TRAIN [Ene – Ago]  -->  TEST [Sep]
Fold 4:  TRAIN [Ene – Sep]  -->  TEST [Oct]
...
Fold N:  TRAIN [Ene – M-2]  -->  TEST [M-1]
```

### 7.2 Protocolo Anti-Data Leakage

| Variable | Riesgo de Leakage | Protocolo |
|----------|-------------------|-----------|
| `CARRIER_AVG_DELAY_HIST` | Alto | Calcular solo con datos del fold TRAIN correspondiente |
| `ROUTE_AVG_DELAY_HIST` | Alto | Calcular solo con datos del fold TRAIN correspondiente |
| `ORIGIN_AVG_TAXI_OUT_BY_HOUR_HIST` | Alto | Calcular solo con datos del fold TRAIN correspondiente |
| `BLOCK_PADDING_PCT` | Ninguno | El mínimo histórico se calcula con TRAIN; el valor a predecir es el programado |
| `INBOUND_ROUTE_AVG_DELAY_HIST` | Medio | Verificar que el inbound sea temporalmente anterior al vuelo analizado |
| `DEP_DELAY` como predictor directo | Crítico — NO USAR | No está disponible antes del vuelo en producción |
| `TAXI_OUT` como valor real | Crítico — NO USAR | Solo usar la media histórica, nunca el valor real del vuelo a predecir |
| `ARR_DELAY` como predictor | Crítico — NO USAR | Es la variable objetivo; usarla como predictor colapsa el modelo |

---

## 8. Uso Operacional

### 8.1 Flujo de Uso Mensual

```
Inicio del mes M:
  1. Cargar horario completo de vuelos programados para mes M+1
  2. Enriquecer cada vuelo con features históricas calculadas hasta M-1
  3. Aplicar modelo entrenado -> obtener score y categoría por vuelo
  4. Exportar resultados a dashboard operacional

Durante el mes M:
  5. Equipo de operaciones revisa vuelos ALTO y CRÍTICO
  6. Configurar mensajes preventivos en app para vuelos con score > 0.45
  7. Asignar agentes adicionales en aeropuertos con alta concentración
     de vuelos CRÍTICOS en un mismo día y franja horaria
  8. Emitir alertas anticipadas a pasajeros de vuelos CRÍTICOS
  9. Preparar opciones de rebooking proactivo para vuelos CRÍTICOS
```

### 8.2 Acciones por Nivel de Riesgo

| Categoría | Canal de Comunicación | Mensaje Tipo | Acción Operacional |
|-----------|----------------------|--------------|-------------------|
| BAJO | Ninguno | — | Operación estándar sin acción adicional |
| MEDIO | App (push suave) | "Tu vuelo podría experimentar ligeros cambios. Te mantendremos informado." | Monitoreo estándar, sin asignación adicional |
| ALTO | App (push + email) | "Tu vuelo tiene una probabilidad elevada de retraso. Recomendamos llegar con tiempo extra." | Agente asignado, plan de rebooking preparado |
| CRÍTICO | App + SMS + llamada proactiva | "Hemos detectado alto riesgo de retraso en tu vuelo. Un agente se pondrá en contacto contigo." | Protocolo de contingencia activado, rebooking proactivo disponible |

### 8.3 Beneficios Esperados del Sistema

| Stakeholder | Beneficio |
|-------------|-----------|
| Pasajero | Anticipación y control. Menor sorpresa y frustración ante retrasos. |
| Operaciones | Asignación eficiente de personal en ventanas de alta demanda conocidas con antelación. |
| Aplicación | Mensajes relevantes y oportunos que aumentan la confianza y retención del usuario. |
| Negocio | Reducción de costos de compensación por retrasos mal gestionados y mejora del NPS. |

---

## 9. Glosario Operacional

| Término | Definición |
|---------|------------|
| **ARR_DELAY** | Minutos de diferencia entre la hora real y programada de llegada a la puerta. Negativo = vuelo adelantado. |
| **Cascada (Late Aircraft)** | Retraso propagado: el avión llega tarde de su vuelo anterior y no puede salir a tiempo en el siguiente. Es la causa de retraso más frecuente en EE.UU. |
| **CRS (Computer Reservation System)** | Sistema de reservas computarizado. Las variables con prefijo CRS son los valores del itinerario programado. |
| **Data Leakage** | Uso de información del futuro para entrenar el modelo. Genera resultados artificialmente buenos que no se sostienen en producción. |
| **DOT** | Department of Transportation de EE.UU. Organismo regulador que define el umbral oficial de 15 minutos para clasificar un vuelo como retrasado. |
| **IATA** | International Air Transport Association. Asigna códigos de 2 letras para aerolíneas y de 3 letras para aeropuertos. |
| **NAS Delay** | Retraso atribuido al Sistema Aéreo Nacional: control de tráfico aéreo, congestión de aeropuerto, operaciones meteorológicas no extremas. |
| **Tail Number** | Matrícula física del avión. Permite rastrear el historial del avión y su vuelo inbound anterior para modelar el efecto cascada. |
| **TAXI_OUT** | Tiempo desde que el avión se aleja de la puerta hasta el despegue. Alta variabilidad en aeropuertos congestionados. |
| **TAXI_IN** | Tiempo desde el aterrizaje hasta llegar a la puerta en destino. Generalmente más corto y estable que TAXI_OUT. |
| **Turnaround Time** | Tiempo en tierra del avión entre su vuelo inbound y el vuelo a predecir. Un turnaround corto (< 45 min) aumenta el riesgo de retraso cascada. |
| **Walk-Forward Validation** | Esquema de validación temporal donde se entrena con el pasado y se evalúa con el futuro inmediato, respetando estrictamente el orden cronológico. |
| **WHEELS_OFF / WHEELS_ON** | Momentos exactos de despegue y aterrizaje. La diferencia entre ambos es el AIR_TIME real. |

---

*Documento generado con base en el diccionario oficial de la BTS — Bureau of Transportation Statistics, U.S. Department of Transportation.*  
*Referencia: https://www.transtats.bts.gov*