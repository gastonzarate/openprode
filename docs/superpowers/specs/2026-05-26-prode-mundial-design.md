# Simulador del Mundial 2026 para completar el Prode

**Fecha**: 2026-05-26
**Estado**: aprobado por el usuario, pendiente de plan de implementación

## 1. Objetivo

Construir un sistema en Markdown + skills locales de Claude Code que permita simular partido a partido el Mundial 2026 (FIFA World Cup 2026, sede USA/Canadá/México, 48 selecciones) para generar predicciones de un prode. Cada simulación produce un resultado, un relato minuto a minuto en español argentino, estadísticas completas, y actualiza el estado de los equipos involucrados para que los partidos siguientes lo tomen en cuenta.

El alcance inicial de este spec es **la fase de grupos** (72 partidos). Las eliminatorias quedan fuera y se diseñarán en un spec posterior, una vez completada la fase de grupos.

## 2. Decisiones de diseño aprobadas

- **Fuente de datos inicial**: WebSearch / WebFetch contra fuentes oficiales (FIFA, federaciones, Wikipedia).
- **Motor de simulación**: narrativo guiado por las stats numéricas de los equipos, con factores contextuales (clima, sede, horario, carga física, lesionados, suspendidos, moral, importancia, árbitro).
- **Cobertura del relato**: minuto a minuto, 1' a 90'+ adicionados. Una línea por minuto, incluso en minutos sin acción.
- **Probabilidades**: 10 corridas Monte Carlo silenciosas por partido para reportar distribución de resultados, además de 1 corrida "oficial" que es la que cuenta como predicción del prode y la que actualiza los MDs.
- **Reproducibilidad**: no se usa seed. Cada corrida es única.
- **Output del prode**: resultado exacto, goleadores con minuto, tarjetas, lesiones, MVP, estadísticas; todo se vuelca también al MD del equipo.
- **Workflow**: se simula un partido a la vez, siguiendo el orden cronológico del fixture. Hay un archivo de tracking (`fixture.md`) que marca el próximo partido pendiente y conserva el resultado de cada partido jugado.
- **Idioma**: español argentino (relato y archivos).
- **Arquitectura**: dos skills locales (`inicializar-mundial`, `simular-partido`) y carpetas de datos en Markdown. Sin código, sin dependencias.
- **Enriquecimiento en vivo en cada partido**: el clima de la sede, el árbitro designado y el historial cara a cara entre las dos selecciones se buscan en internet en el momento de la simulación.

## 3. Estructura del repositorio

```
prode/
├── README.md                    # Guía de uso, fuentes consultadas, lógica de stats
├── fixture.md                   # Cronograma completo + estado y resultado de cada partido
├── grupos.md                    # Tablas de posiciones por grupo (12 grupos)
├── equipos/
│   ├── argentina.md
│   ├── brasil.md
│   └── ... (48 archivos, uno por selección)
├── partidos/
│   └── YYYY-MM-DD-<local-slug>-vs-<visitante-slug>.md
├── docs/
│   └── superpowers/
│       └── specs/
│           └── 2026-05-26-prode-mundial-design.md
└── .claude/
    └── skills/
        ├── inicializar-mundial/
        │   └── SKILL.md
        └── simular-partido/
            └── SKILL.md
```

## 4. Estructura del MD de equipo

Cada archivo `equipos/<pais>.md` contiene:

```markdown
# <País> <bandera>

## Información general
- Confederación: <UEFA / CONMEBOL / CONCACAF / AFC / CAF / OFC>
- DT: <nombre>
- Capitán: <nombre>
- Apodo: <apodo>
- Ranking FIFA: <número>
- Grupo en el Mundial: <A-L>
- Estadio "local" preferido (si aplica): <—|estadio>

## Estilo de juego
- Esquema base: <ej. 4-3-3>
- Estilo: <texto libre>
- Fortalezas: <lista>
- Debilidades: <lista>

## Stats del equipo (0-100)
- Ataque
- Mediocampo
- Defensa
- Arco
- Físico
- Moral
- Cohesión
- Experiencia

## Plantilla (26 jugadores, regla FIFA 2026)
| # | Jugador | Pos | Edad | Club | Ataque | Defensa | Físico | Técnica | Estado |

## XI titular probable
<esquema y nombres>

## Lesionados / Suspendidos
- (vacío al inicio; se llena con cada partido)

## Forma reciente (pre-Mundial, últimos 5 partidos)
- <fecha> vs <rival>: <resultado>

## Carga física acumulada en el Mundial
- 0/100 (sube ~10-15 por partido)

## Historial en el Mundial (en curso)
- (vacío al inicio; se va llenando: PJ, G, E, P, GF, GC, goleadores propios)

## Historial vs rivales del grupo
- vs <Rival1>: últimos enfrentamientos
- vs <Rival2>: ...
- vs <Rival3>: ...
```

