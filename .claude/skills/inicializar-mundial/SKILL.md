---
name: inicializar-mundial
description: Use when starting the Prode Mundial 2026 project from scratch. Downloads the official FIFA World Cup 2026 fixture and the squads of all 48 qualified national teams via WebSearch/WebFetch, then generates equipos/<pais>.md × 48, fixture.md and grupos.md. Runs ONCE per project lifetime.
---

# Inicializar Mundial 2026

Bootstrap one-shot del proyecto Prode Mundial 2026. Descarga datos oficiales del Mundial 2026 y genera todos los archivos base.

**Cuándo usar:** solo al arrancar el proyecto, antes de simular cualquier partido. Si ya existen `fixture.md`, `grupos.md` o archivos en `equipos/`, pedir confirmación antes de sobreescribir.

**Idioma:** todo el contenido generado va en **español argentino**.

## Flujo

### Paso 1 — Verificar estado previo

Antes de hacer nada:

1. Revisar si existen `fixture.md`, `grupos.md` o archivos dentro de `equipos/` en el working directory raíz del proyecto.
2. Si **alguno existe**, mostrarle al usuario qué hay y preguntar:
   > "Ya existen archivos del Mundial inicializado. ¿Querés sobreescribirlos? Esto borra el progreso actual."
   - Si responde "no" o equivalente: cortar la ejecución acá.
   - Si responde "sí" o equivalente: continuar y sobreescribir todo.
3. Si **nada existe**: continuar directamente.

No tocar `partidos/` en este paso — el bootstrap nunca borra partidos ya simulados, pero si el usuario regenera fixture y equipos el contenido de `partidos/` queda inconsistente. Avisar de eso en el prompt de confirmación.

### Paso 2 — Descargar fixture oficial del Mundial 2026

1. Hacer WebSearch con consultas:
   - `"FIFA World Cup 2026 schedule full fixture group stage"`
   - `"Mundial 2026 fixture fase de grupos calendario completo"`
   - `"World Cup 2026 Wikipedia group stage matches"`

2. Tomar la(s) fuente(s) más completa(s) (Wikipedia suele tener el cuadro completo con sedes, fechas y horarios). Hacer WebFetch con un prompt que extraiga, para cada uno de los 72 partidos:
   - Fecha (YYYY-MM-DD)
   - Hora local (HH:MM, 24 h)
   - Sede (estadio + ciudad)
   - Grupo (A-L)
   - Local
   - Visitante
   - Jornada (1, 2 o 3)

3. Validar: deben salir exactamente **72 partidos** (48 equipos × 3 partidos / 2) y **12 grupos × 4 equipos = 48 equipos**. Si la cuenta no da, reintentar con otra fuente o pedir intervención del usuario.

4. Ordenar los partidos por fecha + hora ascendente para escribir el `fixture.md`.

5. Guardar la lista en memoria — se va a usar en los pasos 3 y 5.

### Paso 3 — Descargar plantillas y datos de las 48 selecciones

A partir de los partidos del fixture, extraer la lista única de 48 selecciones clasificadas.

Para **cada** selección, hacer WebSearch + WebFetch para obtener:

- Confederación (UEFA / CONMEBOL / CONCACAF / AFC / CAF / OFC)
- DT (nombre y nacionalidad)
- Capitán
- Apodo
- Ranking FIFA actual (mes más cercano a 2026-05-26)
- Estadio "local" preferido si aplica (México, USA, Canadá en sus estadios)
- Plantel preliminar de 26 jugadores (regla FIFA 2026), con para cada jugador:
  - Dorsal (si está confirmado)
  - Nombre
  - Posición (ARQ / DEF / VOL / DEL)
  - Edad
  - Club
- XI titular probable + esquema (ej. "4-3-3: Arquero; Lateral der, Central, Central, Lateral izq; Volante def, Interior, Interior; Extremo, 9, Extremo")
- Forma reciente: últimos 5 partidos pre-Mundial con resultado y rival.

**Fuentes preferidas (en este orden):**
1. Sitio oficial de la federación nacional (afa.com.ar, fmf.com.mx, etc.).
2. FIFA.com
3. Wikipedia (artículo de la selección + artículo del Mundial 2026).
4. Transfermarkt para verificar club y dorsal.

**Si falta información** para una selección (típicamente plantel no confirmado todavía):
- Completar lo que se pueda.
- Marcar los campos faltantes con `TBD`.
- Insertar al principio del MD del equipo un bloque:
  ```
  > ⚠️ Datos incompletos al momento de inicializar. Última actualización: <fecha>. Volver a correr la skill o editar manualmente cuando la información esté disponible.
  ```

