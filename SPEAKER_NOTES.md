# Notas del Presentador — dbt Theory Session

> Úsalas como referencia durante la explicación. No necesitas leerlas literalmente — son el "fondo de armario" que te da seguridad para improvisar.

---

## S1 — Portada

Nada especial. Mientras carga, di:

> "Esta mañana hacemos toda la teoría de golpe. Sé que prefieren tocar código, y vamos a hacerlo — pero invertir 2-3 horas ahora va a hacer que el resto de la semana tenga sentido. Todo lo que vean en las diapositivas lo van a tocar con sus manos hoy mismo."

---

## S2 — Agenda

Señala que los 10 temas están ordenados por dependencia lógica — no es arbitrario. No puedes entender macros sin entender modelos, y no puedes entender CI/CD sin entender la diferencia entre Core y Cloud.

Pregunta rápida para romper el hielo: *"¿Alguien ha usado dbt antes, aunque sea un poco?"* Calibra el nivel.

---

## S3 — Cómo Llegamos Aquí

**Lo que NO está en la diapositiva y debes explicar:**

**Por qué ETL murió con los cloud DWH:**
En los 90s y 2000s, el almacenamiento en un data warehouse on-premise era carísimo — cientos de dólares por GB. Por eso tenías que transformar los datos ANTES de cargarlos: solo podías permitirte guardar lo que ya estaba limpio y estructurado. Transformar requería servidores ETL separados (Informatica, SSIS), equipos especializados, y meses de desarrollo para cada pipeline.

Cuando llegó Snowflake (2012), BigQuery (2010) y Redshift (2012), el almacenamiento pasó a costar céntimos por GB y el cómputo era elástico — lo enciendes cuando lo necesitas y lo pagas por segundo. De repente, **tiene sentido cargar los datos en bruto primero y transformarlos dentro del warehouse**, donde ya tienes toda la potencia de SQL y un motor columnar optimizado para esto. Eso es ELT.

**Separación de storage y compute:** Snowflake fue el primero en separar completamente el almacenamiento del cómputo. Puedes tener 10 TB de datos y encender un warehouse de 5 minutos para transformarlos, pagando solo esos 5 minutos. Eso cambia completamente el análisis de coste de transformar dentro del warehouse.

**dbt nació en 2016 en Fishtown Analytics** (ahora dbt Labs), fundada por Tristan Handy. Era una consultora de datos en Filadelfia. Estaban harto de escribir los mismos scripts SQL desorganizados para cada cliente. dbt empezó como una herramienta interna. La publicaron como open source y creció orgánicamente — hoy más de 50,000 empresas lo usan en producción (Spotify, JetBlue, Samsara, GitLab…).

---

## S4 — El Analytics Engineer

**Lo que debes explicar con más profundidad:**

El título de "analytics engineer" lo acuñó formalmente dbt Labs en 2016-2017. Antes existía el rol pero nadie sabía cómo llamarlo. Eran los data analysts que sabían suficiente de ingeniería para escribir pipelines, o los data engineers que entendían suficientemente el negocio para diseñar modelos útiles.

**La brecha que existía:**
- El **Data Engineer** construye el pipeline hasta el warehouse. Deposita datos en bruto y su trabajo termina ahí. No sabe qué significa `o_orderstatus = 'P'` desde el punto de vista de negocio.
- El **Analista / BI developer** consume datos del warehouse para hacer dashboards. No tiene tiempo ni skills para mantener pipelines. Si los datos están sucios, escribe SQL defensivo para limpiarlos en cada consulta — duplicando lógica.
- El **Analytics Engineer** vive en el medio. Su output es un conjunto de modelos limpios, testeados y documentados que cualquiera puede consumir. Habla SQL pero también habla negocio.

**Por qué importa que usen dbt específicamente:**
Con dbt, el analytics engineer puede trabajar con las mismas herramientas que un software engineer: Git, PRs, CI/CD, tests automatizados. La diferencia es que el lenguaje es SQL en lugar de Python/Java, y el "runtime" es Snowflake en lugar de un servidor de aplicaciones.

---

## S5 — El Problema Antes de Git

**Historias que puedes contar:**

La "final file problem" no es solo molesta — tiene consecuencias reales en producción:
- Un dashboard ejecutivo muestra la métrica de revenue incorrecta porque alguien corrió `analysis_v2_FINAL_revised.sql` que tiene un bug introducido hace 3 meses.
- Un analista pasa un fin de semana rehaciendo trabajo porque un colega sobreescribió su archivo compartido en Google Drive.
- Nadie puede hacer un deploy de un cambio porque no saben qué scripts se están ejecutando actualmente en producción.

**El problema de colaboración es peor en datos que en software:**
En software, la lógica está en el código. Si dos personas editan el mismo función, Git lo detecta y pide resolver el conflicto explícitamente. En datos, la "lógica" puede vivir en 12 queries distintas que se copian y pegan, y si cambias la definición de "revenue activo" en una, las otras 11 no se enteran.

**Haz esta pregunta a la audiencia:** *"¿Quién ha tenido la experiencia de que dos reportes del mismo negocio muestren números distintos para la misma métrica?"* Casi todos levantan la mano.

---

## S6 — Git: Cómo Funciona