Las stats numéricas son lo que usa el simulador como peso para los eventos. La inferencia inicial de stats se documenta en `README.md` (combinación de ranking FIFA + nivel de los clubes de los jugadores + reputación general).

## 5. Estructura del `fixture.md`

Es la fuente de verdad del próximo partido a jugar. El skill busca el primer `Estado: ⏳ Pendiente` ordenado por fecha + hora.

```markdown
# Fixture Mundial 2026

> Último partido simulado: <YYYY-MM-DD — local X-Y visitante>
> Próximo partido pendiente: <YYYY-MM-DD — local vs visitante (HH:MM, sede)>

## Fase de Grupos

### Jornada 1

| Fecha | Hora | Sede | Grupo | Local | Visitante | Estado | Resultado | Archivo |
|-------|------|------|-------|-------|-----------|--------|-----------|---------|
| 2026-06-11 | 20:00 | Azteca, CDMX | A | México | Marruecos | ✅ Jugado | 2-1 | [ver](partidos/2026-06-11-mexico-vs-marruecos.md) |
| ... |

### Jornada 2
### Jornada 3
```

**Mecánica:**
- El skill busca el primer registro `⏳ Pendiente` por fecha+hora ascendente.
- Al terminar el partido, actualiza el estado a `✅ Jugado`, llena Resultado y Archivo, y refresca las dos líneas del encabezado.
- Si dos partidos comparten fecha+hora exacta, desempata el orden alfabético del local.

## 6. Estructura del `grupos.md`

Tablas de posiciones de los 12 grupos, recalculadas después de cada partido.

```markdown
# Tablas de posiciones — Fase de Grupos

> Actualizado tras: <YYYY-MM-DD — local X-Y visitante>

## Grupo A
| Pos | Equipo | PJ | G | E | P | GF | GC | DG | Pts |
| ... |

## Grupo B
...

## Clasificación a octavos
Pasan los **2 primeros de cada grupo** + **8 mejores terceros** (Mundial de 48, formato 2026).
```

**Criterio de orden (FIFA):** Puntos → DG → GF → resultado entre empatados → fair play → sorteo. El simulador aplica automáticamente los 3 primeros; los desempates posteriores quedan documentados como nota cuando se den.

## 7. Estructura del MD de partido (output principal)

Archivo: `partidos/YYYY-MM-DD-<local-slug>-vs-<visitante-slug>.md`.

Secciones:

1. **Cabecera** — Equipos, marcador, grupo, jornada, fecha, hora, sede.
2. **Contexto pre-partido**:
   - Condiciones del día: clima (con URL de fuente), estadio (aforo, altitud), horario, asistencia esperada, localía.
   - Estado de los planteles: carga física, moral, lesionados, suspendidos.
   - Historial reciente entre ambos (búsqueda en internet, últimos 5-10 enfrentamientos con URLs de fuentes). Si nunca se enfrentaron: "Sin antecedentes registrados — primera vez que se enfrentan".
   - XI confirmados.
3. **Probabilidades (10 corridas Monte Carlo)**:
   - % Victoria local, % Empate, % Victoria visitante.
   - Goles esperados promedio (xG agregado de las 10 corridas).
   - Resultado más probable.
   - Distribución completa de marcadores aparecidos.
4. **Relato minuto a minuto** (corrida oficial, 1' a 90'+ adicionados):
   - Una línea por minuto.
   - Emojis convencionales: ⚽ gol, 🟨 amarilla, 🟥 roja, 🔁 cambio, 🏥 lesión.
