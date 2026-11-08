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
* **Comparación:** "Compara a Jose Aldo e Ilia Topuria. ¿Quién tiene más victorias y mayor alcance?"
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
Cada herramienta se declara con un esquema JSON (nombre, descripción y argumentos con su tipo), que es lo único que ve el modelo. Con eso el modelo elige la herramienta adecuada y sus argumentos. Se usa NVIDIA NIM, se busca un modelo disponible ya que NVIDIA los cambia con el tiempo.

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
| 1 | Habitual | ¿Cuál es el récord, la altura y el alcance de Urijah Faber? | 35 victorias, 11 derrotas, 0 empates; 168 cm de altura; 170 cm de alcance. Herramienta: `consultar_peleador`. | Llamó `consultar_peleador('Urijah Faber')` y respondió: 35 victorias, 11 derrotas, 0 empates (76.1 %), 168 cm de altura y 170 cm de alcance. Añadió que pelea en categoría Gallo (135 lb). | Correcto. Coincide con el CSV y con el resultado de la herramienta. |
| 2 | Habitual | Compara a Jose Aldo e Ilia Topuria. ¿Quién tiene más victorias y mayor alcance? | Jose Aldo tiene más victorias (31 vs 17) y mayor alcance (178 vs 175 cm). Herramienta: `comparar_peleadores`. | Llamó `comparar_peleadores('Jose Aldo', 'Ilia Topuria')` y respondió que Aldo tiene más victorias (31 vs 17) y mayor alcance (178 vs 175 cm). | Correcto según el CSV. Observación: no mencionó que Topuria tiene mejor porcentaje de victorias (94.4 % vs 79.5 %) ni que pelean en categorías distintas. |
| 3 | Habitual | ¿Qué peleadores de peso gallo tienen al menos 31 victorias? | Miguel Torres, Jeff Curran, Iuri Alcantara, Urijah Faber, Jose Aldo y John Lineker. Herramienta: `buscar_por_criterio`. | Llamó `buscar_por_criterio(categoria='Gallo', min_victorias=31, ordenar_por='victorias', n=20)`. Devolvió los 6 peleadores esperados con récord, porcentaje, altura y alcance. | Correcto y completo. Los argumentos reflejan la pregunta. |
| 4 | Exige herramientas | Busca los 2 peleadores de peso gallo con más victorias y compáralos entre sí. | Miguel Torres (44) y Jeff Curran (36). Encadena `buscar_por_criterio` y `comparar_peleadores`. | Llamó `buscar_por_criterio(categoria='Gallo', ordenar_por='victorias', n=2)` y luego `comparar_peleadores('Miguel Torres', 'Jeff Curran')`. Concluyó que Torres tiene ventaja en todos los atributos (44 vs 36 victorias, 9 vs 18 derrotas, 175 vs 168 cm, 193 vs 175 cm de alcance). | Correcto. Usó el resultado de la primera herramienta como argumentos de la segunda (2 llamadas, dentro del límite de 5). |
| 5 | Caso límite | Compara a Jose Aldo con Carlos Prates. | El peleador no está en la tabla: el agente debe decirlo y no inventar datos. | Llamó `comparar_peleadores('Jose Aldo', 'Carlos Prates')`. La herramienta devolvió un error con sugerencias (Carlo Prater, Carlos Condit). El agente dijo que no encontró a Carlos Prates y preguntó con cuál de las sugerencias comparar. | Correcto. No inventó datos y ofreció alternativas. |
 
Las salidas completas de cada caso están al final de esta sección.

### Error o limitación observada y posible mejora
* **Limitación 1: "no está en la base de datos" no es "no existe".** El CSV solo contiene los peleadores con más victorias (más los agregados a mano). Si se consulta a un peleador real que quedó fuera del filtro, el agente responde que no lo encontró, y un usuario podría entender que ese peleador no existe.
* **Limitación 2: comparaciones superficiales.** En el caso 2 el agente respondió solo lo preguntado (victorias absolutas y alcance) aunque la herramienta devolvió también el porcentaje de victorias, donde Topuria supera a Aldo (94.4 % vs 79.5 %), y las categorías son distintas (Gallo y Ligero). Comparar victorias absolutas favorece a quien lleva más años peleando.
* **Posible mejora:** ampliar el CSV (más peleadores o sin tope) y avisar en la respuesta que solo se consultó esa tabla. También se puede ajustar el prompt del sistema para que mencione cuando dos peleadores son de categorías distintas y que cite el porcentaje de victorias al comparar.

