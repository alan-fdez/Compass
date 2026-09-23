# Compass

[![CI](https://github.com/alan-fdez/Compass/actions/workflows/ci.yml/badge.svg)](https://github.com/alan-fdez/Compass/actions/workflows/ci.yml)

**Radar de licitaciones públicas españolas.** [PLACSP](https://contrataciondelestado.es) publica del orden de 800 anuncios cada día. Compass los ingiere, deja los pocos que encajan con tu empresa, y lee el pliego de esos pocos para decirte, con la cláusula y la página delante, si puedes presentarte.

No es un buscador de subvenciones: son contratos que la administración compra, no dinero que reparte.

![El listado de matches, con el embudo y el porqué de cada encaje](docs/images/dashboard-matches.png)

## Arrancar

Hace falta Docker. Nada más: ni Python, ni Node, ni una cuenta en ningún sitio.

```bash
cp backend/.env.example backend/.env
docker compose up -d
```

Eso levanta Postgres, Redis, la API, el worker, el planificador y el dashboard, y aplica las migraciones por el camino. El dashboard queda en **http://localhost:3000** y la API en **http://localhost:8000/docs**.

**Ninguna clave es obligatoria.** Con el `.env` recién copiado y vacío, todo funciona salvo el agente: la ingesta, el embudo, el ranking híbrido y el dashboard entero. Cuando pulses «analizar el pliego», la pantalla te dirá qué falta en vez de fallar. Para desbloquearlo, una clave gratuita de [openrouter.ai](https://openrouter.ai) en `OPENROUTER_API_KEY` (nivel gratuito, 50 peticiones al día, 0 €). Las de [Langfuse](https://langfuse.com) son opcionales y sólo sirven para ver el coste real de cada análisis.

La primera vez, el dashboard te pide el perfil de tu empresa: a qué te dedicas, tus CPV, tu rango de importe, tu facturación y tus certificaciones. Al guardarlo se descargan los últimos tres meses de PLACSP (unos minutos, con el avance en pantalla) y a partir de ahí la ingesta diaria corre sola a las 03:00.

## Cómo funciona

**1. Ingesta.** El feed ATOM/CODICE de PLACSP, leído de forma incremental con una marca de agua sobre `atom:updated`, filtrado al vertical de servicios informáticos (CPV 72) y persistido con *upsert* por expediente: una licitación republicada actualiza su fila, nunca crea otra ni se borra. En la instalación de desarrollo eso son **5.363 licitaciones**.

**2. El embudo.** Tres etapas que reducen ese corpus al puñado que encaja con un proveedor concreto, y **ninguna pasa por un modelo**: filtros duros en SQL (plazo, CPV, importe, ámbito), recuperación híbrida (léxica con `tsvector` y semántica con pgvector) y fusión de los dos rankings por *Reciprocal Rank Fusion*. Es determinista y se puede auditar línea a línea.

**3. El agente.** Bajo demanda, un grafo de LangGraph descarga el PCAP, comprueba que tiene capa de texto y se lo pasa entero al modelo contra un esquema Pydantic cerrado: solvencia económica y técnica, certificaciones, criterios de adjudicación, garantías, plazos, subcontratación y lotes. **Cada valor viaja con su cita**: cláusula, página y texto literal.

**4. El veredicto.** APTO / APTO CON RESERVAS / NO APTO, calculado **en Python** comparando esa extracción con tu perfil.

![La ficha de una licitación, con el veredicto citado](docs/images/dashboard-verdict.png)

## Las decisiones que importan

### El modelo extrae; el veredicto lo calcula el código

El LLM no opina nunca sobre si puedes presentarte. Rellena un esquema cerrado, y el veredicto sale de comparar esos datos con tu perfil en Python plano. Eso lo hace determinista (el mismo pliego y el mismo perfil dan siempre lo mismo), auditable (cada razón enseña la cláusula que la sostiene) y barato de recalcular: el veredicto no se guarda, se recalcula al leerlo, así que editar tu perfil cambia todos los veredictos al instante sin volver a pagar un solo análisis.

### Las citas se verifican en Python, no se creen

Que un modelo escriba una cita no significa que exista. Cada cita se busca en la página que dice, y la ficha enseña qué proporción de las citas de ese análisis ha pasado la comprobación. No es un modelo evaluando a otro: es una comparación contra el texto del propio pliego.

La comprobación distingue **tres resultados, no dos**, y la razón es el formato de los pliegos. Un PCAP mete todo lo que decide una licitación en el «Cuadro de Características», una tabla a dos columnas, y el extractor de PDF la lee por líneas visuales: la etiqueta de la izquierda acaba dentro de la frase de la derecha, o partida alrededor de una fila de casillas. Un modelo que lee esa tabla **bien** escribe una cita que no aparece entera y seguida en ninguna parte. Así que una cita está *verificada* si aparece literal, *verificada con el orden roto* si todas sus palabras están en esa página pero no seguidas, y *sin verificar* si a la página le falta alguna. Sólo la tercera significa que el modelo escribió algo que el pliego no dice. Antes las dos primeras se contaban igual, y el pliego de la captura de arriba puntuaba 56% con las nueve citas correctas.

### Híbrido, no sólo vectorial

Con el perfil sembrado, el embudo deja hoy **5.363 → 104 → 35 → 10**, y de esas 10 finales **6 las trajo únicamente el recuperador vectorial**: comparten significado con el perfil sin compartir sus palabras, así que una búsqueda por palabras clave las habría perdido enteras. Eso es exactamente por lo que hay dos recuperadores y no uno.

Un detalle de esos números que merece contarse: la primera etapa filtraba sólo por el código de estado que publica PLACSP, y hoy dejaría pasar 1.740. Pero PLACSP no mueve ese código de forma fiable al vencer el plazo: **1.636 de esas 1.740 (el 94%) tienen la fecha límite ya pasada**. Ahora la etapa exige las dos cosas, y por eso el número es tan pequeño: son las que de verdad se pueden presentar hoy.

### Qué cuesta de verdad analizar un pliego

Medido con [Langfuse](https://langfuse.com) sobre las trazas reales acumuladas (`uv run python -m compass.analysis.cost_report`), no estimado:

| Medida | Valor real |
|---|---|
| Análisis con extracción completada | 36 |
| Tokens por análisis | 22.300 – 118.569 (media **59.341**) |
| Coste | **0,00 €** en el 100% |
| Tiempo de extremo a extremo | 20 – 338 s (media 155 s) |

El modelo es de razonamiento y el pliego entra completo, sin trocear: de ahí que el tiempo y el volumen de entrada varíen tanto de un pliego a otro.

### Cómo se sabe que la extracción es correcta

Tres medidas, y dos de las tres no necesitan ningún modelo:

- **25 pliegos anotados a mano**, campo a campo, y un *gate* de regresión que corre el grafo de producción contra ellos y compara los nueve campos objetivamente comprobables (`analysis/regression_eval.py`).
- **Fidelidad de citas**, comprobada en Python sobre el texto del PDF (`analysis/verification.py`).
- **Fidelidad de las descripciones en texto libre**, que es la mitad que no se puede comparar cadena a cadena: ahí sí entra un juez LLM, con RAGAS, contra las páginas que cada cita señala (`analysis/freetext_eval.py`). Se ejecuta a mano: cuesta dos llamadas al modelo por descripción.

## Stack

Python 3.13 con `mypy --strict` · FastAPI async sobre Uvicorn · PostgreSQL 17 + pgvector con SQLAlchemy async y Alembic · Celery sobre Redis para la ingesta diaria, los embeddings y los análisis · `ibm-granite/granite-embedding-278m-multilingual` corriendo en local para los embeddings · LangGraph para el agente, con `nvidia/nemotron-3-super-120b-a12b:free` vía OpenRouter · Langfuse para trazas y coste · Next.js 16, React 19 y Tailwind v4 para el dashboard · pytest, ruff, GitHub Actions.

## Qué queda fuera, a propósito

- **Subvenciones y ayudas** (BDNS, TED). Multiplican el modelo de datos y difuminan el producto: esto es un radar de contratos.
- **OCR.** Un pliego escaneado sin capa de texto se marca como no analizable y se dice en pantalla, en vez de adivinar.
- **Digest diario por correo.** Compass corre en tu máquina; un correo a las 03:00 sale de un proceso que a esa hora está apagado, y llegaría justo cuando vas a abrir el dashboard de todas formas.
- **Multi-inquilino y despliegue alojado.** Un perfil, una máquina, tus datos.

### Limitaciones conocidas

**Qué puede rechazarte, y quién decide que puede.** Un pliego nombra certificaciones en tres papeles (exigidas para licitar, puntuadas como criterio de adjudicación, o papeleo que presenta cualquier licitador) y sólo el primero excluye. El esquema no tenía dónde decirlo, así que el veredicto trataba las tres como requisitos y salían **NO APTO falsos**: cuatro de los seis análisis de la base de desarrollo los tenían. Ahora cada certificación viaja con su papel y su cita.

Pero el papel lo rellena el modelo, y **medido sobre tres pliegos reales, no lo rellena bien**: devolvió «exigida para licitar» en todos los casos, incluidos cuatro perfiles de equipo («Responsable técnico del proyecto») y una declaración responsable. Así que el código no se fía: una certificación sólo bloquea si además es un esquema reconocible: ISO, UNE-EN, ENS, CMMI, CCN-CERT o ENAC. Lo que no lo es aparece como **reserva**, con su cláusula citada, para que lo compruebes tú. Un APTO CON RESERVAS de más cuesta leer un pliego; un NO APTO de más cuesta un contrato que nunca llegaste a ver.

Es el mismo principio que el resto del proyecto: el modelo extrae, el código decide. Aquí decide incluso sobre lo que el modelo afirma de sí mismo.

## Desarrollo

Con [uv](https://docs.astral.sh/uv/), levantando sólo la infraestructura:

```bash
docker compose up -d db redis
cd backend
uv run alembic upgrade head
uv run python -m compass --reload     # API en http://localhost:8000/docs

# en otras dos terminales
uv run celery -A compass.core.celery_app worker --pool=solo --loglevel=info
uv run celery -A compass.core.celery_app beat --loglevel=info
```

`uv run pytest` corre los **296 tests** contra Postgres y Redis reales; `uv run ruff check`, `uv run ruff format --check` y `uv run mypy` cierran el resto. Todo eso pasa también en [CI](.github/workflows/ci.yml) en cada push, contra los mismos contenedores de Postgres con pgvector y Redis.

Cinco tests quedan fuera de CI (`@pytest.mark.real_corpus`): comparan el golden set y el embudo contra el corpus real de PLACSP persistido en local, y contra uno sintético no probarían nada. Los evals que llaman al modelo quedan fuera por lo mismo, y porque una corrida se come la cuota diaria del nivel gratuito.

## El razonamiento completo

Este README cuenta qué es y cómo funciona. **El porqué de cada decisión (con lo que se midió, lo que se descartó y lo que salió mal por el camino) vive en [`docs/phases/`](docs/phases/)**: un documento por subfase, escrito mientras se construía, con la evidencia delante. Ahí está por qué el veredicto no lo emite el modelo, por qué el embudo es híbrido, por qué el golden set tiene 25 pliegos y no 4, y qué encontró cada revisión.

## Licencia

[MIT](LICENSE). Descárgalo, ejecútalo, cámbialo o reutiliza lo que te sirva; lo único que pide es que el aviso de copyright viaje con el código. Sin garantía de ningún tipo: esto lee pliegos y te da una opinión calculada, no asesoramiento jurídico. La decisión de presentarte a una licitación sigue siendo tuya.