**Lo que la diapositiva no explica suficientemente:**

**Git vs GitHub — son cosas distintas:**
- **Git** es el software de control de versiones. Vive en tu máquina. Es local.
- **GitHub** (o GitLab, Bitbucket) es un servidor remoto que aloja repositorios Git. Es el "Google Drive" de tu código, pero con superpoderes.
- Cuando haces `git push`, estás enviando tus commits locales a GitHub.

**Pull Requests (PRs) — el mecanismo de revisión de código:**
Cuando terminas de trabajar en una feature branch, abres un Pull Request en GitHub. Eso le dice al equipo: "quiero fusionar estos cambios a `main`". El equipo puede revisar el diff línea por línea, dejar comentarios, aprobar o pedir cambios. Solo cuando tiene aprobación se hace el merge. En datos esto es CRÍTICO: nadie debería poder cambiar la definición de `net_revenue` sin que otra persona lo revise.

**Por qué "distributed" importa:**
A diferencia de sistemas centralizados como SVN, cada desarrollador tiene una copia completa del historial. Si el servidor de GitHub cae, puedes seguir trabajando localmente y sincronizar cuando vuelva. También significa que el `git log` que ves en tu máquina es el mismo que el de cualquier otro miembro del equipo.

**Los 3 estados — analogía del correo:**
1. Working directory = carta que estás escribiendo en tu escritorio
2. Staging area (`git add`) = carta que metiste en el sobre pero no has sellado
3. Repository (`git commit`) = carta que llevaste al buzón — ya no puedes editarla, pero sí puedes escribir una nueva

---

## S7 — Vida Antes de dbt

**Añade contexto concreto a cada pain point:**

**"SQL script chaos":** En empresas reales, los pipelines de producción son archivos SQL sin nombre coherente, sin comentarios, ejecutados por cronjobs anónimos en un servidor que nadie recuerda haber configurado. Cuando ese servidor muere, nadie sabe qué se estaba ejecutando ni en qué orden. Este es el estado de "madurez 0" del 80% de los departamentos de datos del mundo.

**"No tests, no trust":** Un dashboard de ventas empieza a mostrar 0 para ciertas regiones. ¿Cuándo empezó a fallar? Nadie lo sabe porque no había alertas. ¿Qué parte del pipeline rompió? Nadie lo sabe porque no hay tests. Resultado: el equipo de dirección deja de confiar en los datos, vuelve a Excel, y el departamento de datos pierde credibilidad.

**"Copy-paste logic":** La definición de "cliente activo" es `last_order_date > NOW() - INTERVAL 90 DAY`. Esa lógica vive en 8 queries distintas. Llega el día que marketing decide que "activo" son 60 días, no 90. Tienes que encontrar y cambiar las 8 queries. Inevitablemente te olvidas de una. Ahora dos dashboards dan números distintos y nadie sabe cuál es correcto.

**"Knowledge silos":** La persona que construyó el pipeline sabe que para calcular revenue hay que excluir las órdenes con `status = 'CANCELLED'` y que además hay que dividir por 100 porque los precios están en centavos. Esa persona dejó la empresa hace 6 meses. El conocimiento se fue con ella.

---

## S8 — ¿Qué es dbt?

**Aclaraciones importantes que la diapositiva no dice:**

**dbt NO es:**
- Un cargador de datos. No extrae datos de Salesforce ni los carga en Snowflake. Para eso están Fivetran, Airbyte, etc.
- Una herramienta de BI. No hace dashboards ni visualizaciones. Para eso está Looker, Tableau, etc.
- Un orquestador. No programa trabajos ni gestiona dependencias a nivel de pipeline empresarial (eso lo hace Airflow, Prefect, o el scheduler de dbt Cloud).

**dbt SOLO hace la T en ELT:** transforma datos que ya están en tu warehouse usando SQL SELECT statements. Eso es todo. Y ese enfoque estrecho es su fortaleza — hace una cosa y la hace muy bien.

**¿Qué hace dbt internamente cuando ejecutas `dbt run`?**
1. Lee todos los archivos `.sql` en `models/`
2. Compila el Jinja (resuelve `{{ ref() }}`, `{{ source() }}`, macros)
3. Envuelve cada SELECT en `CREATE OR REPLACE TABLE/VIEW AS (...)`
4. Envía el DDL compilado a Snowflake (o el warehouse configurado)
5. Snowflake ejecuta, devuelve resultado
6. dbt reporta éxito/fallo, tiempo de ejecución, filas procesadas

Nunca mueve datos fuera del warehouse. Todo ocurre dentro de Snowflake.

**"Modular" en la práctica:**
Un modelo = un archivo = una tabla o vista. En lugar de un script de 2000 líneas que hace todo, tienes 20 modelos de 50-100 líneas cada uno, cada uno con un propósito claro. Puedes re-ejecutar solo el que cambió. Puedes testear solo el que te interesa. Puedes documentar cada uno de forma independiente.

---

## S9 — ETL vs ELT

**La explicación del "por qué" que no está en la diapositiva:**

**Por qué ETL era obligatorio antes:**
En 1995, un terabyte de almacenamiento en un data warehouse on-premise costaba entre 100,000 y 1,000,000 de dólares. El cómputo también era on-premise y había que comprarlo por adelantado. Con esas restricciones, cargar datos en bruto era un lujo imposible — tenías que transformar antes para no desperdiciar espacio.