Guardar los datos en memoria — se van a usar en el paso 4 (inferencia de stats) y paso 5 (escritura de archivos).

### Paso 4 — Inferir stats numéricas (0-100)

Para cada selección, calcular las 8 stats del equipo y el rating individual de cada jugador.

**Rating individual del jugador (0-100):**

| Nivel del club | Rating base |
|---|---|
| Top-3 mundial (Real Madrid, City, PSG, Bayern, Liverpool, Barça, Arsenal, Inter) | 88-95 |
| Top-5 ligas europeas top (Atleti, Milan, Napoli, Dortmund, Atalanta, etc.) | 80-87 |
| Primera europea sólida (Premier media, La Liga media, Serie A media, Bundesliga media, Ligue 1 top) | 73-80 |
| Primera europea de cola / segundas divisiones top / MLS top | 65-72 |
| Ligas sudamericanas top (Boca, River, Palmeiras, Flamengo, Fluminense, etc.) | 65-75 |
| Resto (otras ligas, segundas divisiones medias) | 50-64 |

Ajustes individuales:
- Edad >34: −3.
- Edad 32-34: −1.
- Edad <21 con titularidad en club top: +2 (proyección).
- Capitán: +2.
- Si la fuente menciona lesión o forma muy floja reciente: −3.

Asignar Posición → bucket (ARQ, DEF, VOL, DEL).

**Stats del equipo (0-100):**

- **Ataque**: promedio de los ratings de los DEL del XI titular + 0.3 × rating del mejor VOL.
- **Mediocampo**: promedio de los ratings de los VOL del XI titular.
- **Defensa**: promedio de los ratings de los DEF del XI titular.
- **Arco**: rating del ARQ titular.
- **Físico**: 75 base − 0.5 × (edad promedio del XI − 27). Ajuste de altitud/calor: ignorar acá (se aplica por partido).
- **Moral**: 70 base + ajuste por forma reciente (últimos 5): V = +3, E = +1, D = −2.
- **Cohesión**: 90 si el DT lleva ≥2 años, 75 si <2 años, 60 si hubo cambio en los últimos 6 meses.
- **Experiencia**: promedio de los ratings de Mundiales/torneos continentales previos jugados por el XI titular (asumir 80 si ≥2 Mundiales jugados, 70 si 1, 55 si ninguno).

Todas las stats clampadas a [40, 99]. No usar 100 ni <40.

Documentar la fórmula usada como comentario al inicio del MD del equipo (ya está cubierto en `README.md`, no repetirlo en cada equipo).

### Paso 5 — Escribir los archivos

#### 5.1 `equipos/<pais-slug>.md` × 48

Slug del archivo: nombre del país en minúscula sin tildes y guiones por espacios (`argentina.md`, `corea-del-sur.md`, `cabo-verde.md`).

Plantilla exacta:

````markdown
# <País> <bandera-emoji>

## Información general
- Confederación: <UEFA / CONMEBOL / CONCACAF / AFC / CAF / OFC>
- DT: <nombre>
- Capitán: <nombre>
- Apodo: <apodo>
- Ranking FIFA: <número>
- Grupo en el Mundial: <A-L>
- Estadio "local" preferido (si aplica): <— | nombre del estadio>

## Estilo de juego
- Esquema base: <4-3-3 | 4-4-2 | 3-5-2 | etc.>
- Estilo: <descripción 1-2 líneas>
- Fortalezas: <lista corta>
- Debilidades: <lista corta>

## Stats del equipo (0-100)
- Ataque: <n>
- Mediocampo: <n>
- Defensa: <n>
- Arco: <n>
- Físico: <n>
- Moral: <n>
- Cohesión: <n>
- Experiencia: <n>

## Plantilla (26 jugadores, regla FIFA 2026)
| # | Jugador | Pos | Edad | Club | Ataque | Defensa | Físico | Técnica | Estado |
|---|---------|-----|------|------|--------|---------|--------|---------|--------|
| <#> | <nombre> | <ARQ/DEF/VOL/DEL> | <edad> | <club> | <n> | <n> | <n> | <n> | 100% |
...

## XI titular probable
<esquema>: <nombre1>; <nombre2>, <nombre3>, ...

## Lesionados / Suspendidos
- (ninguno al inicio del Mundial)

## Forma reciente (pre-Mundial, últimos 5 partidos)
- <YYYY-MM-DD> vs <rival>: <resultado>
- ...