### Salidas completas del agente
Cada bloque muestra la pregunta, la herramienta llamada con sus argumentos, el resultado devuelto y la respuesta final.
 
<details>

<summary>Caso 1: consulta de un peleador</summary>

PREGUNTA: ¿Cuál es el récord, la altura y el alcance de Urijah Faber?
 
  Llamada 1: consultar_peleador({'nombre': 'Urijah Faber'})
  Resultado: {"nombre": "Urijah Faber", "categoria": "Gallo", "victorias": 35, "derrotas": 11, "empates": 0, "porcentaje_victorias": 76.1, "altura_cm": 168, "alcance_cm": 170, "peso_lbs": 135}
 
RESPUESTA FINAL: Aquí están los datos de **Urijah Faber**:
 
- **Récord:** 35 victorias, 11 derrotas, 0 empates (76.1% de victorias)
- **Altura:** 168 cm
- **Alcance:** 170 cm
 
Además, pelea en la categoría **Gallo** (135 lbs).

</details>

<details>
<summary>Caso 2: comparación de dos peleadores</summary>
PREGUNTA: Compara a Jose Aldo e Ilia Topuria. ¿Quién tiene más victorias y mayor alcance?

 
  Llamada 1: comparar_peleadores({'nombre1': 'Jose Aldo', 'nombre2': 'Ilia Topuria'})
  Resultado: {"peleador_1": {"nombre": "Jose Aldo", "categoria": "Gallo", "victorias": 31, "derrotas": 8, "empates": 0, "porcentaje_victorias": 79.5, "altura_cm": 170, "alcance_cm": 178, "peso_lbs": 135}, "peleador_2": {"nombre": "Ilia Topuria", "categoria": "Ligero", "victorias": 17, "derrotas": 1, "empates": 0, "porcentaje_victorias": 94.4, "altura_cm": 170, "alcance_cm": 175, "peso_lbs": 155}, "ventaja_por_...
 
RESPUESTA FINAL: Según los datos obtenidos:
 
**Más victorias:** **Jose Aldo** con **31 victorias** (frente a las 17 de Ilia Topuria).
 
**Mayor alcance:** **Jose Aldo** con **178 cm** (frente a los 175 cm de Ilia Topuria).
 
Por tanto, Jose Aldo aventaja a Ilia Topuria en ambos aspectos.
</details>

<details>
<summary>Caso 3: filtro por criterio</summary>

PREGUNTA: ¿Qué peleadores de peso gallo tienen al menos 31 victorias?
 
  Llamada 1: buscar_por_criterio({'categoria': 'Gallo', 'min_victorias': 31, 'ordenar_por': 'victorias', 'n': 20})
  Resultado: {"total_que_cumplen": 6, "mostrando": 6, "resultados": [{"nombre": "Miguel Torres", "categoria": "Gallo", "victorias": 44, "derrotas": 9, "empates": 0, "porcentaje_victorias": 83.0, "altura_cm": 175, "alcance_cm": 193, "peso_lbs": 135}, {"nombre": "Jeff Curran", "categoria": "Gallo", "victorias": 36, "derrotas": 18, "empates": 1, "porcentaje_victorias": 65.5, "altura_cm": 168, "alcance_cm": 175, ...
 
RESPUESTA FINAL: Encontré **6 peleadores de peso gallo con al menos 31 victorias**:
 
| Peleador | Victorias | Derrotas | % Victorias | Altura | Alcance |
|----------|-----------|----------|-------------|--------|---------|
| **Miguel Torres** | 44 | 9 | 83.0% | 175 cm | 193 cm |
| **Jeff Curran** | 36 | 18 | 65.5% | 168 cm | 175 cm |
| **Iuri Alcantara** | 35 | 10 | 77.8% | 175 cm | 180 cm |
| **Urijah Faber** | 35 | 11 | 76.1% | 168 cm | 170 cm |
| **Jose Aldo** | 31 | 8 | 79.5% | 170 cm | 178 cm |
| **John Lineker** | 31 | 9 | 77.5% | 160 cm | 170 cm |
 
Todos pelean en 135 lbs (peso gallo). Miguel Torres lidera la lista con 44 victorias.
</details>

<details>
<summary>Caso 4: encadena dos herramientas</summary>

PREGUNTA: Busca los 2 peleadores de peso gallo con más victorias y compáralos entre sí.
 
  Llamada 1: buscar_por_criterio({'categoria': 'Gallo', 'ordenar_por': 'victorias', 'n': 2})
  Resultado: {"total_que_cumplen": 10, "mostrando": 2, "resultados": [{"nombre": "Miguel Torres", "categoria": "Gallo", "victorias": 44, "derrotas": 9, "empates": 0, "porcentaje_victorias": 83.0, "altura_cm": 175, "alcance_cm": 193, "peso_lbs": 135}, {"nombre": "Jeff Curran", "categoria": "Gallo", "victorias": 36, "derrotas": 18, "empates": 1, "porcentaje_victorias": 65.5, "altura_cm": 168, "alcance_cm": 175, ...
 
  Llamada 2: comparar_peleadores({'nombre1': 'Miguel Torres', 'nombre2': 'Jeff Curran'})
  Resultado: {"peleador_1": {"nombre": "Miguel Torres", "categoria": "Gallo", "victorias": 44, "derrotas": 9, "empates": 0, "porcentaje_victorias": 83.0, "altura_cm": 175, "alcance_cm": 193, "peso_lbs": 135}, "peleador_2": {"nombre": "Jeff Curran", "categoria": "Gallo", "victorias": 36, "derrotas": 18, "empates": 1, "porcentaje_victorias": 65.5, "altura_cm": 168, "alcance_cm": 175, "peso_lbs": 135}, "ventaja_p...
 
RESPUESTA FINAL: Aquí están los **2 peleadores de peso gallo con más victorias**:
 
| Peleador | Victorias | Derrotas | Empates | % Victorias | Altura | Alcance |
|----------|-----------|----------|---------|-------------|--------|---------|
| **Miguel Torres** | 44 | 9 | 0 | 83.0% | 175 cm | 193 cm |
| **Jeff Curran** | 36 | 18 | 1 | 65.5% | 168 cm | 175 cm |
 
**Comparación directa:**
- 🥊 **Victorias:** Miguel Torres (44 vs 36)
- 🛡️ **Derrotas:** Miguel Torres tiene menos (9 vs 18)
- 📊 **Porcentaje de victorias:** Miguel Torres (83.0% vs 65.5%)
- 📏 **Altura:** Miguel Torres (175 cm vs 168 cm)
- 💪 **Alcance:** Miguel Torres (193 cm vs 175 cm)
 
**Conclusión:** Miguel Torres tiene ventaja en **todos los atributos** comparados.

</details>

<details>

<summary>Caso 5: caso límite (peleador inexistente en la tabla)</summary>

PREGUNTA: Compara a Jose Aldo con Carlos Prates.
 
  Llamada 1: comparar_peleadores({'nombre1': 'Jose Aldo', 'nombre2': 'Carlos Prates'})
  Resultado: {"error": "No se encontró ningún peleador llamado 'Carlos Prates' en la base de datos.", "sugerencias": ["Carlo Prater", "Carlos Condit"]}
 
RESPUESTA FINAL: No encontré a **"Carlos Prates"** en la base de datos. Las sugerencias que devuelve el sistema son:
 
- **Carlo Prater**
- **Carlos Condit**
 
¿Con cuál de estos dos quieres comparar a **Jose Aldo**?

</details>

## 4. Guía de ejecución

El prototipo es el notebook `agente_mma.ipynb`. Está pensado para Google Colab, pero también corre en Jupyter local.

### Requisitos
* Una clave de API de NVIDIA NIM. Se obtiene gratis creando una cuenta en [build.nvidia.com](https://build.nvidia.com).
* El archivo `peleadores.csv` de este repositorio.
* Python 3 con `openai` y `pandas`. La primera celda del notebook los instala.

### Paso 1. Configurar la clave
* **En Colab:** en el panel lateral, abrir *Secretos* (ícono de llave), crear un secreto llamado `NVIDIA_API_KEY` con la clave y activar el acceso del notebook.
* **En local:** definir la variable de entorno antes de abrir Jupyter:
  ```bash
  export NVIDIA_API_KEY="tu_clave"
  jupyter notebook agente_mma.ipynb
  ```

Si la clave no está configurada, el notebook se detiene con el mensaje *"Falta la clave..."*.

### Paso 2. Cargar los datos
* **En Colab:** al ejecutar la celda de la sección *2. Datos*, si `peleadores.csv` no está en la sesión, se abre un diálogo para subirlo.
* **En local:** dejar `peleadores.csv` en la misma carpeta del notebook.

La celda imprime cuántos peleadores se cargaron y las categorías disponibles.

### Paso 3. Ejecutar el notebook
Ejecutar las celdas en orden (en Colab, *Entorno de ejecución > Ejecutar todas*):

| Sección del notebook | Qué hace |
|---|---|
| 1. Instalación y configuración | Instala las librerías, conecta con NVIDIA NIM y elige el modelo. |
| 2. Datos | Carga `peleadores.csv`. |
| 3. Herramientas | Define las funciones que consultan la tabla. |
| 4. Descripción de las herramientas | Declara los esquemas JSON que ve el modelo. |
| 5. Agente | Define el prompt del sistema y el ciclo del agente. |
| 6. Prueba rápida | Hace una pregunta de ejemplo sobre Jose Aldo. |
| 7. Casos de prueba | Ejecuta los 5 casos de la sección 3, los verifica contra el CSV y demuestra el límite de llamadas (`max_llamadas=1`). |
| 8. Preguntar al modelo | Pide una pregunta por teclado y la responde. |

Para hacer otras preguntas se puede ejecutar de nuevo la última celda o llamar directamente a la función:
```python
agente("¿Quiénes son los 3 peleadores de peso pesado con mayor alcance?")
```

### Elección del modelo
NVIDIA cambia con el tiempo los modelos que ofrece, así que el notebook no fija uno. Recorre una lista de preferidos (`PREFERIDOS`), prueba cada uno con una llamada que usa herramientas y se queda con el primero que responde. Imprime el modelo en uso. En nuestras pruebas fue `mistralai/mistral-large-4-1t-a52b`.

Para forzar un modelo, se escribe su identificador completo en la variable `MODEL` de la primera celda de código, por ejemplo `MODEL = "mistralai/mistral-large-4-1t-a52b"`.

### Problemas frecuentes
| Problema | Solución |
|---|---|
| *"Falta la clave..."* | Revisar que el secreto se llame exactamente `NVIDIA_API_KEY` y que el notebook tenga acceso a él. |
| *"Ningún candidato funciona"* | Elegir en [build.nvidia.com/models](https://build.nvidia.com/models) un modelo que admita herramientas (*Tool Use*) y escribir su identificador en `MODEL`. |
| No encuentra `peleadores.csv` | Subir el archivo a la sesión de Colab o ponerlo en la carpeta del notebook. |
| Las respuestas varían entre ejecuciones | Es normal en un modelo de lenguaje. Los datos deben coincidir siempre con el CSV. Si el modelo cambió, la redacción también puede cambiar. |

### Fuentes y herramientas
* Dataset: *UFC Fighters' Statistics Dataset* (Kaggle y Hugging Face), con datos de [ufcstats.com](http://ufcstats.com).
* Modelo: `mistralai/mistral-large-4-1t-a52b` a través de NVIDIA NIM.
* Librerías: `openai`, `pandas`.