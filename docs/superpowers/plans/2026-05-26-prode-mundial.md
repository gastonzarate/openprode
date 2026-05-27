# Prode Mundial 2026 — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Construir un sistema en Markdown + 2 skills locales para simular partido a partido la fase de grupos del Mundial 2026, generando relato minuto a minuto, probabilidades Monte Carlo y actualización del estado de los equipos.

**Architecture:** Repositorio plano de archivos Markdown como base de datos viva: `equipos/` (48 MDs), `fixture.md`, `grupos.md`, `partidos/`. Dos skills locales orquestadas por el LLM: `inicializar-mundial` (bootstrap one-shot, baja datos con WebSearch) y `simular-partido` (corre N veces, una por partido, baja clima/historial cara a cara, simula 10 corridas MC + 1 oficial relatada).

**Tech Stack:** Markdown puro, skills Claude Code locales (`.claude/skills/<nombre>/SKILL.md`), WebSearch + WebFetch como única dependencia externa, sin código ni runtime adicional.

**Spec:** `docs/superpowers/specs/2026-05-26-prode-mundial-design.md`

**Nota sobre TDD:** Este proyecto no tiene código ejecutable — los entregables son archivos Markdown que el LLM interpreta al invocar las skills. Por eso reemplazamos "tests unitarios" por **validaciones manuales** explícitas al final de cada fase: invocar la skill correspondiente y verificar el output contra criterios concretos.

---

## File Structure

```
prode/
├── README.md                       (Task 3)
├── fixture.md                      (generado por inicializar-mundial)
├── grupos.md                       (generado por inicializar-mundial)
├── equipos/                        (generado por inicializar-mundial; 48 archivos)
├── partidos/                       (generado por simular-partido; 72 archivos)
├── docs/
│   └── superpowers/
│       ├── specs/2026-05-26-prode-mundial-design.md   (ya existe)
│       └── plans/2026-05-26-prode-mundial.md          (este archivo)
└── .claude/
    └── skills/
        ├── inicializar-mundial/SKILL.md               (Tasks 5-10)
        └── simular-partido/SKILL.md                   (Tasks 11-20)
```

**Responsabilidades:**
- `README.md`: punto de entrada humano. Explica qué es el proyecto, cómo correr cada skill, cómo se infieren las stats.
- `inicializar-mundial/SKILL.md`: instrucciones one-shot para bajar fixture y plantillas, generar 48 MDs de equipo + `fixture.md` + `grupos.md`.
- `simular-partido/SKILL.md`: instrucciones para simular el próximo partido pendiente (o uno específico): enriquecer con internet, correr MC, escribir relato y actualizar todos los archivos afectados.

---

## Fase 1 — Scaffolding del repositorio

### Task 1: Inicializar repo git

**Files:**
- Create: `.gitignore`

- [ ] **Step 1: Inicializar git**

```bash
cd /Users/gastonzarate/Documents/Code/prode
git init
git branch -m main
```

- [ ] **Step 2: Crear `.gitignore` mínimo**

```
.DS_Store
*.swp
*.swo
```

- [ ] **Step 3: Commit inicial**

```bash
git add .gitignore docs/
git commit -m "chore: initial repo with spec and plan"
```

---

### Task 2: Crear estructura de directorios vacíos

**Files:**
- Create: `equipos/.gitkeep`
- Create: `partidos/.gitkeep`
- Create: `.claude/skills/inicializar-mundial/.gitkeep`
- Create: `.claude/skills/simular-partido/.gitkeep`

- [ ] **Step 1: Crear directorios**

```bash
cd /Users/gastonzarate/Documents/Code/prode
mkdir -p equipos partidos .claude/skills/inicializar-mundial .claude/skills/simular-partido
touch equipos/.gitkeep partidos/.gitkeep
touch .claude/skills/inicializar-mundial/.gitkeep
touch .claude/skills/simular-partido/.gitkeep
```

- [ ] **Step 2: Verificar estructura**

```bash
find . -type d -not -path './.git*' | sort
```

Expected output incluye: `./.claude/skills/inicializar-mundial`, `./.claude/skills/simular-partido`, `./equipos`, `./partidos`.

- [ ] **Step 3: Commit**

```bash
git add equipos/.gitkeep partidos/.gitkeep .claude/
git commit -m "chore: scaffold project directories"
```

---

### Task 3: Crear `README.md`

**Files:**
- Create: `README.md`

- [ ] **Step 1: Escribir `README.md` con contenido completo**

