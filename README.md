# Mapadrinamiento UTFSM TEL

Script para emparejar mechones con mapadrinos maximizando afinidad global.

## Quick Start

### 1) Clonar repositorio

```bash
git clone https://github.com/michifeli/mapadrinamiento.git
cd mapadrinamiento
```

### 2) Crear entorno e instalar dependencias

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### 3) Preparar datos

- Deja el Excel de respuestas dentro de la carpeta `data/`.
- El script lee el primer archivo que coincida con `data/*.xlsx`.

### 4) Configurar `.env` (opcional, para apoyo IA)

Si quieres usar IA en casos ambiguos, crea un archivo `.env` en la raíz con:

```env
USE_AI=1
API_KEY=tu_token
HF_MODEL=meta-llama/Llama-3.1-8B-Instruct
AI_ALLOWED_CATEGORIES=Pref
AI_MAX_CALLS=12
AI_MIN_CONFIDENCE_TO_SKIP=0.85
```

Si no configuras IA, el sistema funciona 100% local (determinístico).

### 5) Ejecutar matching

```bash
python main.py
```

Se generan:

- `match.csv`: resultado final de emparejamientos.
- `reporte_ia.csv`: trazabilidad del saneamiento de respuestas.

### 6) Ejecutar tests

```bash
python -m unittest discover -s tests -p 'test_*.py' -v
```

## Estructura del proyecto

- [main.py](main.py): orquesta todo el flujo.
- [src/config.py](src/config.py): lee configuración desde entorno.
- [src/catalogs.py](src/catalogs.py): opciones oficiales y aliases.
- [src/text_normalization.py](src/text_normalization.py): limpieza de texto.
- [src/semantic_mapper.py](src/semantic_mapper.py): mapeo local + IA opcional.
- [src/data_pipeline.py](src/data_pipeline.py): lectura y normalización del Excel.
- [src/scoring.py](src/scoring.py): cálculo de puntajes.
- [src/matching.py](src/matching.py): asignación uno-a-uno (Hungarian).
- [tests/test_main.py](tests/test_main.py): pruebas básicas.

## Matemática (explicada simple)

### Problema

Queremos asignar cada mechón a un mapadrino maximizando el puntaje total de afinidad.

### 1) Similitud por categoría

Para una categoría $c$ y una pareja $(m,p)$:

- $A_c(m)$: respuestas normalizadas de $m$ en la categoría $c$.
- $A_c(p)$: respuestas normalizadas de $p$ en la categoría $c$.

Se usa Jaccard:

$$
sim_c(m,p)=\frac{|A_c(m)\cap A_c(p)|}{|A_c(m)\cup A_c(p)|}
$$

Luego se pondera por importancia de categoría ($w_c$):

$$
raw(m,p)=\sum_c w_c\cdot sim_c(m,p)
$$

### 2) Ajustes del score

Se agregan 3 ajustes para que el score sea más realista:

1. **Bonus de cobertura** (si coinciden en más categorías):

$$
bonus=0.15\cdot N_{match}
$$

donde $N_{match}$ es la cantidad de categorías con similitud mayor a 0.

2. **Penalización de cola** (evita aceptar pares demasiado débiles):

$$
gap=\max(0, FLOOR-raw)
$$

$$
penalidad=K\cdot\frac{gap^2}{FLOOR}
$$

En el código: $FLOOR = 8.0$ y $K = 0.55$.

3. **Multiplicador vital** según afinidad en `Pref` y `Hobby`:

- $1.00$ si coinciden ambas,
- $0.92$ si coincide solo una,
- $0.80$ si no coincide ninguna.

Score final:

$$
effective(m,p)=\max\left(0,(raw+bonus-penalidad)\cdot mult\right)
$$

### 3) Optimización global

No se elige el mejor padrino de cada mechón por separado.
Se optimiza el conjunto completo:

$$
\max_{\pi}\sum_i effective(m_i,p_{\pi(i)})
$$

donde $\pi$ es una asignación uno-a-uno.

Como el algoritmo húngaro resuelve minimización, se usa:

$$
cost_{ij}=-effective(m_i,p_j)
$$