**Por qué ELT es mejor ahora:**
1. **Almacenamiento barato:** Snowflake Storage cuesta ~$23/TB/mes. Puedes permitirte guardar los datos en bruto indefinidamente.
2. **Cómputo elástico:** Si necesitas más poder de procesamiento, aumentas el tamaño del virtual warehouse de Snowflake y lo pagas por segundos.
3. **Iteración rápida:** Si un analista quiere probar una nueva transformación, puede hacer `dbt run --select mi_nuevo_modelo` en segundos — sin tener que tocar la infraestructura de ETL.
4. **Datos en bruto preservados:** Si descubres que una transformación fue incorrecta, tienes los datos originales para rehacerla desde cero. Con ETL, si la transformación fue mala antes de cargar, los datos originales se perdieron.

**El punto clave para los estudiantes:**
dbt asume que los datos ya están en el warehouse (alguien los cargó con Fivetran o similar). A partir de ese punto, dbt toma el control.

---

## S10 — Modern Data Stack

**Explica el flujo completo de dato-a-dashboard:**

```
Salesforce CRM  →  [Fivetran extrae cada hora]  →  SNOWFLAKE_RAW.SALESFORCE.OPPORTUNITIES
                                                           ↓
                                               [dbt transforma]
                                                           ↓
                                       ANALYTICS.BRONZE.BRONZE_SALESFORCE_OPPORTUNITIES
                                       ANALYTICS.SILVER.SILVER_PIPELINE_ENRICHED
                                       ANALYTICS.GOLD.GOLD_PIPELINE_SUMMARY
                                                           ↓
                                               [Tableau conecta a GOLD]
                                                           ↓
                                         Dashboard de Pipeline de Ventas
```

**dbt solo ve la parte en negrita del flujo.** No sabe cómo llegaron los datos de Salesforce a Snowflake. No sabe qué hace Tableau con el output. Solo transforma lo que encuentra en el warehouse.

**Por qué "Modern" Data Stack:**
La diferencia con stacks anteriores es que cada capa es reemplazable. Si cambias de Fivetran a Airbyte, dbt no se entera. Si cambias de Tableau a Looker, dbt no se entera. Cada herramienta hace una cosa bien y tiene una interfaz clara con las demás. Esto es opuesto al mundo del "todo-en-uno" de herramientas como Informatica que intentaban controlar toda la cadena.

---

## S11 — Estructura del Proyecto dbt

**Explica cada archivo con más detalle:**

**`dbt_project.yml`** — El archivo más importante. Define:
- `name`: el nombre del proyecto (en este curso: `my_new_project`, que es el default que genera dbt Cloud)
- `model-paths`: dónde buscar archivos `.sql` (por defecto `models/`)
- `models:` — configuración por directorio: materialización por defecto, schema de destino, tags

Ejemplo de lo que verán en el curso:
```yaml
models:
  my_new_project:
    bronze:
      +materialized: view
      +schema: bronze
    silver:
      +materialized: view
      +schema: silver
    gold:
      +materialized: table
      +schema: gold
```

**`profiles.yml`** — Este archivo **nunca va al repositorio Git**. Contiene las credenciales de conexión a Snowflake (usuario, contraseña, cuenta). Vive en `~/.dbt/` en tu máquina local. En dbt Cloud, esto lo gestiona la interfaz gráfica — no tienes que tocarlo.

**`packages.yml`** — Como `requirements.txt` de Python o `package.json` de Node. Listas las dependencias externas. Después de modificarlo, ejecutas `dbt deps` para instalarlas.

**La carpeta `analyses/`** — Para SQL ad hoc que quieres versionar pero que no es un modelo del pipeline (ej: queries exploratorias, análisis puntuales). No se materializan en el warehouse.

**¿Por qué `models/bronze/`, `models/silver/`, `models/gold/`?**
Es la arquitectura Medallion. No es obligatorio en dbt — podrías llamar las carpetas como quieras — pero es la convención más extendida en la industria y la que usamos en este curso.

---

## S12 — Modelos

**Lo que la diapositiva no explica:**

**El ciclo de vida de un modelo cuando ejecutas `dbt run`:**
1. dbt lee `models/bronze/bronze_tpch_orders.sql`
2. Compila el Jinja: `{{ source('tpch', 'orders') }}` → `SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS`
3. Envuelve: `CREATE OR REPLACE VIEW ANALYTICS.BRONZE.BRONZE_TPCH_ORDERS AS (SELECT ...)`
4. Envía a Snowflake
5. Snowflake ejecuta y responde OK
6. dbt muestra: `1 of 1 OK bronze_tpch_orders [CREATE VIEW in 0.45s]`

**El bloque `{{ config() }}`:**
Si un modelo empieza con `{{ config(materialized='table') }}`, ese setting tiene prioridad sobre lo que diga `dbt_project.yml`. Útil para excepciones: "todos los modelos de bronze son views, excepto este que es incremental".

```sql
{{ config(
    materialized = 'incremental',
    unique_key   = 'order_id',
    schema       = 'gold'
) }}

SELECT ...
```