```markdown
# Prode Mundial 2026

Sistema para simular partido a partido la fase de grupos del Mundial 2026 (FIFA World Cup 2026, USA/Canadá/México, 48 selecciones) y generar predicciones para un prode.

## Cómo funciona

1. **Bootstrap**: invocar la skill `inicializar-mundial` una sola vez. La skill usa WebSearch para bajar el fixture oficial y las plantillas de las 48 selecciones, y genera:
   - `equipos/<pais>.md` × 48 — DT, plantilla, stats, historial.
   - `fixture.md` — los 72 partidos de fase de grupos, ordenados por fecha y hora, todos `⏳ Pendiente`.
   - `grupos.md` — las 12 tablas de posiciones en cero.

2. **Simular partidos**: invocar la skill `simular-partido` sin argumentos para simular el próximo partido pendiente, o con argumento `local vs visitante` para forzar uno específico. La skill:
   - Lee `fixture.md` y los dos MDs de equipo.
   - Busca en internet el clima de la sede, el árbitro designado y el historial cara a cara.
   - Corre 10 simulaciones Monte Carlo silenciosas para calcular probabilidades.
   - Corre 1 simulación oficial relatada minuto a minuto.
   - Escribe `partidos/YYYY-MM-DD-<local>-vs-<visitante>.md`.
   - Actualiza los MDs de los dos equipos, `fixture.md` y `grupos.md`.

3. **Iterar**: repetir paso 2 hasta completar los 72 partidos. El prode se completa con los resultados de la corrida oficial de cada partido.

## Cómo se infieren las stats numéricas

Cada equipo tiene 8 stats (0-100): Ataque, Mediocampo, Defensa, Arco, Físico, Moral, Cohesión, Experiencia. Se generan así al inicializar:

- **Base**: ranking FIFA invertido y normalizado (top 1 ≈ 95, top 50 ≈ 70).
- **Ataque/Defensa/Arco/Mediocampo**: promedio ponderado del rating individual de los jugadores titulares de esa zona del campo, donde el rating individual se infiere del nivel del club (top-5 liga europea: 85-95, primera europea sólida: 75-85, ligas medias: 65-75, otros: 50-65).
- **Físico**: media ponderada por edad (penalización suave a >32 años).
- **Moral**: 70 base + ajustes por forma reciente (últimos 5 partidos pre-Mundial).
- **Cohesión**: alta si el cuerpo técnico lleva ≥2 años, media si <2, baja si hubo cambio reciente.
- **Experiencia**: minutos de Mundial y Copa América/Euro/Asiática promediados en titulares.

Estas stats luego evolucionan partido a partido (carga, moral, lesiones, suspendidos).

## Estructura del repo

- `equipos/<pais>.md` — MD vivo por selección.
- `fixture.md` — calendario y resultados.
- `grupos.md` — tablas de posiciones.
- `partidos/YYYY-MM-DD-<local>-vs-<visitante>.md` — relato y stats de cada partido jugado.
- `docs/superpowers/specs/` — diseño.
- `docs/superpowers/plans/` — plan de implementación.
- `.claude/skills/` — las dos skills.

## Marco narrativo

Ver sección 8 del spec en `docs/superpowers/specs/2026-05-26-prode-mundial-design.md`.

## Idioma

Todo en **español argentino**: relatos, archivos, comentarios.
```

- [ ] **Step 2: Validar formato**

```bash
head -20 README.md
```

Expected: el archivo arranca con `# Prode Mundial 2026` y describe los 3 pasos.

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs: add project README with usage and stats inference logic"
```

---

## Fase 2 — Skill `inicializar-mundial`

### Task 4: Crear esqueleto de la skill con frontmatter

**Files:**
- Create: `.claude/skills/inicializar-mundial/SKILL.md`
- Delete: `.claude/skills/inicializar-mundial/.gitkeep`

- [ ] **Step 1: Eliminar `.gitkeep` y crear `SKILL.md` con frontmatter + overview**

```markdown
---
name: inicializar-mundial
description: Use when starting the Prode Mundial 2026 project from scratch. Downloads the official FIFA World Cup 2026 fixture and the squads of all 48 qualified national teams via WebSearch/WebFetch, then generates equipos/<pais>.md × 48, fixture.md and grupos.md. Runs ONCE per project lifetime.
---

# Inicializar Mundial 2026

Bootstrap one-shot del proyecto Prode Mundial 2026. Descarga datos oficiales del Mundial 2026 y genera todos los archivos base.

**Cuándo usar:** solo al arrancar el proyecto, antes de simular cualquier partido. Si ya existen `fixture.md`, `grupos.md` o archivos en `equipos/`, pedir confirmación antes de sobreescribir.