5. **Estadísticas finales**:
   - Tabla: posesión, tiros (al arco), córners, faltas, amarillas, rojas, offsides.
   - Goleadores: minuto, jugador, asistencia, tipo de jugada.
   - Tarjetas con minuto.
   - Lesiones con minuto y diagnóstico breve.
   - MVP con justificación corta.
6. **Actualización a los MDs**:
   - "Diff" auditable de cada equipo: carga física antes → después, moral antes → después, nuevos lesionados, suspendidos por amarilla, historial Mundial sumado.
   - Confirmación de actualización a `fixture.md` y `grupos.md`.

## 8. Marco narrativo del motor de simulación

### 8.1 Factores fijos del partido

Se determinan al inicio de cada partido y se mantienen hasta el final.

| Factor | Determinación |
|---|---|
| Clima | WebSearch del pronóstico real para esa fecha y ciudad. Pronóstico si la fecha está dentro de ~10 días; promedio histórico mensual si no. URL de la fuente queda registrada en el MD del partido. |
| Sede | Tomada del fixture; se valida contra FIFA si el fixture parece desactualizado. |
| Horario | Tomada del fixture; se valida contra FIFA. |
| Localía / Asistencia | Localía: se aplica para selecciones anfitrionas (México/USA/Canadá) jugando en sus estadios. Asistencia esperada: capacidad del estadio, salvo nota oficial. |
| Importancia | Calculada del estado del grupo: J1 (nervios de debut), J2 (intermedia), J3 (define clasificación). |
| Árbitro | WebSearch del árbitro designado si está disponible (con su estilo conocido). Si no, random ponderado: permisivo / estricto / equilibrado. |

**Efectos:**
- Lluvia → +5% errores, levemente ↓ Técnica.
- Calor >30°C → ↓ Físico tras 60'.
- Viento fuerte → ↓ efectividad de pases largos y centros.
- Altitud Azteca → ↓ Físico del visitante en el segundo tiempo.
- Localía → +5 a Moral del local.
- J1 → ↑ errores, ↓ ritmo primeros 15'.
- J3 con algo en juego → ↑ intensidad.
- Árbitro estricto → ×1.5 tarjetas; permisivo → ×0.6.

### 8.2 Factores variables (de los MDs de equipo)

Para cada equipo se calcula una **Fuerza Efectiva** al inicio del partido (concepto-guía, no fórmula rígida):

```
FuerzaEfectiva = Ataque*0.35 + Mediocampo*0.25 + Defensa*0.20 + Arco*0.10
               + Físico*0.05 + Cohesión*0.05
               + ajustes por Moral, Carga, Localía, Clima, Bajas
```

- Moral alta (>80) → +3 a la fuerza efectiva. Moral baja (<60) → −5.
- Carga física entra como degradación a partir del minuto 60: cada 20 puntos de carga acumulada resta 1-2 a Físico durante el segundo tiempo.
- Bajas titulares (lesionados/suspendidos): el reemplazo entra con su rating real. Si el rating cae >10 vs el titular, además ↓2 a Cohesión por ese partido.

### 8.3 Eventos por minuto y sus probabilidades base

| Evento | Prob base / minuto | Resultado típico |
|---|---|---|
| Sin acción relevante | ~55% | Una línea breve de contexto |
| Posesión / circulación con riesgo bajo | ~20% | Una jugada descrita sin tiro |
| Tiro / situación de gol | ~14% | xG asignado → tirada decide gol / atajada / afuera / palo |
| Falta táctica / amarilla | ~5% | Posible tarjeta según árbitro y minuto |
| Córner / pelota parada | ~4% | Mini-evento con tiro posible |
| Lesión | ~0.5% | Sale el jugador; si era titular ↑ desgaste defensivo |
| Roja directa | ~0.1% | Inferioridad numérica el resto del partido |
| Cambio | minuto 60+ según situación | Decisión del DT |

Las probabilidades se sesgan por **dominio territorial** del minuto, que sale de comparar la Fuerza Efectiva de ambos equipos +/- ruido.

### 8.4 Algoritmo conceptual por minuto