**Un modelo = una SELECT.** dbt no soporta múltiples statements en un archivo de modelo. No puedes hacer `INSERT INTO` ni `UPDATE` — solo `SELECT`. Eso es intencional: mantiene los modelos declarativos y predecibles.

**CTEs en modelos:**
Los modelos complejos usan CTEs (WITH clauses) como bloques de construcción:
```sql
WITH orders AS (
    SELECT * FROM {{ ref('bronze_tpch_orders') }}
),
customers AS (
    SELECT * FROM {{ ref('bronze_tpch_customers') }}
)
SELECT o.*, c.customer_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
```

---

## S13 — Sources

**La diferencia clave con ref():**

`{{ source('tpch', 'orders') }}` → apunta a datos que NO son de dbt (llegaron de Fivetran, de un loader, son datos en bruto).

`{{ ref('bronze_tpch_orders') }}` → apunta a un modelo dbt (fue creado por dbt, existe en el linaje).

**¿Por qué no simplemente hardcodear el path?**
```sql
-- ❌ Mal
FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS

-- ✅ Bien
FROM {{ source('tpch', 'orders') }}
```

Razones:
1. **Linaje:** la versión con `{{ source() }}` aparece en el DAG de dbt. La hardcodeada no.
2. **Flexibilidad:** si el schema cambia de `TPCH_SF1` a `TPCH_SF10`, cambias solo el `sources.yml` y todos los modelos que lo usan se actualizan automáticamente.
3. **Freshness checks:** `dbt source freshness` puede alertar si los datos de la fuente no han llegado en las últimas N horas. Solo funciona con sources declarados.

**`freshness: null`** — para fuentes de solo lectura (como TPC-H que es estática) desactivas el check de freshness explícitamente. Si no lo pones, dbt intentará comprobar frescura y fallará.

---

## S14 — ref() y el DAG

**Explica qué es un DAG:**

DAG = Directed Acyclic Graph (Grafo Dirigido Acíclico).
- **Directed** = las dependencias tienen dirección (A depende de B, no al revés)
- **Acyclic** = no hay ciclos (A no puede depender de sí mismo ni de forma indirecta)
- **Graph** = red de nodos (modelos) y aristas (dependencias)

**Por qué "acíclico" importa:**
Si A depende de B y B depende de A, ¿cuál ejecutas primero? Es imposible. dbt detecta ciclos en tiempo de compilación y los rechaza antes de ejecutar nada. Esto es una diferencia fundamental con SQL puro: en SQL, un ciclo solo falla en tiempo de ejecución, cuando ya has gastado cómputo.

**Cómo dbt construye el orden de ejecución:**
dbt lee todos los modelos, parsea todos los `{{ ref() }}` y `{{ source() }}`, construye el grafo, y hace un topological sort — garantiza que cada modelo se ejecuta después de todos sus padres. No necesitas especificar ningún orden.

**Qué pasa cuando un padre falla:**
Si `bronze_tpch_orders` falla (error de SQL, por ejemplo), todos los modelos que dependen de él (`silver_orders_enriched`, `gold_orders`, `gold_customers`) se marcan como `SKIPPED` — no se ejecutan. Esto es intencional: no tiene sentido construir un modelo silver sobre bronze corrupto.

**`{{ ref() }}` también resuelve el schema correcto:**
Cuando dbt compila `{{ ref('bronze_tpch_orders') }}`, no solo pone el nombre de la tabla — pone el schema completo correcto para el entorno actual: en desarrollo `ANALYTICS.BRONZE.BRONZE_TPCH_ORDERS`, en producción también `ANALYTICS.BRONZE.BRONZE_TPCH_ORDERS` (o lo que esté configurado). No necesitas escribir el schema hardcodeado en las referencias entre modelos.

---

## S15 — Materializaciones

**La diapositiva marca "view" como default, pero en dbt Cloud el default es `table`.** Aclararlo explícitamente antes de que lo descubran.

**Cuándo usar cada una:**

| Materialización | Úsala cuando... | No la uses cuando... |
|---|---|---|
| `view` | La query es rápida, los datos no cambian mucho, o es bronze sobre una fuente externa ya en Snowflake | El consumidor hace muchas queries en paralelo (regenera el query cada vez) |
| `table` | La query tarda >30s, es consultada frecuentemente por BI tools | Los datos cambian cada hora y reconstruir desde cero es costoso |
| `incremental` | La tabla crece continuamente (fact tables), procesar todo de nuevo sería demasiado lento | Necesitas datos históricos corregidos (late-arriving data problem) |
| `ephemeral` | Es solo un CTE intermedio que se usa en UN solo modelo | Múltiples modelos necesitan ese resultado (se re-ejecuta en cada uno) |

**Ejemplo concreto de cuándo NO usar view para bronze:**
Si TPC-H `LINEITEM` tiene 6 millones de filas y tu modelo bronze es una view, CADA vez que alguien consulte `silver_orders_enriched` (que referencia esa view), Snowflake lee los 6 millones de filas desde cero. Si 50 analistas hacen queries en paralelo, son 50 × 6M = 300M filas leídas. Con una tabla, se leen una vez y se sirven las 50 queries desde caché/tabla.

---

## S16 — Incremental Models