**Idioma:** todo el contenido generado va en **español argentino**.

## Flujo

(secciones siguientes — Tasks 5-9)
```

- [ ] **Step 2: Eliminar `.gitkeep`**

```bash
rm .claude/skills/inicializar-mundial/.gitkeep
```

- [ ] **Step 3: Commit**

```bash
git add .claude/skills/inicializar-mundial/
git commit -m "feat(skill): scaffold inicializar-mundial with frontmatter"
```

---

### Task 5: Documentar paso 1 — verificación de estado previo

**Files:**
- Modify: `.claude/skills/inicializar-mundial/SKILL.md`

- [ ] **Step 1: Reemplazar `(secciones siguientes — Tasks 5-9)` por la sección de verificación**

Agregar al final del archivo:

```markdown
### Paso 1 — Verificar estado previo

Antes de hacer nada:

1. Revisar si existen `fixture.md`, `grupos.md` o archivos dentro de `equipos/` en el working directory raíz del proyecto.
2. Si **alguno existe**, mostrarle al usuario qué hay y preguntar:
   > "Ya existen archivos del Mundial inicializado. ¿Querés sobreescribirlos? Esto borra el progreso actual."
   - Si responde "no" o equivalente: cortar la ejecución acá.
   - Si responde "sí" o equivalente: continuar y sobreescribir todo.
3. Si **nada existe**: continuar directamente.

No tocar `partidos/` en este paso — el bootstrap nunca borra partidos ya simulados, pero si el usuario regenera fixture y equipos el contenido de `partidos/` queda inconsistente. Avisar de eso en el prompt de confirmación.
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/inicializar-mundial/SKILL.md
git commit -m "feat(skill): inicializar-mundial step 1 (state check)"
```

---

### Task 6: Documentar paso 2 — descarga y parseo del fixture oficial

**Files:**
- Modify: `.claude/skills/inicializar-mundial/SKILL.md`

- [ ] **Step 1: Agregar la sección de descarga del fixture al final**

```markdown
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
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/inicializar-mundial/SKILL.md
git commit -m "feat(skill): inicializar-mundial step 2 (fixture download)"
```

---

### Task 7: Documentar paso 3 — descarga de las 48 plantillas

**Files:**
- Modify: `.claude/skills/inicializar-mundial/SKILL.md`

- [ ] **Step 1: Agregar la sección al final**

```markdown
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
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/inicializar-mundial/SKILL.md
git commit -m "feat(skill): inicializar-mundial step 3 (squad data download)"
```

---

### Task 8: Documentar paso 4 — inferencia de stats numéricas

**Files:**
- Modify: `.claude/skills/inicializar-mundial/SKILL.md`

- [ ] **Step 1: Agregar la sección al final**

```markdown
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
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/inicializar-mundial/SKILL.md
git commit -m "feat(skill): inicializar-mundial step 4 (stat inference rules)"
```

---

### Task 9: Documentar paso 5 — escritura de archivos generados

**Files:**
- Modify: `.claude/skills/inicializar-mundial/SKILL.md`

- [ ] **Step 1: Agregar la sección al final**

````markdown
### Paso 5 — Escribir los archivos

#### 5.1 `equipos/<pais-slug>.md` × 48

Slug del archivo: nombre del país en minúscula sin tildes y guiones por espacios (`argentina.md`, `corea-del-sur.md`, `cabo-verde.md`).

Plantilla exacta:

```markdown
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

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/inicializar-mundial/SKILL.md
git commit -m "feat(skill): inicializar-mundial step 5 (file generation templates)"
```

---

### Task 10: Documentar manejo de errores de `inicializar-mundial`

**Files:**
- Modify: `.claude/skills/inicializar-mundial/SKILL.md`

- [ ] **Step 1: Agregar la sección al final**

```markdown
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
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/inicializar-mundial/SKILL.md
git commit -m "feat(skill): inicializar-mundial error handling + final validation"
```

---

### Task 11: Validación manual de `inicializar-mundial`

**Files:**
- (ninguno — es validación, no edición)

- [ ] **Step 1: Verificar que la skill se ve y cumple criterios**

```bash
cat .claude/skills/inicializar-mundial/SKILL.md | head -30
wc -l .claude/skills/inicializar-mundial/SKILL.md
```

Expected: arranca con frontmatter `name: inicializar-mundial` + `description: Use when starting the Prode Mundial 2026...`. Tiene secciones "Paso 1" a "Paso 5", "Manejo de errores" y "Validación final".

- [ ] **Step 2: Smoke test invocando la skill (semi-automático)**

> **Nota:** este paso necesita al usuario para correrlo. La skill se invoca en una sesión interactiva. Pedirle al usuario:
>
> "Invocá la skill `inicializar-mundial` ahora desde Claude Code y verificá que:
> 1. Crea los 48 archivos en `equipos/`.
> 2. Crea `fixture.md` con 72 partidos.
> 3. Crea `grupos.md` con 12 grupos.
> 4. La línea 'Próximo partido pendiente' apunta al primer partido por fecha+hora.
> 5. Avisame si algún archivo viene con bloque ⚠️ de datos incompletos."

- [ ] **Step 3 (post-smoke-test): Commit los archivos generados como snapshot inicial**

```bash
git add equipos/ fixture.md grupos.md
git commit -m "data: initial bootstrap of fixture, groups, and 48 team files"
```

---

## Fase 3 — Skill `simular-partido`

### Task 12: Crear esqueleto de `simular-partido` con frontmatter

**Files:**
- Create: `.claude/skills/simular-partido/SKILL.md`
- Delete: `.claude/skills/simular-partido/.gitkeep`

- [ ] **Step 1: Crear `SKILL.md`**

```markdown
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

