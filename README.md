# Mapadrinamiento UTFSM TEL

Sistema de emparejamiento entre mechones y padrinos para maximizar afinidad global usando optimización combinatoria.

## Objetivo

Dado un conjunto de mechones $M$ y padrinos $P$, se busca una asignación uno-a-uno que maximice la afinidad total, evitando decisiones locales que perjudiquen el resultado global.

---

## Arquitectura (estándar `src/`)

- [main.py](main.py): punto de entrada y orquestación.
- [src/config.py](src/config.py): configuración por entorno.
- [src/catalogs.py](src/catalogs.py): categorías oficiales + aliases.
- [src/text_normalization.py](src/text_normalization.py): normalización de texto.
- [src/semantic_mapper.py](src/semantic_mapper.py): mapeo local determinístico + IA controlada.
- [src/data_pipeline.py](src/data_pipeline.py): lectura/preprocesamiento del Excel.
- [src/scoring.py](src/scoring.py): puntaje y guardrails.
- [src/matching.py](src/matching.py): algoritmo Húngaro.
- [tests/test_main.py](tests/test_main.py): pruebas unitarias.

---

## Flujo del programa

1. **Carga de datos** desde `data/*.xlsx`.
2. **Unificación de columnas** (mechón/padrino tienen headers levemente distintos).
3. **Saneamiento semántico** de respuestas abiertas:
   - primero reglas locales determinísticas,
   - luego IA _solo si aporta_ (según configuración).
4. **Construcción de matriz de afinidad** mechón × padrino.
5. **Optimización global** con algoritmo Húngaro.
6. **Exportación**:
   - `match.csv`
   - `reporte_ia.csv`

---

## Matemática del score

### 1) Similitud por categoría

Para cada categoría $c$ y para una pareja $(m,p)$:

$$
S_m^c = \text{set de respuestas normalizadas de } m
$$

$$
S_p^c = \text{set de respuestas normalizadas de } p
$$

Similitud de Jaccard:

$$
\text{sim}_c(m,p)=\frac{|S_m^c \cap S_p^c|}{|S_m^c \cup S_p^c|}
$$

Con pesos jerárquicos $w_c$:

$$
\text{raw}(m,p)=\sum_c w_c\,\text{sim}_c(m,p)
$$

### 2) Ajustes de robustez

- **Bonus por cobertura** (coincidir en varias categorías):

$$
\text{bonus}=0.15\cdot \#\{c:\text{sim}_c>0\}
$$

- **Penalización de cola** para scores muy bajos:

$$
\text{gap}=\max(0,\text{SCORE\_FLOOR}-\text{raw})
$$

$$
\text{penalidad}=\text{LOW\_SCORE\_PENALTY\_FACTOR}\cdot\frac{\text{gap}^2}{\text{SCORE\_FLOOR}}
$$

- **Multiplicador vital** por afinidad en `Pref`/`Hobby`:
  - 1.00 si ambas coinciden,
  - 0.92 si coincide solo una,
  - 0.80 si no coincide ninguna.

Score efectivo:

$$
\text{effective}=\max\left(0,\left(\text{raw}+\text{bonus}-\text{penalidad}\right)\cdot \text{vital\_multiplier}\right)
$$

### 3) Optimización global

Se resuelve:

$$
\max_{\pi}\sum_i \text{effective}(m_i,p_{\pi(i)})
$$

donde $\pi$ es una permutación uno-a-uno entre mechones y padrinos.

Como `linear_sum_assignment` minimiza costo, se usa:

$$
\text{cost}_{ij}=-\text{effective}(m_i,p_j)
$$

---

## Modo IA (controlado)

Parámetros en `.env`:

- `USE_AI=1` activa IA.
- `AI_ALLOWED_CATEGORIES=Pref` limita categorías donde IA puede intervenir.
- `AI_MAX_CALLS=12` presupuesto máximo de llamadas.
- `AI_MIN_CONFIDENCE_TO_SKIP=0.85` si local supera eso, no llama IA.

Si hay error HTTP `401/402/403/429`, el sistema desactiva IA en esa corrida y sigue local.

---

## Ejecutar

```bash
python main.py
```

## Ejecutar tests

```bash
python -m unittest discover -s tests -p 'test_*.py' -v
```

---

## Buenas prácticas implementadas

- Separación por responsabilidades (SRP).
- Configuración desacoplada por entorno.
- Fallback seguro: el pipeline no se cae por problemas externos de IA.
- Registro auditable de saneamiento.
- Tests unitarios para reglas críticas.