**Explica los conceptos que la diapositiva asume:**

**`{{ this }}`** = la relación actual del modelo (la tabla que YA existe en Snowflake de la última ejecución). La usas para saber el estado actual y calcular qué filas son "nuevas".

**`is_incremental()`** = una función Jinja que dbt inyecta. Devuelve `False` en la primera ejecución (la tabla no existe aún) y `True` en las siguientes. Por eso el bloque `{% if is_incremental() %}` solo aplica el filtro de fecha cuando ya existe data.

**`unique_key`:**
- Sin `unique_key`: dbt solo hace INSERT de las filas nuevas. Si una orden ya existente cambia, tendrás duplicados.
- Con `unique_key`: dbt hace MERGE (DELETE + INSERT o UPDATE en algunas plataformas). Las filas existentes con el mismo key se sobreescriben.

**El problema de los late-arriving data:**
El filtro `WHERE order_date > MAX(order_date)` es simple pero frágil. Si llegan órdenes del día de ayer con retraso (late data), el filtro las excluiría porque `MAX(order_date)` ya pasó. En producción real, se usa un lookback window: `WHERE order_date > MAX(order_date) - INTERVAL 3 DAY`. Esto lo verán cuando hagan el ejercicio.

**`--full-refresh`:**
`dbt run --full-refresh` ignora el `is_incremental()` y reconstruye la tabla desde cero. Úsalo cuando:
- Cambiaste la definición de una columna
- Hay errores de lógica en datos históricos
- Cuando el `unique_key` cambió

---

## S17 — Tests

**Explica el comportamiento en detalle:**

**¿Cuándo se ejecutan los tests?**
- `dbt test` — solo tests, no ejecuta modelos
- `dbt build` — ejecuta modelo Y tests en orden, por nodo
- `dbt test --select bronze_tpch_orders` — tests de un modelo específico

**`dbt build` vs `dbt run + dbt test`:**
Con `dbt run && dbt test`: primero construyes TODOS los modelos, luego testeas TODOS. Si `bronze_tpch_orders` falla el test, `silver_orders_enriched` ya se construyó con datos malos.

Con `dbt build`: ejecuta `bronze_tpch_orders`, inmediatamente lo testea, y SOLO si pasa continúa con los modelos downstream. Detecta el problema antes de que se propague.

**`severity: error` vs `severity: warn`:**
Por defecto, un test fallido tiene `severity: error`: detiene la ejecución downstream y dbt devuelve exit code 1 (marca el build como fallido en CI).

Con `severity: warn`, el test falla pero dbt continúa ejecutando downstream y devuelve exit code 0 (CI sigue verde). Úsalo para tests informativos que no son bloqueantes.

**¿Qué genera un test fallido internamente?**
dbt transforma cada test en una SELECT query que devuelve las filas que FALLAN. Si la query devuelve 0 filas → test pasa. Si devuelve ≥1 filas → test falla. Para un test `unique` sobre `order_id`, dbt genera internamente:
```sql
SELECT order_id
FROM analytics.bronze.bronze_tpch_orders
WHERE order_id IS NOT NULL
GROUP BY order_id
HAVING COUNT(*) > 1
```

**Los 4 tests genéricos explicados:**
- `unique` — no hay valores duplicados
- `not_null` — no hay NULL
- `accepted_values` — solo existen los valores listados (útil para status codes, categorías)
- `relationships` — la FK existe en la tabla padre (integridad referencial que Snowflake no enforcea por defecto)

---

## S18 — Documentación

**La documentación de dbt es más poderosa de lo que parece en la diapositiva:**

**Qué contiene el sitio generado:**
- DAG interactivo completo del proyecto — puedes hacer click en cada nodo
- Página por modelo: descripción, columnas, tests, SQL compilado, linaje upstream/downstream
- Página por source: descripción, frescura, tablas
- Buscador global por nombre de modelo o columna
- Nodos de exposure (qué dashboards/herramientas consumen qué modelos)

**Doc blocks** — para reutilizar descripciones largas:
```markdown
<!-- models/docs.md -->
{% docs customer_tier %}
Segmentación de clientes por revenue histórico:
- Platinum: > $500K
- Gold: > $100K
- Silver: > $0
- No Orders: sin pedidos
{% enddocs %}
```
Luego en schema.yml: `description: "{{ doc('customer_tier') }}"`. Escribe la descripción una vez, úsala en múltiples modelos.

**`persist_docs`** — persiste las descripciones como comentarios en los objetos de Snowflake:
```yaml
gold:
  +persist_docs:
    relation: true
    columns: true
```
Cuando alguien haga `DESCRIBE TABLE ANALYTICS.GOLD.GOLD_ORDERS` en Snowflake, verá las descripciones de cada columna directamente en el warehouse.

**Por qué la documentación es una característica de primera clase en dbt:**
En equipos grandes, la documentación desacoplada del código (en Confluence, en Notion, en un Google Doc) inevitablemente se desactualiza. Con dbt, las descripciones viven junto al SQL en el mismo archivo de Git. Si cambias la definición de una columna, la documentación está ahí en el mismo PR. La revisión de código incluye la revisión de la documentación.

---

## S19 — Seeds

**Añade los detalles operacionales:**