(secciones siguientes — Tasks 13-19)
```

- [ ] **Step 2: Eliminar `.gitkeep`**

```bash
rm .claude/skills/simular-partido/.gitkeep
```

- [ ] **Step 3: Commit**

```bash
git add .claude/skills/simular-partido/
git commit -m "feat(skill): scaffold simular-partido with frontmatter"
```

---

### Task 13: Documentar paso 1 — localizar el partido a simular

**Files:**
- Modify: `.claude/skills/simular-partido/SKILL.md`

- [ ] **Step 1: Reemplazar `(secciones siguientes — Tasks 13-19)` por la sección al final**

```markdown
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
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/simular-partido/SKILL.md
git commit -m "feat(skill): simular-partido step 1 (match selection)"
```

---

### Task 14: Documentar paso 2 — cargar contexto de los equipos

**Files:**
- Modify: `.claude/skills/simular-partido/SKILL.md`

- [ ] **Step 1: Agregar al final**

```markdown
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
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/simular-partido/SKILL.md
git commit -m "feat(skill): simular-partido step 2 (load team context)"
```

---

### Task 15: Documentar paso 3 — enriquecimiento con WebSearch

**Files:**
- Modify: `.claude/skills/simular-partido/SKILL.md`

- [ ] **Step 1: Agregar al final**

```markdown
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
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/simular-partido/SKILL.md
git commit -m "feat(skill): simular-partido step 3 (live WebSearch enrichment)"
```

---

### Task 16: Documentar paso 4 — establecer factores fijos del partido

**Files:**
- Modify: `.claude/skills/simular-partido/SKILL.md`

- [ ] **Step 1: Agregar al final**

```markdown
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
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/simular-partido/SKILL.md
git commit -m "feat(skill): simular-partido step 4 (fixed match factors)"
```

---

### Task 17: Documentar paso 5 — 10 corridas Monte Carlo silenciosas

**Files:**
- Modify: `.claude/skills/simular-partido/SKILL.md`

- [ ] **Step 1: Agregar al final**

````markdown
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
````

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/simular-partido/SKILL.md
git commit -m "feat(skill): simular-partido step 5 (silent Monte Carlo runs)"
```

---

### Task 18: Documentar paso 6 — corrida oficial relatada

**Files:**
- Modify: `.claude/skills/simular-partido/SKILL.md`

- [ ] **Step 1: Agregar al final**

````markdown
### Paso 6 — Corrida oficial relatada minuto a minuto

Aplicar **el mismo algoritmo** que el Paso 5 una vez más (no usar ninguna de las 10 corridas anteriores), generando esta vez una línea de relato por minuto en español argentino.

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
````

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/simular-partido/SKILL.md
git commit -m "feat(skill): simular-partido step 6 (narrated official run)"
```

---

### Task 19: Documentar paso 7 — composición del MD del partido

**Files:**
- Modify: `.claude/skills/simular-partido/SKILL.md`

- [ ] **Step 1: Agregar al final**

````markdown
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
````

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/simular-partido/SKILL.md
git commit -m "feat(skill): simular-partido step 7 (match MD composition)"
```

---

### Task 20: Documentar paso 8 — actualización de los archivos vivos

**Files:**
- Modify: `.claude/skills/simular-partido/SKILL.md`

- [ ] **Step 1: Agregar al final**