```
Para cada minuto m de 1 a 90 + adicionados:
  1. Decidir dominio del minuto (sesgo por Fuerza Efectiva relativa).
  2. Decidir tipo de evento (tabla 8.3 con ajustes).
  3. Si es tiro:
     - xG según calidad de la jugada y rating del rematador.
     - tirada vs (xG, rating del arquero, clima).
     - resultado: gol / atajada / afuera / palo / bloqueo.
  4. Si es falta peligrosa o táctica:
     - Tirada de tarjeta según árbitro y zona/momento del partido.
  5. Si es lesión: elegir jugador (sesgo por minutos jugados + estado pre-partido).
  6. Escribir 1 línea de relato en español argentino.
  7. Actualizar estado mutable: marcador, tarjetas activas, lesiones, carga.
```

### 8.5 Plan de partido (decisiones del DT)

- Minuto 1–60: cada equipo juega su esquema base del MD.
- Minuto 60–75: si va perdiendo por 1, primer cambio ofensivo (Ataque +3, Defensa −2).
- Minuto 75–85: si va perdiendo por 2, doble cambio ofensivo (Ataque +5, Defensa −5).
- Minuto 75–90: si va ganando por 1, cambio defensivo (Defensa +3, Ataque −2).
- 2 amarillas → el jugador sale y el equipo termina con 10 (no hay cambio).

### 8.6 Monte Carlo silencioso vs corrida oficial

- 10 corridas MC silenciosas: mismo algoritmo sin generar relato textual. Solo se registra marcador final, goleadores y stats agregadas. Se reportan probabilidades agregadas en la sección 3 del MD del partido.
- 1 corrida oficial: mismo algoritmo escribiendo cada minuto. Es la corrida que cuenta para el prode y la que actualiza los MDs.
- Las 11 corridas son independientes (sin seed). La oficial puede no coincidir con la moda de MC.

### 8.7 Reglas duras