**¿Qué hace `dbt seed` exactamente?**
Lee todos los archivos `.csv` en la carpeta `seeds/`, los sube fila a fila a Snowflake, y crea una tabla por cada CSV. Por defecto, todas las columnas son `VARCHAR`. Para sobreescribir tipos:
```yaml
seeds:
  - name: customers_seed
    config:
      column_types:
        customer_id: integer
        account_balance: float
```

**`dbt seed` vs `dbt seed --full-refresh`:**
Sin `--full-refresh`: si la tabla ya existe, dbt hace un truncate + insert (recarga los datos sin recrear la tabla).
Con `--full-refresh`: elimina y recrea la tabla (útil si cambiaste los tipos de columna).

**¿Por qué RAW y no BRONZE?**
Los seeds representan datos que "llegan" al warehouse directamente — conceptualmente son como una fuente de datos, no una transformación. Por convención van a un schema `RAW` separado, junto con las tablas cargadas por Fivetran/Airbyte. Los modelos bronze son el primer paso de transformación de esos datos.

**Cuándo usar seeds vs cuándo usar sources:**
Seeds son para datos que tú mismo controlas y que tiene sentido versionar en Git (porque quieres ver el historial de cambios en un Pull Request). Ejemplos: mapeos de categorías, calendarios fiscales, listas de países. Sources son para datos que llegan desde sistemas externos — no tienen sentido en Git porque cambian continuamente.

---

## S20 — Snapshots

**Explica SCD Type 2 desde cero:**

**¿Qué es un SCD Type 2 (Slowly Changing Dimension)?**
Un patrón de modelado de datos para rastrear cambios históricos en tablas de dimensiones. En lugar de sobreescribir el valor anterior cuando algo cambia, guardas AMBAS versiones con las fechas de validez.

**Ejemplo sin snapshots:**
```
customer_id | country | (última actualización)
1           | ES      | hoy
```
Si mañana el cliente se muda a DE, el registro queda:
```
customer_id | country
1           | DE
```
El valor anterior (ES) se perdió para siempre.

**Con snapshots (SCD Type 2):**
```
customer_id | country | dbt_valid_from        | dbt_valid_to
1           | ES      | 2024-01-01 10:00:00   | 2024-06-15 08:30:00
1           | DE      | 2024-06-15 08:30:00   | NULL
```
`NULL` en `dbt_valid_to` = registro actual vigente. Puedes responder: "¿en qué país estaba el cliente cuando hizo ese pedido?" con un JOIN usando las fechas de validez.

**Las 4 columnas que dbt añade automáticamente:**
- `dbt_scd_id` — clave subrogada única para cada versión del registro (MD5 del unique_key + updated_at)
- `dbt_updated_at` — timestamp de cuando dbt procesó esa fila
- `dbt_valid_from` — inicio de validez de esa versión
- `dbt_valid_to` — fin de validez (NULL si es la versión actual)

**Las dos estrategias:**
- `timestamp`: compara el campo `updated_at` de la fuente. Más eficiente. Requiere que la fuente tenga un campo de timestamp confiable.
- `check`: compara columna por columna (las que especifiques). Más robusto para fuentes sin timestamps, pero más costoso porque tiene que comparar todos los valores.

**Caso de uso real del curso:**
Los pedidos de TPC-H son read-only (no cambian). Por eso usamos `customers_seed` — una tabla de 30 clientes que TÚ puedes modificar directamente en Snowflake con un `UPDATE`. Simula un sistema fuente real donde los clientes cambian de email, país, etc.

---

## S21 — Macros y Jinja

**Explica cuándo usar macros vs CTEs:**

Un CTE resuelve el problema de "quiero dividir este modelo complejo en pasos". Una macro resuelve el problema de "tengo la misma lógica SQL repetida en MÚLTIPLES modelos".

**Regla práctica:**
- Lógica que solo aparece una vez → CTE dentro del modelo
- Lógica que aparece en 2+ modelos → macro

**Jinja en dbt tiene dos modos:**
1. **Expresiones** `{{ }}` — se evalúan y su resultado se inserta en el SQL
2. **Statements** `{% %}` — control de flujo (if, for, set) — no generan output directamente

**`{{ this }}`** — el nombre completo de la relación del modelo actual. Solo disponible dentro de modelos incrementales. Si estás en `gold_orders` y corres en dev, `{{ this }}` resuelve a `ANALYTICS.GOLD.GOLD_ORDERS`.

**`{{ execute }}`** — vale `True` cuando dbt está ejecutando SQL (run time), y `False` cuando solo está compilando (compile time). Importante en macros que hacen run_query — evita ejecutar queries durante la compilación.

**`dbt run-operation`** — ejecuta una macro directamente, sin que esté asociada a ningún modelo. Útil para tareas operacionales:
```bash
dbt run-operation drop_old_relations --args '{"schema": "bronze", "days_old": 30}'
```

**`run_query()`** — permite ejecutar SQL dentro de una macro y capturar el resultado como una variable. Úsalo con cuidado — solo tiene sentido dentro de macros operacionales, no en modelos.

---

## S22 — Packages

**Explica el ecosistema:**