````markdown
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
````

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/simular-partido/SKILL.md
git commit -m "feat(skill): simular-partido step 8 (update equipos/fixture/grupos)"
```

---

### Task 21: Documentar paso 9 + manejo de errores + validación final

**Files:**
- Modify: `.claude/skills/simular-partido/SKILL.md`

- [ ] **Step 1: Agregar al final**

```markdown
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
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/simular-partido/SKILL.md
git commit -m "feat(skill): simular-partido step 9 + error handling + validation"
```

---

### Task 22: Validación manual de `simular-partido` (smoke test del primer partido)

**Files:**
- (ninguno — validación)

- [ ] **Step 1: Verificar la skill**

```bash
wc -l .claude/skills/simular-partido/SKILL.md
head -10 .claude/skills/simular-partido/SKILL.md
```

Expected: frontmatter `name: simular-partido`, archivo con 9 pasos + manejo de errores + validación.

- [ ] **Step 2: Smoke test del primer partido (semi-automático)**

Pedirle al usuario:

> "Invocá la skill `simular-partido` sin argumentos. Debería simular el primer partido del Mundial 2026. Verificá que:
> 1. Crea `partidos/<YYYY-MM-DD>-<local>-vs-<visitante>.md`.
> 2. El relato tiene una línea por cada minuto del 1 al 90 + adicionados.
> 3. La sección 2 muestra probabilidades MC con 10 corridas sumando 100%.
> 4. Los MDs de los dos equipos se actualizaron: revisá carga física, moral, historial Mundial.
> 5. `fixture.md` marcó ese partido como ✅ Jugado y la línea 'Próximo partido pendiente' apunta al siguiente.
> 6. `grupos.md` tiene la tabla del grupo correspondiente con +1 PJ en cada equipo.
> 7. Avisame si algún punto falla o si el relato no se siente fluido."

- [ ] **Step 3: Commit los archivos generados como snapshot**

```bash
git add partidos/ fixture.md grupos.md equipos/
git commit -m "data: first simulated match (smoke test)"
```

---

## Fase 4 — Validación integral y cierre

### Task 23: Simular 2-3 partidos más y validar coherencia entre partidos

**Files:**
- (ninguno — validación)

- [ ] **Step 1: Simular 3 partidos más (segundo, tercero y cuarto del fixture)**

Pedirle al usuario:

> "Invocá `simular-partido` 3 veces seguidas (sin argumento cada vez)."

- [ ] **Step 2: Verificar consistencia entre partidos**

Pedirle al usuario que verifique:

> "Después de simular los 4 primeros partidos, chequeá:
> 1. Cada uno tiene su archivo en `partidos/`.
> 2. Los equipos que jugaron más de una vez tienen su carga física acumulada subiendo coherentemente.
> 3. Si alguien quedó suspendido en un partido, no aparece en el XI del siguiente.
> 4. Las tablas de los grupos involucrados están correctamente sumando puntos y goles.
> 5. La línea 'Último partido simulado' del fixture refleja el cuarto partido jugado."

- [ ] **Step 3: Commit**

```bash
git add partidos/ fixture.md grupos.md equipos/
git commit -m "data: 4 simulated matches, cross-match coherence verified"
```

---

### Task 24: Documento final de cierre

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Agregar al `README.md` una sección final con estado actual**

```markdown
## Estado

- ✅ Skills implementadas: `inicializar-mundial`, `simular-partido`.
- ✅ Inicializado: <fecha de la corrida de inicialización>.
- 🏃 Simulación en curso. Partidos completados: <N>/72. Ver `fixture.md`.
- ⏭ Próximo: invocar `simular-partido` para el siguiente partido pendiente.
- 🚧 Eliminatorias fuera de alcance — spec separado a partir de fase de grupos completa.
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add project status section"
```

---

## Apéndice — Notas para quien ejecute este plan

- Este proyecto **no tiene código ejecutable**. Las skills son archivos Markdown que el LLM interpreta al invocarlas.
- "TDD" tradicional no aplica. Reemplazamos por validaciones manuales explícitas en Tasks 11, 22 y 23.
- Cada Task termina con un commit. La granularidad es a nivel de "sección del SKILL.md" para que el progreso sea trazable.
- Si una skill, una vez invocada, no funciona como espera el spec, **no modificar la skill ad-hoc en plena corrida** — abortar, anotar el problema, y crear una nueva Task de corrección en este plan antes de continuar.
- El relato minuto a minuto es la parte más subjetiva: revisar después del primer partido completo si el tono se siente cómodo (sección 8.6 del spec, "español argentino"). Si no, ajustar instrucciones de estilo en el Paso 6 del skill.
