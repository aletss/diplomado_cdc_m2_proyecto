# Métricas de Aviación - Guía Sencilla

## Conceptos Básicos

Las métricas de aviación documentan cada fase del viaje de un vuelo, desde que el avión se prepara para salir hasta que llega a su destino final. Estas métricas se dividen en tiempos **programados** (CRS = Computer Reservation System) y tiempos **reales**.

---

## Fases de un Vuelo

### 1. **En la Puerta de Salida (Origin Gate)**

**CRS_DEP_TIME** (Hora Programada de Salida)
- La hora a la que el vuelo *debería* salir de la puerta según el itinerario
- Ejemplo: 14:30 (2:30 PM)

**DEP_TIME** (Hora Real de Salida)
- La hora a la que el avión realmente se aleja de la puerta
- Ejemplo: 14:45 (2:45 PM) → hay un retraso de 15 minutos

**DEP_DELAY** (Retraso de Salida)
- Diferencia entre la hora programada y real
- Ejemplo: 15 minutos (valores negativos = salida temprana)

---

### 2. **Rodaje de Salida (Taxi Out)**

**TAXI_OUT** (Tiempo de Rodaje de Salida)
- Tiempo que el avión tarda rodando desde la puerta hasta la pista de despegue
- Incluye: esperas en cola, maniobras en el aeropuerto
- Ejemplo: 12 minutos

**WHEELS_OFF** (Hora de Despegue)
- Momento exacto en que el avión despega y las ruedas dejan el suelo
- Ejemplo: 14:57

---

### 3. **En el Aire (Air Time)**

**AIR_TIME** (Tiempo de Vuelo)
- Tiempo que el avión pasa en el aire, desde despegue hasta aterrizaje
- No incluye rodaje
- Ejemplo: 120 minutos (2 horas)

---

### 4. **Aterrizaje y Rodaje de Llegada**

**WHEELS_ON** (Hora de Aterrizaje)
- Momento exacto en que las ruedas del avión tocan la pista
- Ejemplo: 16:57

**TAXI_IN** (Tiempo de Rodaje de Llegada)
- Tiempo que el avión tarda rodando desde la pista hasta la puerta
- Ejemplo: 8 minutos

---

### 5. **En la Puerta de Llegada (Destination Gate)**

**CRS_ARR_TIME** (Hora Programada de Llegada)
- La hora a la que el vuelo *debería* llegar a la puerta según el itinerario
- Ejemplo: 17:00 (5:00 PM)

**ARR_TIME** (Hora Real de Llegada)
- La hora a la que el avión realmente llega a la puerta
- Ejemplo: 17:05 (5:05 PM)

**ARR_DELAY** (Retraso de Llegada)
- Diferencia entre la hora programada y real de llegada
- Ejemplo: 5 minutos

---

## Tiempos Totales

**CRS_ELAPSED_TIME** (Tiempo Total Programado)
- Tiempo total programado desde salida de la puerta origen hasta llegada a la puerta destino
- CRS_ARR_TIME - CRS_DEP_TIME
- Ejemplo: 150 minutos (2.5 horas)

**ACTUAL_ELAPSED_TIME** (Tiempo Total Real)
- Tiempo total real desde salida de la puerta origen hasta llegada a la puerta destino
- ARR_TIME - DEP_TIME
- Ejemplo: 140 minutos

---

## Fórmulas Importantes

```
ACTUAL_ELAPSED_TIME = TAXI_OUT + AIR_TIME + TAXI_IN

DEP_DELAY = DEP_TIME - CRS_DEP_TIME

ARR_DELAY = ARR_TIME - CRS_ARR_TIME
```

---

## Ejemplo Práctico

**Vuelo: AA 1234 de Dallas (DFW) a Nueva York (JFK)**

| Métrica | Valor | Nota |
|---------|-------|------|
| CRS_DEP_TIME | 14:30 | Salida programada |
| DEP_TIME | 14:45 | Salida real (15 min tarde) |
| TAXI_OUT | 12 min | Rodaje a pista |
| WHEELS_OFF | 14:57 | Despegue |
| AIR_TIME | 120 min | Tiempo volando |
| WHEELS_ON | 16:57 | Aterrizaje |
| TAXI_IN | 8 min | Rodaje a puerta |
| ARR_TIME | 17:05 | Llegada real a puerta |
| CRS_ARR_TIME | 17:00 | Llegada programada |
| ARR_DELAY | 5 min | Llegada 5 min tarde |

**Nota**: Aunque salió 15 minutos tarde, solo llegó 5 minutos tarde porque el vuelo fue más rápido de lo programado.

---

## Conceptos Adicionales

### Bloques de Tiempo (Time Blocks)

**DEP_TIME_BLK** y **ARR_TIME_BLK**
- Agrupan vuelos en intervalos horarios (ej: 14:00-14:59)
- Útil para análisis de patrones de tráfico

### Vuelos Especiales

**CANCELLED** = 1
- El vuelo fue cancelado
- Los tiempos de vuelo serán NULL

**DIVERTED** = 1
- El vuelo fue desviado a otro aeropuerto
- Tendrá información adicional de aeropuertos de desvío

---

## Tips para Analizar Datos

1. **Retrasos**: Un número negativo en DEP_DELAY o ARR_DELAY significa que el vuelo salió/llegó temprano

2. **Vuelos Cancelados**: Si CANCELLED = 1, muchos campos estarán vacíos (NULL)

3. **Tiempo en Tierra**: TAXI_OUT + TAXI_IN = tiempo total rodando

4. **Eficiencia**: Comparar ACTUAL_ELAPSED_TIME vs CRS_ELAPSED_TIME muestra si los vuelos son más rápidos o lentos que lo programado

5. **Causas de Retraso**: Solo disponibles cuando ARR_DELAY > 0 y el vuelo llegó tarde