## Carga física acumulada en el Mundial
- 0/100

## Historial en el Mundial (en curso)
| PJ | G | E | P | GF | GC | DG | Pts |
|----|---|---|---|----|----|----|-----|
| 0  | 0 | 0 | 0 | 0  | 0  | 0  | 0   |

### Goleadores propios
- (ninguno al inicio)

## Historial vs rivales del grupo
- vs <Rival1>: <últimos enfrentamientos, breve>
- vs <Rival2>: <...>
- vs <Rival3>: <...>
```

Para "Ataque/Defensa/Físico/Técnica" de cada jugador en la tabla: derivados del rating individual y la posición. Aproximación:
- DEL: Ataque = rating, Defensa = rating × 0.3, Físico = rating × 0.85, Técnica = rating × 0.95.
- VOL: Ataque = rating × 0.7, Defensa = rating × 0.7, Físico = rating × 0.9, Técnica = rating.
- DEF: Ataque = rating × 0.3, Defensa = rating, Físico = rating × 0.95, Técnica = rating × 0.8.
- ARQ: Ataque = 10, Defensa = rating, Físico = rating × 0.85, Técnica = rating × 0.85.

#### 5.2 `fixture.md`

```markdown
# Fixture Mundial 2026

> Último partido simulado: —
> Próximo partido pendiente: <YYYY-MM-DD — local vs visitante (HH:MM, sede)>

## Fase de Grupos

### Jornada 1

| Fecha | Hora | Sede | Grupo | Local | Visitante | Estado | Resultado | Archivo |
|-------|------|------|-------|-------|-----------|--------|-----------|---------|
| <YYYY-MM-DD> | <HH:MM> | <Estadio, Ciudad> | <A> | <Local> | <Visitante> | ⏳ Pendiente | — | — |
...

### Jornada 2
...

### Jornada 3
...
```

La línea "Próximo partido pendiente" del encabezado se llena con el primer partido de la lista ordenada.

#### 5.3 `grupos.md`

```markdown
# Tablas de posiciones — Fase de Grupos

> Actualizado tras: —

## Grupo A
| Pos | Equipo | PJ | G | E | P | GF | GC | DG | Pts |
|-----|--------|----|----|---|---|----|----|----|----|
| 1 | <eq1> | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| 2 | <eq2> | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| 3 | <eq3> | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| 4 | <eq4> | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |

## Grupo B
...
(idem para los 12 grupos)

## Clasificación a octavos
Pasan los **2 primeros de cada grupo** + **8 mejores terceros** (Mundial de 48, formato 2026).

Clasificados directos: —
Mejores terceros: —
```

#### 5.4 Resumen al usuario

Después de escribir todo, mostrar:

> ✅ Inicialización completa
> - Equipos: 48 archivos en `equipos/`
> - Partidos en fixture: 72
> - Grupos: 12
> - Primer partido: `<YYYY-MM-DD HH:MM>` `<local>` vs `<visitante>` en `<sede>`
> - Próximo paso: invocar la skill `simular-partido`
````

### Manejo de errores

| Error | Acción |
|---|---|
| WebSearch no devuelve resultados útiles para el fixture | Reintentar con consulta alternativa. Si tras 3 intentos no hay éxito, abortar y pedirle al usuario que provea la lista o un link. |
| El fixture descargado no suma 72 partidos / 48 equipos / 12 grupos | Reintentar con otra fuente. Si persiste, mostrarle al usuario lo que se parseó y pedir confirmación o corrección manual. |
| Plantilla incompleta para una selección | Completar lo posible, marcar campos faltantes con `TBD`, incluir aviso al inicio del MD del equipo. Continuar con el resto. |
| Rating FIFA no disponible al mes corriente | Usar el último ranking publicado disponible. Documentar la fecha del ranking usado en una nota al final del README. |
| Conflicto de nombres entre selecciones (ej. dos países con nombre similar) | Usar slug con país completo (`republica-checa.md`, `corea-del-sur.md`). |

### Validación final

Antes de declarar éxito, verificar:

- [ ] Existen 48 archivos en `equipos/`.
- [ ] `fixture.md` tiene 72 filas de partidos.
- [ ] `grupos.md` tiene 12 grupos × 4 equipos cada uno.
- [ ] La línea "Próximo partido pendiente" del encabezado de `fixture.md` apunta al primer partido por fecha+hora.
- [ ] Todas las selecciones del fixture tienen su MD correspondiente en `equipos/`.

Si alguna falla, reportarla al usuario y no marcar la inicialización como completa.
