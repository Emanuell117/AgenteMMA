# AgenteMMA

Asistente que responde preguntas sobre peleadores de MMA usando un modelo de lenguaje y herramientas que consultan una tabla de datos.

## Integrantes
* Emanuell Torres López
* David Cossio

**Opción elegida:** B. Agente con herramientas.

## 1. Definición del problema

### 1. ¿Qué problema se quiere resolver y para quién?
Un aficionado al MMA que quiere consultar datos de peleadores o compararlos debe buscar en varias páginas y hacer las comparaciones a mano. AgenteMMA es un agente al que se le pregunta en lenguaje natural y que consulta una tabla de peleadores para responder con datos verificables. Está pensado para aficionados al MMA y para quienes quieren una consulta rápida sin revisar varias fuentes.

### 2. ¿Qué preguntas o tareas podrá atender el prototipo?
* **Consulta:** "¿Cuál es el récord y el alcance de Jose Aldo?"
* **Comparación:** "Compara a Jose Aldo y Urijah Faber. ¿Quién tiene más victorias y mayor alcance?"
* **Filtro por criterio:** "¿Qué peleadores de peso gallo tienen al menos 31 victorias?" o "Los 3 con mayor alcance".
* **Tareas combinadas:** "Busca los 2 peleadores de peso gallo con más victorias y compáralos entre sí."

### 3. ¿Por qué eligieron un agente para ese problema?
El modelo por sí solo no conoce los datos exactos de la tabla y puede inventarlos o equivocarse al comparar. Con herramientas, el modelo decide cuál usar y con qué argumentos, la aplicación ejecuta la consulta sobre datos reales y devuelve el resultado al modelo, que redacta la respuesta. Los datos son estructurados (una tabla con números), así que filtrar, ordenar y comparar con funciones es más preciso que buscar fragmentos de texto por similitud, que es lo que haría un RAG. Además, las respuestas se pueden verificar contra el CSV.

### 4. ¿Qué queda por fuera del alcance?
* Predicción de resultados de peleas y apuestas.
* Datos en tiempo real o actualización automática de la tabla.
* Planes de entrenamiento o consejos médicos.
* Estadísticas detalladas por pelea y múltiples agentes.

## 2. Descripción del agente

### Datos
Archivo `peleadores.csv` extraido a partir del dataset público *UFC Fighters' Statistics Dataset* (datos extraídos de ufcstats.com, publicado en Kaggle y Hugging Face). Se dejan los peleadores con más victorias y al menos 12 peleas, y se agregó manualmente a Ilia Topuria con datos de 2026.

| Columna | Descripción |
|---|---|
| `nombre` | Nombre del peleador |
| `categoria` | Categoría de peso (Paja, Mosca, Gallo, Pluma, Ligero, Welter, Mediano, Semipesado, Pesado), derivada del peso |
| `victorias`, `derrotas`, `empates` | Récord profesional |
| `altura_cm`, `alcance_cm` | Medidas en centímetros |
| `peso_lbs` | Peso en libras |

### Herramientas
Son funciones Python que consultan el CSV con pandas. Si no encuentran lo pedido, devuelven un error con sugerencias en lugar de fallar.

| Herramienta | Argumentos | Qué hace |
|---|---|---|
| `consultar_peleador` | `nombre` | Devuelve la ficha del peleador: categoría, récord, porcentaje de victorias, altura, alcance y peso. |
| `comparar_peleadores` | `nombre1`, `nombre2` | Compara dos peleadores en victorias, derrotas, porcentaje de victorias, altura y alcance, e indica quién tiene ventaja en cada atributo. |
| `buscar_por_criterio` | `categoria`, `min_victorias`, `ordenar_por`, `n` | Filtra por categoría y mínimo de victorias, ordena de mayor a menor por un atributo y devuelve los `n` primeros (máximo 20). Todos los argumentos son opcionales. |

Los nombres se buscan sin importar mayúsculas ni acentos. Si un nombre coincide con varios peleadores (por ejemplo "Silva"), la herramienta devuelve la lista de coincidencias.

### Cómo se le describen las herramientas al modelo
Cada herramienta se declara con un esquema JSON (nombre, descripción y argumentos con su tipo), que es lo único que ve el modelo. Con eso el modelo elige la herramienta adecuada y sus argumentos. Se usa NVIDIA NIM (modelo `meta/llama-3.3-70b-instruct`) con su API compatible con OpenAI.

### Ciclo del agente
1. El usuario hace una pregunta.
2. El modelo responde con texto o pide una herramienta con sus argumentos.
3. La aplicación ejecuta la herramienta y devuelve el resultado al modelo.
4. Se repite hasta que el modelo responde o se alcanza el límite de llamadas.

### Límite de llamadas
Máximo de 5 llamadas a herramientas por pregunta, para evitar ciclos infinitos y controlar el costo y el tiempo. Cuando se alcanza, la aplicación deja de ejecutar herramientas, le indica al modelo que ya no puede consultar más y le pide que responda con la información que obtuvo, avisando qué parte de la tarea no pudo completar. El notebook incluye una demostración que baja el límite a 1 para mostrar este comportamiento.

## 3. Pruebas y análisis

La evaluación es manual: se compara la respuesta con los datos del CSV. En cada caso el notebook imprime la herramienta llamada, sus argumentos, el resultado y la respuesta final.

| # | Tipo | Pregunta | Resultado esperado (según el CSV) | Respuesta obtenida | Valoración |
|---|---|---|---|---|---|
| 1 | Habitual | ¿Cuál es el récord, la altura y el alcance de Urijah Faber? | 35 victorias, 11 derrotas, 0 empates; 168 cm de altura; 170 cm de alcance. Herramienta: `consultar_peleador`. | *(completar)* | *(completar)* |
| 2 | Habitual | Compara a Jose Aldo y Urijah Faber. ¿Quién tiene más victorias y mayor alcance? | Faber tiene más victorias (35 vs 31); Aldo tiene mayor alcance (178 vs 170 cm). Herramienta: `comparar_peleadores`. | *(completar)* | *(completar)* |
| 3 | Habitual | ¿Qué peleadores de peso gallo tienen al menos 31 victorias? | Miguel Torres, Jeff Curran, Iuri Alcantara, Urijah Faber, Jose Aldo y John Lineker. Herramienta: `buscar_por_criterio`. | *(completar)* | *(completar)* |
| 4 | Exige herramientas | Busca los 2 peleadores de peso gallo con más victorias y compáralos entre sí. | Miguel Torres (44) y Jeff Curran (36). Encadena `buscar_por_criterio` y `comparar_peleadores`. | *(completar)* | *(completar)* |
| 5 | Caso límite | Compara a Jose Aldo con Carlos Prates. | El peleador no está en la tabla: el agente debe decirlo y no inventar datos. | *(completar)* | *(completar)* |

### Error o limitación observada y posible mejora
*(completar después de ejecutar las pruebas)*

* **Error o limitación:**
* **Posible mejora:**

## 4. Guía de ejecución


### Fuentes y herramientas
* Dataset: *UFC Fighters' Statistics Dataset* (Kaggle y Hugging Face), con datos de [ufcstats.com](http://ufcstats.com).
* Modelo: `meta/llama-3.3-70b-instruct` a través de NVIDIA NIM.
* Librerías: `openai`, `pandas`.