Los packages de dbt son repositorios de Git con modelos, macros y tests predefinidos. Se instalan con `dbt deps` y su código queda en la carpeta `dbt_packages/` (que se excluye del repositorio con `.gitignore`).

**`dbt_utils` — el más importante:**
Creado por dbt Labs. Tiene más de 40 macros. Las más usadas:
- `generate_surrogate_key(['col1', 'col2'])` — MD5 de columnas combinadas para crear una clave subrogada
- `star(from=ref('...'), except=['col1'])` — SELECT * excepto ciertas columnas
- `date_spine(datepart, start_date, end_date)` — genera una secuencia completa de fechas (para encontrar días sin datos)
- `expression_is_true(expression)` — test que valida una expresión SQL arbitraria

**Dónde encontrar packages:**
hub.getdbt.com — directorio oficial de packages. Actualmente >400 packages.

**Versioning:**
```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: [">=1.0.0", "<2.0.0"]  # range semver
```
No uses `version: latest` en producción — un update de un package podría romper tu pipeline.

---

## S23 — dbt Core vs dbt Cloud

**La diapositiva tiene un error menor:** en la fila "Environment management" dice "Dev / Bronze / Prod" — debería decir "Dev / Staging / Prod".

**La decisión Core vs Cloud:**
La mayoría de equipos pequeños (1-5 personas) empiezan con dbt Core + GitHub Actions para CI. Cuando el equipo crece, la complejidad de mantener Airflow/Prefect + la infraestructura de CI + el hosting de docs justifica pagar por dbt Cloud.

**Precio de dbt Cloud (referencia aproximada, verificar en getdbt.com):**
- Free: 1 developer seat, 1 job, hosting de docs básico
- Team: ~$100/seat/mes, múltiples jobs, Slim CI, alertas
- Enterprise: precio negociado, SSO, RBAC avanzado, audit logs

**En este curso:**
Días 1-4: dbt Cloud IDE (el más cómodo para aprender).
Día 5: configuramos dbt Core localmente y vemos cómo las dos opciones son complementarias.

---

## S24 — dbt + Snowflake

**Explica los objetos de Snowflake que dbt usa:**

**Schemas:**
En este curso, cada estudiante tiene su propia cuenta de Snowflake, así que no hay riesgo de colisión. Los modelos van a:
- `ANALYTICS.BRONZE.BRONZE_TPCH_ORDERS`
- `ANALYTICS.SILVER.SILVER_ORDERS_ENRICHED`
- `ANALYTICS.GOLD.GOLD_ORDERS`

El schema se determina por el `+schema:` en `dbt_project.yml` + el macro `generate_schema_name` que configuramos en el Día 1.

**Transient tables:**
Snowflake tiene un tipo especial: `transient table`. No tiene fail-safe (el periodo de recuperación de 7 días que Snowflake cobra extra). Para tablas intermedias de silver/bronze que puedes reconstruir fácilmente desde raw, usar transient ahorra ~40% del coste de almacenamiento. En dbt: `{{ config(materialized='table', transient=true) }}`.

**Query tags:**
Snowflake permite taggear queries para tracking de costes. dbt puede configurar automáticamente:
```yaml
# dbt_project.yml
query-comment:
  comment: "dbt {{ invocation_id }} | {{ node.unique_id }}"
```
En Snowflake, cada query aparece en el Query History con el modelo que la generó. Fundamental para saber qué modelos consumen más créditos.

**Roles y permisos:**
- El role `TRANSFORMER` que configuramos tiene permisos para: crear/reemplazar objetos en ANALYTICS, leer desde SNOWFLAKE_SAMPLE_DATA.
- Nunca le das a dbt el rol `ACCOUNTADMIN` — principio de mínimo privilegio.

---

## S25 — Comandos Esenciales

**Añade los que faltan en la diapositiva:**

```bash
# Ver el SQL compilado sin ejecutarlo
dbt compile --select gold_orders

# Pasar variables al proyecto
dbt run --vars '{"start_date": "2024-01-01"}'

# Slim CI — solo modelos modificados vs producción
dbt build --select state:modified+ --defer --state ./prod-manifest

# Ver qué modelos cambiarían (dry run del selector)
dbt ls --select state:modified+

# Verificar configuración y conexión
dbt debug

# Ver todos los modelos del proyecto
dbt ls
dbt ls --select gold.*
```

**`dbt compile`** — muy útil para debugging. Resuelve todo el Jinja y muestra el SQL final que se enviaría a Snowflake, sin ejecutar nada. También es lo que hace el botón "Compile" del Cloud IDE.

**El orden en `dbt build`:**
seeds → snapshots → models → tests, en orden topológico. Para cada nodo: primero run, luego tests. Si los tests fallan, los nodos downstream quedan como SKIPPED.

---

## S26 — Best Practices

**La diapositiva solo tiene la mitad del grid. Aquí está todo:**

**Más prácticas que mencionar:**

**Cobertura de tests mínima recomendada:**
- `unique` + `not_null` en todas las PKs
- `accepted_values` en todos los campos de status/categoría
- `relationships` en todas las FKs hacia tablas de dimensión
- Al menos 1 test singular por modelo gold

**Commits pequeños y descriptivos:**
Convención: `feat: add gold_supplier_revenue model`, `fix: exclude cancelled orders from net_revenue`, `docs: add column descriptions for gold_orders`. Un commit = un cambio lógico. No commits de "varios cambios".