- Mundial 2026 fase de grupos: 90 minutos + adicionados (4–7' según incidencias). Sin tiempo extra ni penales.
- Suspensión por 2 amarillas acumuladas en distintos partidos → no juega el siguiente.
- 5 cambios permitidos por equipo.

## 9. Skill A — `inicializar-mundial`

**Entrada**: ninguna (argumento opcional `--force` para regeneración).

**Flujo:**

1. **Verificar estado**: si ya existen `fixture.md`, `grupos.md` o `equipos/*.md`, avisar y pedir confirmación antes de sobreescribir.
2. **Bajar fixture oficial 2026**:
   - WebSearch: "FIFA World Cup 2026 fixture schedule full" + Wikipedia.
   - WebFetch a la fuente más completa.
   - Parsear los 72 partidos de fase de grupos: fecha, hora local, sede, ciudad, grupo, local, visitante.
3. **Bajar plantillas y datos de 48 selecciones**:
   - Para cada selección: WebSearch + WebFetch para obtener DT, capitán, ranking FIFA, plantel preliminar de 26, club y edad de cada jugador.
   - Fuentes preferidas: FIFA y federación nacional. Wikipedia como fallback.
   - Stats numéricas (0-100): se infieren a partir del ranking FIFA + nivel del club de cada jugador + reputación general. Lógica documentada en `README.md`.
4. **Generar archivos**:
   - `equipos/<pais>.md` × 48.
   - `fixture.md` con los 72 partidos ordenados.
   - `grupos.md` con las 12 tablas en cero.
   - `README.md` con instrucciones de uso y la lógica de generación de stats.
5. **Resumen final**: cantidad de equipos generados, cantidad de partidos, fecha del primer partido.

**Errores:**
- WebSearch sin resultados → reintentar con consulta alternativa; si sigue fallando avisar y pedir intervención manual.
- Plantilla incompleta para una selección → completar con "TBD" y marcar el archivo del equipo con un aviso al principio.

## 10. Skill B — `simular-partido`

**Entrada:**
- Sin argumentos → simula el próximo partido pendiente por fecha+hora.
- Con argumento `local vs visitante` → fuerza ese partido específico (debe estar pendiente).

**Flujo:**

1. **Localizar partido**:
   - Leer `fixture.md`.
   - Sin argumento: tomar el primer `⏳ Pendiente` por fecha+hora ascendente.
   - Con argumento: buscar por equipos, validar estado pendiente.
   - Si no hay nada pendiente: anunciar "Fase de grupos completa" y salir.
2. **Cargar contexto**:
   - Leer `equipos/<local>.md` y `equipos/<visitante>.md`.
   - Leer sede, fecha, hora, grupo, jornada del `fixture.md`.
3. **Enriquecer con datos de internet**:
   - WebSearch del clima para la fecha y ciudad de la sede (pronóstico si la fecha está dentro del horizonte de pronóstico; promedio histórico mensual si no).
   - WebSearch del historial cara a cara entre las dos selecciones (últimos 5-10 enfrentamientos: fecha, competencia, resultado).
   - WebSearch del árbitro designado si está disponible.
   - Si la sede u horario del fixture parecen desactualizados, validar contra FIFA.
   - Registrar las URLs consultadas para citarlas en el MD del partido.
4. **Establecer factores fijos** (sección 8.1).
5. **Correr 10 corridas Monte Carlo silenciosas** (sección 8.6):
   - Aplicar el algoritmo de 8.4 sin generar texto.
   - Guardar marcador final y goleadores por corrida.
   - Agregar probabilidades: %V/E/D, goles esperados promedio, distribución de marcadores, resultado más probable.
6. **Correr la corrida oficial** (sección 8.6):
   - Mismo algoritmo generando 1 línea de relato por minuto (1' a 90'+ adicionados).
   - Mantener estado mutable: marcador, tarjetas activas, lesiones, sustituciones, carga.
7. **Componer el MD del partido** (sección 7):
   - Secciones 1 a 6 completas.
   - Calcular MVP a partir de stats de la corrida oficial.
   - Computar el "diff" para los MDs.
8. **Escribir archivos**:
   - `partidos/YYYY-MM-DD-<local-slug>-vs-<visitante-slug>.md`.
   - Actualizar `equipos/<local>.md` y `equipos/<visitante>.md`: carga física, moral, lesionados, suspendidos, historial Mundial, forma reciente.
   - Actualizar `fixture.md`: marcar `✅ Jugado`, agregar resultado y link, refrescar las dos líneas del encabezado.
   - Actualizar `grupos.md`: recalcular la tabla del grupo, refrescar línea "Actualizado tras".
9. **Resumen al usuario**: marcador, goleadores, próximo partido pendiente.

**Errores:**
- Equipo no encontrado en `equipos/` → abort con mensaje claro.
- Partido ya `✅ Jugado` → confirmar si se quiere re-simular.
- Fecha+hora repetida entre dos partidos → desempate por orden alfabético del local.
- WebSearch del clima sin resultados → caer a promedio histórico estimado y dejar nota en el MD.
- WebSearch del historial cara a cara sin resultados → "Sin antecedentes registrados — primera vez que se enfrentan".

## 11. Datos efímeros vs permanentes

- **Permanente (en MDs)**: stats numéricas del equipo (evolucionan), plantilla, lesionados, suspendidos, carga, moral, historial Mundial.
- **Efímero (solo dentro de la simulación)**: factores fijos del día, eventos minuto a minuto, estado del marcador durante la corrida. No se persisten fuera del MD del partido.

## 12. Fuera de alcance (a definir en specs posteriores)

- Fase eliminatoria (32avos, 16avos, cuartos, semis, tercer puesto, final).
- Prórroga y penales.
- Visualizaciones / dashboards web.
- Ejecución en lote o por jornada.

## 13. Criterio de éxito

- `inicializar-mundial` corre 1 vez y deja el repo listo con 48 MDs de equipo + `fixture.md` + `grupos.md` + `README.md`.
- `simular-partido` puede correrse 72 veces en orden cronológico y cada corrida deja:
  - 1 archivo nuevo en `partidos/` con relato minuto a minuto, probabilidades MC, stats y MVP.
  - `equipos/<local>.md` y `equipos/<visitante>.md` actualizados con el diff documentado.
  - `fixture.md` con el partido marcado como jugado.
  - `grupos.md` con la tabla del grupo recalculada.
- Tras los 72 partidos, `grupos.md` muestra las 12 tablas finales y la lista de clasificados a octavos (16 directos + 8 mejores terceros).
