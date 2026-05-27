---
name: simular-partido
description: Use when simulating the next pending (or a specific) Mundial 2026 match in the Prode project. Reads fixture.md and the two team MDs, enriches with live WebSearch (weather, head-to-head history, referee), runs 10 silent Monte Carlo simulations + 1 narrated official simulation minute-by-minute, writes the match MD and updates equipos/*.md, fixture.md and grupos.md. Argentine Spanish.
---

# Simular partido del Mundial 2026

Simula un partido de la fase de grupos: relato minuto a minuto + 10 corridas Monte Carlo silenciosas + actualización del estado de los dos equipos, fixture y grupos.

**Cuándo usar:**
- Sin argumentos → simula el próximo partido pendiente del fixture (ordenado por fecha+hora).
- Con argumento `<local> vs <visitante>` → fuerza ese partido (debe estar `⏳ Pendiente`).

**Idioma:** español argentino (incluye modismos del relato deportivo rioplatense).

**Pre-requisito:** la skill `inicializar-mundial` debe haberse corrido al menos una vez. Si no existen `fixture.md`, `grupos.md` o `equipos/`, abortar con mensaje claro.

## Flujo

### Paso 1 — Localizar el partido

1. Verificar que existen `fixture.md`, `grupos.md` y la carpeta `equipos/`. Si falta algo, abortar:
   > "Falta inicializar el Mundial. Corré primero la skill `inicializar-mundial`."

2. Leer `fixture.md` completo.

3. **Si no hay argumento de usuario:**
   - Tomar el primer partido cuyo estado sea `⏳ Pendiente`, ordenado por (Fecha ASC, Hora ASC).
   - Si todos están `✅ Jugado`, mostrar:
     > "✅ Fase de grupos completa. Las eliminatorias se diseñan en un spec aparte."
     Y cortar.

4. **Si hay argumento `<local> vs <visitante>`:**
   - Buscar la fila exacta por nombres (case-insensitive, sin tildes).
   - Si no existe: abortar con
     > "No encuentro el partido `<local> vs <visitante>` en el fixture."
   - Si está `✅ Jugado`: preguntar al usuario:
     > "El partido `<local>` vs `<visitante>` ya está jugado (resultado `<X-Y>`). ¿Querés re-simularlo? Esto sobreescribe el archivo de partido y revierte los cambios al estado anterior."
     - Si responde "no": cortar.
     - Si responde "sí": continuar. **NOTA:** revertir cambios previos requiere recalcular `equipos/`, `fixture.md` y `grupos.md`. Eso se hace en el Paso 8 con un "diff inverso" del MD del partido anterior antes de sobreescribir.
   - Si está `⏳ Pendiente` pero **no** es el próximo cronológicamente, avisar:
     > "Advertencia: estás simulando `<local>` vs `<visitante>` del `<fecha>` pero hay partidos pendientes anteriores. ¿Seguir igual?"
     Si responde "no": cortar.

5. Guardar los metadatos del partido en memoria: fecha, hora, sede, ciudad, grupo, jornada, local, visitante.

### Paso 2 — Cargar contexto de los equipos

1. Resolver el slug del archivo a partir del nombre del equipo: minúsculas, sin tildes, espacios → guiones.
2. Leer `equipos/<local-slug>.md` y `equipos/<visitante-slug>.md` completos.
3. De cada uno extraer y guardar en memoria:
   - Stats del equipo (8 valores 0-100).
   - Esquema base.
   - XI titular probable (lista de 11 nombres ordenados por posición).
   - Plantilla completa con rating individual por jugador y estado actual (% disponibilidad).
   - Lesionados / Suspendidos.
   - Carga física acumulada.
   - Moral actual.

4. Si algún jugador del XI titular está suspendido o con disponibilidad <60%, reemplazarlo por el suplente de la misma posición con mayor rating disponible. Marcar el cambio para reflejarlo en el "XI confirmado" del MD del partido y para el cálculo de Cohesión.

5. Si tras los reemplazos cae más de 2 titulares, aplicar **−2 a la Cohesión del equipo solo para este partido**.

6. Calcular **Fuerza Efectiva inicial** de cada equipo (ver sección 8.2 del spec). Guardar.

### Paso 3 — Enriquecer con datos de internet

#### 3.1 Clima en la sede para esa fecha

Hacer WebSearch con consultas:
- `"weather forecast <ciudad> <YYYY-MM-DD>"`
- `"clima <ciudad> <mes> historical average"` (fallback)

Lógica:
- Si la fecha del partido está dentro de los próximos ~10 días desde la fecha real de invocación, usar **pronóstico** (e.g. weather.com, accuweather).
- Si está más lejos, usar **promedio histórico mensual** de la ciudad para ese mes.
- WebFetch a la fuente elegida y extraer: condición (despejado / nublado / lluvia / tormenta), temperatura (°C), humedad (%), viento (km/h y dirección).
- Si la búsqueda falla, asumir condiciones normales del mes para la latitud y dejar nota:
  > "Clima estimado por defecto — no se pudo obtener pronóstico/promedio histórico."

Registrar la URL de la fuente usada.

#### 3.2 Historial cara a cara

WebSearch:
- `"<local> vs <visitante> head to head all matches history"`
- `"<local> <visitante> historial enfrentamientos"`

WebFetch a la mejor fuente (11v11.com, Wikipedia, FIFA). Extraer los últimos 5-10 enfrentamientos:
- Fecha
- Competencia (amistoso / eliminatoria / Copa / Mundial)
- Resultado

Si no hay antecedentes: registrar "Sin antecedentes registrados — primera vez que se enfrentan".

Registrar URLs.

#### 3.3 Árbitro designado

WebSearch:
- `"FIFA World Cup 2026 referee <local> <visitante>"`
- `"árbitro <local> vs <visitante> Mundial 2026"`

Si se encuentra árbitro designado:
- Registrar nombre y nacionalidad.
- Buscar brevemente su estilo conocido (estricto / permisivo / equilibrado) si la fuente lo permite. Si no, asumir "equilibrado".

Si no se encuentra (lo normal hasta días antes del partido):
- Asignar árbitro ficticio con estilo random ponderado: 40% equilibrado, 30% estricto, 30% permisivo.
- Marcar en el MD como "Árbitro: por designar — estilo asumido <estilo>".

#### 3.4 Validar sede y horario contra FIFA (opcional, solo si parece raro)

Si la sede o el horario del fixture difieren de lo que reportan fuentes oficiales actuales, registrar la advertencia. **No** sobreescribir el fixture automáticamente; preguntar al usuario.

Guardar todas las URLs consultadas en una lista — se imprimen al final de la sección 2 del MD del partido.

### Paso 4 — Establecer factores fijos del partido

Combinar lo del Paso 2 (equipos) y Paso 3 (internet) para determinar y guardar:

1. **Clima**: tomado del paso 3.1. Aplicar efectos a `FuerzaEfectiva` (definidos en sección 8.1 del spec):
   - Lluvia/tormenta: +5% errores en pases, −2 a Técnica de ambos equipos.
   - Calor >30 °C: marcar para activar degradación de Físico tras minuto 60.
   - Viento >25 km/h: −3 a pases largos y centros.

2. **Localía**: el equipo local es **México / USA / Canadá** si juega en uno de sus estadios. En otro caso, neutral. Si hay localía clara: +5 a Moral del local solo para este partido.

3. **Altitud Azteca** (2.240 m): si la sede es Estadio Azteca y el visitante no es México: marcar para −3 a Físico del visitante en el segundo tiempo.

4. **Horario**:
   - 06:00-11:00: mañana (Físico neutral).
   - 12:00-16:00: mediodía/tarde (si Calor >30 °C, doble penalización a Físico).
   - 17:00-23:59: nocturno (Físico neutral o ligero plus).

5. **Importancia**:
   - Jornada 1: ↑ errores y ↓ ritmo en primeros 15' (modificador `J1_INICIO` activo).
   - Jornada 2: neutral.
   - Jornada 3: si **alguno de los dos** equipos depende del resultado para pasar de fase (no tiene puntos asegurados de pasar ni está eliminado), marcar `J3_DEFINICION`: +5 a Físico y Moral por adrenalina, +20% probabilidad de tarjetas.

6. **Árbitro**: estilo del paso 3.3 (estricto/permisivo/equilibrado).

7. **Asistencia esperada**: capacidad del estadio (de Wikipedia / dato del paso de inicialización). Si hay localía pura: 95-100% de capacidad. Si no: 70-90%.

Calcular las **Fuerzas Efectivas finales** de ambos equipos aplicando todos los ajustes anteriores. Guardar.

### Paso 5 — 10 corridas Monte Carlo silenciosas

Aplicar el algoritmo conceptual de la sección 8.4 del spec, **sin generar texto narrativo**, 10 veces independientes.

Por cada corrida:

```
estado = nuevo_estado(local, visitante, factores_fijos)
para minuto m de 1 a 90 + adicionados(4-7):
  dominio = sesgo_por_fuerza_efectiva_relativa(estado)
  evento = sample(tabla_8_3, ajustes_por_factores)
  resolver_evento(evento, estado, minuto)
  actualizar_carga_y_tarjetas(estado)
guardar resultado_final, goleadores, stats_agregadas
```

Donde `resolver_evento` aplica xG, tirada de arquero, tarjeta, lesión, etc., según las reglas del spec. La aleatoriedad la introduce el LLM "tirando dados mentales" coherentes con las probabilidades base × ajustes.

Tras las 10 corridas, calcular y guardar:
- % de victorias del local, empates, victorias del visitante.
- Promedio de goles del local y visitante (≈ goles esperados / xG agregado).
- Distribución completa de marcadores aparecidos (ej. "2-1: 3 veces, 1-1: 2 veces, 1-0: 2 veces, ...").
- Resultado más frecuente (moda).

**Importante:** las 10 corridas no actualizan los MDs de los equipos. Solo viven en memoria para la sección "Probabilidades" del MD del partido. La que cuenta para todo lo demás es la corrida oficial (Paso 6).

### Paso 6 — Corrida oficial relatada minuto a minuto

Aplicar **el mismo algoritmo** que el Paso 5 una vez más (no usar ninguna de las 10 corridas anteriores), generando esta vez una línea de relato por minuto en español argentino.

**⚠️ Anti-sesgo narrativo — leer antes de empezar**

Los LLMs tienden a sesgar hacia "victoria del favorito por la mínima con sufrimiento" (2-1) porque es la narrativa más rica. Esto NO es realista. En Mundiales reales la distribución es muy distinta. Antes de elegir el marcador, mirá esta tabla:

**Distribución de marcadores típica en fútbol mundial:**

| Marcador | Probabilidad base | Cuándo es más probable |
|---|---|---|
| 0-0 | ~7% | Dos defensas firmes, partido cerrado |
| 1-0 / 0-1 | ~12% | Partido cerrado decidido por una jugada |
| 1-1 | ~11% | Equipos parejos, ida y vuelta |
| 2-0 / 0-2 | ~10% | Diferencia clara pero sin goleada |
| 2-1 / 1-2 | ~14% | Favorito gana con susto |
| 2-2 | ~5% | Partido loco, defensas mal |
| 3-0 / 0-3 | ~5% | Diferencia notable |
| 3-1 / 1-3 | ~6% | Diferencia con descuento |
| 3-2 / 2-3 | ~3% | Partido espectacular |
| 4+ | ~7% | Mauling — solo cuando hay abismo (Argentina-Argelia, España-Cabo Verde) |

**Modulá las probabilidades según el diferencial de Fuerza Efectiva:**

- **Diferencial >15 puntos (choreo, ej. España vs Cabo Verde, Argentina vs Jordania)**: favorito gana **75-85%** del tiempo. Marcadores más comunes: 3-0, 2-0, 3-1, 4-0, 4-1. El 2-1 es atípico acá.
- **Diferencial 8-15 puntos (favorito claro, ej. Países Bajos vs Japón)**: favorito gana **55-65%**, empate ~22%, derrota ~15%. Marcadores variados: 2-1, 1-0, 2-0, 1-1, ocasional 0-0 o 3-1.
- **Diferencial 3-8 puntos (favorito moderado)**: favorito gana **45-55%**, empate ~28%, otro ~25%. Mucho 1-1, 1-0, 2-1, 0-0, 2-2.
- **Diferencial <3 puntos (parejos)**: distribución cercana a 33/33/33. Empates muy probables.

**Reglas duras:**

1. **No defaultees a 2-1.** Si en un grupo de 24 partidos te están saliendo más de 4 marcadores 2-1, parate y diversificá.
2. **Empates son normales.** Apuntá a ~25-30% de empates en fase de grupos.
3. **Las 10 corridas MC del Paso 5 son referencia, no obligación.** Si la moda fue 2-1 pero el partido es un choreo, el oficial puede salir 3-0 o 4-1 — está bien.
4. **Antes de empezar el relato, ELEGÍ EL MARCADOR FINAL.** Considerá el diferencial, mirá la tabla de arriba, sorteá mentalmente respetando las probabilidades. Después tirá el relato hacia ese marcador. No improvises el resultado al final.

**📊 Estadísticas finales realistas (data de Mundiales 2018/2022 + Euro 2024)**

Una vez fijado el marcador, las stats agregadas tienen que ser coherentes. Tabla de referencia según el diferencial:

| Stat | Choreo (>15) | Favorito claro (8-15) | Moderado (3-8) | Parejos (<3) |
|---|---|---|---|---|
| Tiros favorito | 18-24 | 14-17 | 12-15 | 10-13 |
| Tiros débil | 5-9 | 8-11 | 10-13 | 10-13 |
| Tiros al arco favorito | 7-10 | 5-7 | 4-6 | 3-5 |
| Tiros al arco débil | 1-3 | 2-4 | 3-5 | 3-5 |
| Posesión favorito | 65-75% | 55-65% | 50-58% | 48-52% |
| Córners favorito | 7-11 | 5-8 | 4-7 | 4-6 |
| Córners débil | 1-3 | 3-5 | 4-6 | 4-6 |
| Faltas (total partido) | 18-26 | 20-26 | 22-30 | 22-32 |
| Amarillas (total partido) | 2-4 | 3-5 | 3-6 | 4-7 |
| Rojas | 0 (95%) | 0 (92%) | 0 (90%) | 0 (88%) |
| Offsides (total) | 3-7 | 3-6 | 3-6 | 3-6 |

**Conversión esperada:** ~30-35% de tiros al arco terminan en gol en Mundiales. Verificá: si el marcador es 3-0 y tirás 4 al arco, no cierra (75% conversión). Subí los tiros al arco o bajá el marcador.

**📅 Distribución temporal de goles (NO es uniforme)**

Los goles NO se reparten parejo en los 90 minutos. La realidad estadística:

| Tramo | % de goles | Por qué |
|---|---|---|
| 1'-15' | 11% | Equipos cautelosos al inicio (sobre todo J1) |
| 16'-30' | 14% | Empieza a abrirse el partido |
| 31'-45'+ | 18% | Cansancio leve, errores defensivos |
| 46'-60' | 17% | Salida nueva del entretiempo |
| 61'-75' | 19% | Cambios y desgaste físico aparecen |
| 76'-90'+ | 21% | Cansancio, repliegues, descuento largo |

Aplicar al elegir los minutos de los goles del partido. **Evitar concentrar todos los goles entre los minutos 20'-45'** (sesgo típico LLM). Repartir realista.

**🎬 Tipos de jugada de gol**

Distribución típica en Mundiales (usar como guía al describir cada gol):

| Tipo | Probabilidad |
|---|---|
| Jugada elaborada / pase filtrado | ~25% |
| Pelota parada (córner, tiro libre directo o indirecto) | ~25% |
| Contraataque | ~17% |
| Definición individual / regate | ~12% |
| Error defensivo / rebote | ~10% |
| Penal | ~8% (cada 12 partidos hay penal; conversión ~75%) |
| Cabezazo en juego (no córner) | ~3% |

**Si decís "tiro al ángulo, asistencia de X" en todos los goles, cae el realismo. Mezclá: un partido con 2-1 debería tener variedad en las jugadas (ej. uno por jugada elaborada, uno por pelota parada, descuento por error).**

**⏱ Tiempo de descuento**

- **Primer tiempo:** 1-4 minutos (promedio 2.5'). Más alto si hubo lesión o cambio.
- **Segundo tiempo:** 4-9 minutos (promedio 6-7' en Mundiales recientes por revisiones VAR). Más alto si hubo gol tardío, lesión, o muchos cambios.

**🏥 Lesiones y tarjetas — distribución por partido**

- **Amarillas:** 3-5 por partido típico. Más en J3 con definiciones.
- **Roja directa:** ~1 cada 20 partidos (5% por partido).
- **Doble amarilla → expulsión:** ~1 cada 12 partidos.
- **Lesión que saca al jugador antes del 90':** ~1 cada 3 partidos (33% por partido).

**Estado mutable que mantenemos durante la corrida:**
- Marcador local / visitante.
- Tarjetas activas por jugador (acumulada).
- Lesiones aparecidas.
- Cambios consumidos (máx 5 por equipo).
- Carga física por jugador (sube minuto a minuto según intensidad).
- Tarjetas amarillas activas (segunda amarilla → expulsión).

**Plan de partido del DT** (aplicar dinámicamente, sección 8.5 del spec):
- 60-75': si va perdiendo por 1, primer cambio ofensivo (+3 Ataque, −2 Defensa).
- 75-85': si va perdiendo por 2, doble cambio ofensivo (+5/-5).
- 75-90': si va ganando por 1, cambio defensivo (+3 Defensa, −2 Ataque).

**Formato del relato:**

Cada minuto se escribe una línea en una de estas formas:

```
- **<MM>'**: <descripción breve, 1 frase>
- **<MM>'** ⚽ **¡GOL DE <EQUIPO>!** <descripción del gol>. **<marcador>.**
- **<MM>'** 🟨 Amarilla a <jugador> (<EQUIPO>) por <motivo>.
- **<MM>'** 🟥 ¡Roja a <jugador> (<EQUIPO>)! <motivo>. <EQUIPO> se queda con 10.
- **<MM>'** 🔁 Cambio en <EQUIPO>: sale <jugador>, entra <jugador>.
- **<MM>'** 🏥 Lesión: <jugador> (<EQUIPO>) se retira con <molestia>. <duda/baja confirmada>.
- **<MM>'**: <jugada relevante sin gol — tiro, atajada, palo, despeje>.
```

Cubrir minutos 1 a 45 (primer tiempo), insertar `### Segundo tiempo` y cubrir 46 a 90. Agregar adicionados como `45'+1`, `45'+2`, ..., `90'+1`, etc. (4-7' a definir según incidencias del partido).

**Estilo del relato** (español argentino, no excesivo):
- Usar verbos en presente histórico ("Messi remata", no "Messi remató").
- Modismos sutiles permitidos: "la pisa", "la peina", "tira el centro", "la clava", "se la come". No abusar.
- Minutos sin acción: 1 frase neutra, no inventar drama ("circulación lenta en mitad de cancha", "saque lateral para X").

Al terminar:
- Marcador final.
- Lista de goleadores (minuto, jugador, equipo, tipo de gol, asistencia si la hubo).
- Lista de tarjetas (minuto, jugador, equipo, color).
- Lista de lesiones.
- Lista de cambios.
- Stats agregadas: posesión %, tiros, tiros al arco, córners, faltas, offsides.
- MVP: jugador con mayor impacto narrativo (gol + asistencias + intervenciones decisivas). Justificación en 1 frase.

### Paso 7 — Componer el MD del partido

Escribir `partidos/<YYYY-MM-DD>-<local-slug>-vs-<visitante-slug>.md` con esta estructura exacta:

```markdown
# <Local-bandera> <Local> <X> - <Y> <Visitante-bandera> <Visitante>
**Grupo <X> · Jornada <N> · <YYYY-MM-DD> · <HH:MM> · <Estadio, Ciudad>**

---

## 1. Contexto pre-partido

### Condiciones del día
- **Clima**: <condición>, <T°C>°C, humedad <H>%, viento <V> km/h <Dir>.
  - Fuente: <URL>
- **Estadio**: <nombre>, aforo <N>, <altitud si aplica>.
- **Horario local**: <HH:MM> (<mañana/tarde/noche>).
- **Asistencia estimada**: <N> (<descripción de localía>).

### Estado de los planteles
- **<Local>**: <plantel completo / X bajas>, carga física <N>/100. Moral <N>.
- **<Visitante>**: <ídem>.

### Historial reciente entre ambos (búsqueda en internet)
- <YYYY-MM-DD> — <competencia>: <resultado>.
- ...
- Resumen últimos N cruces: <X>V <Local>, <Y> empates, <Z>V <Visitante>.
- Fuentes consultadas: <URL1>, <URL2>.

(Si no hay antecedentes: "Sin antecedentes registrados — primera vez que se enfrentan.")

### Árbitro
- <Nombre, Nacionalidad> — estilo: <estricto/permisivo/equilibrado>.
- (o: "Por designar — estilo asumido equilibrado.")

### XI confirmados
- **<Local> (<esquema>)**: <ARQ>; <DEF1>, <DEF2>, ..., ; <VOL1>, ..., ; <DEL1>, ..., .
- **<Visitante> (<esquema>)**: <ídem>.

---

## 2. Probabilidades (10 corridas Monte Carlo)
- Victoria <Local>: **<X>%**
- Empate: **<Y>%**
- Victoria <Visitante>: **<Z>%**
- Goles esperados (promedio MC): <Local> <a.b> – <Visitante> <c.d>
- Resultado más probable: **<X-Y> <equipo>** (<n>/10 corridas)
- Distribución de resultados:
  - <a-b>: <X>% · <c-d>: <Y>% · ...

---

## 3. Relato minuto a minuto

### Primer tiempo

- **1'**: ...
- ...
- **45'+<N>**: Final del primer tiempo.

### Segundo tiempo

- **46'**: ...
- ...
- **90'+<N>**: Final del partido.

---

## 4. Estadísticas finales

| Estadística | <Local> | <Visitante> |
|---|---|---|
| Posesión | <X>% | <Y>% |
| Tiros (al arco) | <a (b)> | <c (d)> |
| Córners | <X> | <Y> |
| Faltas | <X> | <Y> |
| Amarillas | <X> | <Y> |
| Rojas | <X> | <Y> |
| Offsides | <X> | <Y> |

### Goleadores
- <MM>' <jugador> (<eq>) — <tipo de jugada>, asist. <jugador o "—">
- ...

### Tarjetas
- 🟨 <MM>' <jugador> (<eq>), ...
- 🟥 <MM>' <jugador> (<eq>) (si hubo).

### Lesiones
- <jugador> (<eq>): <minuto> con <molestia>. <duda/baja confirmada para próximo partido>.

### MVP
**<Jugador>** — <una frase justificando>.

---

## 5. Actualización a los MDs

### <Local>
- Carga física: <antes> → <después>
- Moral: <antes> → <después> (<+/- razón>)
- Lesionados nuevos: <lista o "—">
- En capilla (1 amarilla acumulada): <lista o "—">
- Suspendidos para próximo partido (2 amarillas): <lista o "—">
- Historial Mundial: +1 PJ, <+1 G / +1 E / +1 P>, +<n> GF, +<m> GC

### <Visitante>
- ... (ídem)

### Fixture y grupos
- `fixture.md`: partido marcado como ✅ Jugado, resultado <X-Y>.
- `grupos.md`: tabla del Grupo <X> actualizada.
```

Generar el archivo con la información acumulada de los pasos previos.

### Paso 8 — Actualizar archivos vivos

#### 8.1 `equipos/<local>.md` y `equipos/<visitante>.md`

Para cada equipo, modificar en su MD:

1. **Stats del equipo**:
   - Moral: nueva = anterior + ajuste. Ajustes:
     - Victoria por ≥2 goles: +5.
     - Victoria por 1 gol: +4.
     - Empate (favorito): −2.
     - Empate (no favorito): +2.
     - Derrota por 1 gol: −5.
     - Derrota por ≥2 goles: −8.
     - Clampar a [40, 99].
   - Físico (carga acumulada): subir entre 10 y 18 según intensidad del partido (más alto si calor o tiempo extra de adicionados).

2. **Plantilla**:
   - Para cada jugador que jugó: bajar levemente su columna "Estado" (100% → 95% si jugó 90', 100% → 98% si jugó <30', etc.). Documentar como porcentaje.
   - Para lesionados: bajar Estado al % correspondiente (45-70% según gravedad) y agregarlos a "Lesionados / Suspendidos".

3. **Lesionados / Suspendidos**:
   - Agregar lesionados nuevos del partido.
   - Si un lesionado anterior ya no juega este partido, considerar baja confirmada o ya recuperado según contexto.
   - Marcar suspendidos por 2 amarillas o roja.

4. **Carga física acumulada en el Mundial**: actualizar al nuevo valor.

5. **Historial en el Mundial (en curso)**: actualizar la tabla y la lista de goleadores propios.

6. **Forma reciente**: agregar este partido al inicio de la lista (mantener máx. 5).

#### 8.2 `fixture.md`

1. En la fila del partido jugado:
   - Estado: `⏳ Pendiente` → `✅ Jugado`.
   - Resultado: `<X>-<Y>`.
   - Archivo: `[ver](partidos/<YYYY-MM-DD>-<local-slug>-vs-<visitante-slug>.md)`.

2. Encabezado del archivo:
   - "Último partido simulado": actualizar al partido recién jugado.
   - "Próximo partido pendiente": calcular el siguiente `⏳ Pendiente` por fecha+hora ascendente y reflejarlo.

#### 8.3 `grupos.md`

1. Identificar el grupo del partido (del fixture).
2. Recalcular la tabla del grupo a partir de **todos** los partidos `✅ Jugado` de ese grupo en el fixture (no llevar contador incremental; recalcular desde cero cada vez para evitar drift). Para cada equipo del grupo:
   - PJ = partidos `✅ Jugado` en los que figura.
   - G/E/P según resultados.
   - GF / GC sumados de los resultados.
   - DG = GF − GC.
   - Pts = G × 3 + E.
3. Ordenar las filas de la tabla:
   1. Pts desc.
   2. DG desc.
   3. GF desc.
   4. (Resto de criterios FIFA → notar en pie de tabla si hay empate después del paso 3.)
4. Línea "Actualizado tras": fecha + partido recién jugado.
5. Si todos los partidos del grupo (`PJ = 3` para todos) están jugados, agregar nota al final de la tabla:
   > "✅ Grupo cerrado. Clasificados: <eq1>, <eq2>. Tercero: <eq3>."
6. Si los 12 grupos están cerrados, también actualizar el bloque "Clasificados a octavos" con los 16 directos (1° y 2° de cada grupo) + los 8 mejores terceros (ordenados por Pts → DG → GF entre los 12 terceros).

#### 8.4 Caso de re-simulación

Si en el Paso 1 el usuario aceptó re-simular un partido ya jugado:
1. Leer el archivo de partido anterior en `partidos/`.
2. Aplicar el "diff inverso" de su sección 5 a los MDs de los dos equipos: revertir Moral, carga, lesionados, suspendidos, historial.
3. En `fixture.md`, marcar provisoriamente el partido como `⏳ Pendiente`.
4. Continuar con el flujo normal (escribir nueva versión del MD del partido).

### Paso 9 — Resumen al usuario

Imprimir:

> ⚽ <Local> <X> - <Y> <Visitante> (Grupo <X>, Jornada <N>)
> Goleadores: <minuto> <jugador> (<eq>), ...
> MVP: <jugador> (<eq>)
> Archivo: `partidos/<YYYY-MM-DD>-<local-slug>-vs-<visitante-slug>.md`
>
> Próximo partido pendiente: <YYYY-MM-DD HH:MM> <local> vs <visitante> en <sede>.

### Manejo de errores

| Error | Acción |
|---|---|
| `fixture.md` no existe | Abortar: "Falta inicializar. Corré `inicializar-mundial` primero." |
| Equipo del partido no tiene MD en `equipos/` | Abortar con nombre del archivo faltante. |
| El partido pedido por argumento no existe en el fixture | Abortar con mensaje claro. |
| WebSearch del clima sin resultados útiles tras 2 intentos | Asumir condiciones normales del mes para la latitud y dejar nota en el MD. |
| WebSearch del historial cara a cara sin resultados | "Sin antecedentes registrados — primera vez que se enfrentan." |
| WebSearch del árbitro sin resultados | Árbitro ficticio con estilo random ponderado. Anotar como "por designar". |
| Sustitución entra a jugador suspendido por amarillas | Imposible (verificar antes de hacer la sustitución; usar el siguiente suplente). |
| Si tras todo el flujo el marcador de la corrida oficial difiere de la moda MC | OK, es esperable. No reintentar. |

### Validación final

Antes de declarar éxito, verificar:

- [ ] Existe el archivo `partidos/<YYYY-MM-DD>-<local-slug>-vs-<visitante-slug>.md`.
- [ ] Su sección 3 tiene líneas para todos los minutos del 1 al 90 + adicionados (sin huecos).
- [ ] Sección 5 ("Actualización a los MDs") lista explícitamente los cambios.
- [ ] Los MDs de los dos equipos reflejan los cambios listados en la sección 5.
- [ ] `fixture.md` marca el partido como `✅ Jugado` y la línea "Próximo partido pendiente" del encabezado es coherente.
- [ ] `grupos.md` tiene la tabla del grupo actualizada y la línea "Actualizado tras" refleja el partido.

Si algo falla, reportarlo al usuario sin marcar el partido como completado.