**No hacer `SELECT *` en modelos silver/gold:**
Siempre lista las columnas explícitamente. `SELECT *` oculta la estructura del modelo a cualquiera que lo lea — y si la fuente añade columnas, las heredas silenciosamente.

**Usar variables para lógica de entorno:**
```sql
{% if var('is_test_run', false) %}
  LIMIT 100
{% endif %}
```
Ejecutar con `dbt run --vars '{"is_test_run": true}'` en desarrollo para correr rápido.

**Convención de nombres:**
- `bronze_<source>_<tabla>` — ej: `bronze_tpch_orders`
- `silver_<dominio>_<descripcion>` — ej: `silver_orders_enriched`
- `gold_<entidad o hecho>` — ej: `gold_customers`, `gold_daily_revenue`

---

## S27 — Comandos Git Esenciales

**Añade el flujo completo que usarán en el curso:**

```bash
# Flujo diario en este curso:

# 1. Antes de empezar a trabajar
git checkout main && git pull origin main
git checkout -b feat/bronze-lineitem

# 2. Mientras desarrollas
dbt run --select mi_modelo
dbt test --select mi_modelo
git diff                          # qué cambié
git add models/bronze/mi_modelo.sql
git commit -m "feat: add bronze_tpch_lineitem"

# 3. Antes de hacer PR
dbt build --select +mi_modelo+   # modelo + upstream + downstream tests
git push origin feat/bronze-lineitem

# 4. En GitHub: abrir Pull Request hacia main
# 5. dbt Cloud ejecuta CI automáticamente sobre el PR
# 6. Merge si CI pasa
# 7. Limpiar
git checkout main && git pull
git branch -d feat/bronze-lineitem
```

**La conexión entre ramas Git y entornos dbt:**
- Rama `feat/*` → schema de desarrollo (personal, nadie más lo ve)
- Rama `main` → schema de producción (gold, silver, bronze compartidos)

---

## S28 — Lo Que Construiremos Esta Semana

Aquí es donde conectas la teoría abstracta con lo concreto. Di:

> "Todo lo que acabamos de ver — models, sources, refs, materializations, tests, snapshots, macros — lo van a implementar sobre este pipeline real. No estamos construyendo ejemplos de juguete. Esta es la arquitectura que verían en producción en una empresa real."

Señala el DAG del slide y recorrelo de arriba abajo:
1. TPC-H es una base de datos de benchmark que viene gratis en toda cuenta de Snowflake. Simula una empresa que vende partes a clientes en todo el mundo — 6 millones de line items, 1.5M de órdenes, 150K clientes.
2. Bronze: renombramos columnas (de `o_orderkey` a `order_id`) y declaramos sources
3. Silver: joins entre tablas, enriquecimiento, agregaciones intermedias
4. Gold: modelos finales listos para BI — `gold_orders`, `gold_customers`, `gold_daily_revenue`

---

## S29 — Cierre

Termina con:

> "Hay una cosa que les garantizo: las primeras dos veces que corran `dbt run` y vean el output verde, va a haber un momento de 'ah, esto tiene sentido'. Y la primera vez que introduzcan un error y vean cómo el test downstream falla con exactamente el mensaje correcto — eso es cuando entienden por qué dbt existe."

Abre preguntas, luego ve directo a Exercise 01.

---

## Preguntas frecuentes de la audiencia

**"¿Podemos usar dbt con Python?"**
Sí, dbt tiene "Python models" — archivos `.py` en lugar de `.sql` que usan Snowpark (en Snowflake) o Spark. Útiles para transformaciones que SQL no puede hacer bien (ML, estadísticas complejas). No lo veremos en este curso — el 95% del trabajo del día a día es SQL.

**"¿Cuánto tiempo tarda en aprenderse dbt?"**
El 80% del trabajo diario usa el 20% de las features: `run`, `test`, `ref()`, `source()`, y materializations básicas. Con una semana intensiva como esta, pueden trabajar productivamente en un equipo real. La profundidad (macros avanzados, Slim CI, snapshots) llega con la práctica.

**"¿dbt reemplaza a Airflow?"**
No. dbt gestiona las dependencias DENTRO de un run (qué modelos se ejecutan en qué orden). Airflow gestiona CUÁNDO se lanza ese run y en qué secuencia con otros pipelines (ej: "corre dbt después de que Fivetran termine de cargar datos"). En la práctica, muchos equipos usan los dos juntos, o usan el scheduler de dbt Cloud en lugar de Airflow para pipelines simples.

**"¿Funciona dbt con otros warehouses aparte de Snowflake?"**
Sí: BigQuery, Redshift, Databricks, DuckDB, PostgreSQL, Spark, y más. El código SQL es el mismo — lo que cambia es el adapter (instalado como librería Python). En este curso usamos el adapter de Snowflake (`dbt-snowflake`).

**"¿Qué es dbt Semantic Layer?"**
Una feature más reciente de dbt que permite definir métricas de negocio (revenue, MAU, churn rate) en YAML una sola vez, y consultarlas desde cualquier herramienta BI conectada. No lo veremos en este curso pero es el futuro de la capa de semántica.
