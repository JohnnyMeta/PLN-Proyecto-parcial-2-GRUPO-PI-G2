# Chatbot de Admisiones y Nivelación — Universidad de Guayaquil

Agente conversacional en español que resuelve dudas sobre el **proceso de admisión y el curso de nivelación** de la Universidad de Guayaquil (UG). 

Está construido con **PLN clásico**: lematización con spaCy, representación **TF-IDF implementada desde cero**, clasificación de intención por **similitud coseno** y **extracción de entidades por expresiones regulares**, con una interfaz web en **Gradio**.

Toda la información base proviene de la página oficial de admisiones de la UG (`https://admision.ug.edu.ec/admision/`) y se transformó en tres archivos JSON que funcionan como base de datos del sistema.

---

## Docente

|            Ángel Eduardo Cuenca Ortega             |


## Grupo

|     Rol     |               Nombre                 |
|-------------|--------------------------------------|
| Docente     | Ángel Eduardo Cuenca Ortega          |
|-------------|--------------------------------------|
| Integrante  | Sergio Bernard Luna Vásquez          |
| Integrante  | Alejandra Janelly Salavarría Estrada |
| Integrante  | Johnny Aarón Ortíz López             |
|-------------|--------------------------------------|


**Materia:** Procesamiento de Lenguaje Natural (PLN) · **Carrera:** Ciencia de Datos e IA · **Paralelo:** CDDEIA-ELNO-4-2 · 4.º semestre · Proyecto Parcial II · Grupo PI-G2.

---

## ¿Qué hace el sistema?

Responde en lenguaje natural preguntas como:

- *"¿Qué requisitos necesito para inscribirme?"*
- *"¿Cuándo me toca inscribirme si mi cédula termina en 5?"*
- *"¿Qué materias evalúa el bloque 3?"*
- *"Cuéntame sobre la carrera de Software"*
- *"¿Cómo apruebo el curso de nivelación?"*

Si no entiende la consulta (baja confianza), responde con un mensaje de *fallback* pidiendo reformular.

---

## Arquitectura y flujo (paso a paso)

El proyecto son **4 cuadernos de Jupyter** que se ejecutan en un orden **secuencial y dependiente**. 

Los cuadernos `02` y `03` producen los datos crudos; esos JSON se suben a **GitHub**; el `01` los combina en el `corpus.json`; y el `main` consume los tres archivos para conversar.

```
┌─────────────────────────────┐     ┌───────────────────────────────────────┐
│ 02_generar_entidades.ipynb  │     │  03_generar_base_conocimiento.ipynb   │
│      -> entidades.json      │     │       ->base_conocimiento.json        │
│         (NER / regex)       │     │           (ontología / datos)         │
└──────────┬──────────────────┘     └─────────────┬─────────────────────────┘
           │                                      │
           └─────────────────────┬────────────────┘
                                 |
                    ┌───────────────────────────┐
                    │     Repositorio GitHub    │
                    │  (carpeta data/ = backend)│
                    └───────────┬───────────────┘
                                |
                    ┌───────────────────────────┐
                    │ 01_generar_corpus.ipynb   │
                    │  lee entidades + base     │
                    │         ->corpus.json     │
                    │  (25 intenciones)         │
                    └───────────┬───────────────┘
                                |
                ┌─────────────────────────────────────┐
                │   PLN_PARCIAL_2_GRUPO-PI-G2.ipynb   │
                │      spaCy + TF-IDF + coseno        │
                │      + regex + Gradio               │
                └─────────────────────────────────────┘
```

### Paso 1 · `02_generar_entidades.ipynb` -> `entidades.json`
Genera el catálogo de **entidades reconocibles** dentro del texto del usuario (para el NER por regex): **17 facultades**, **59 carreras** (cada una con sus alias, `facultad_id` y `bloque_id`), **10 procesos académicos**, **7 documentos** y **3 patrones RegEx** (cédula, fecha, bloque). Solo guarda `id`, `nombre` y `alias`: nada de texto narrativo.

### Paso 2 · `03_generar_base_conocimiento.ipynb` -> `base_conocimiento.json`
Genera la **base de conocimiento** con la que se redactan las respuestas: los **6 bloques de conocimiento** y sus materias evaluadas (%, número de preguntas), las **fichas completas de las 59 carreras** (título, duración, modalidad, sede, descripción, perfil), el **cronograma oficial** (fechas clave + inscripción por último dígito de cédula), las **cuotas de admisión / acción afirmativa**, la **información institucional** y el detalle del **curso de nivelación**.

### Paso 3 · Repositorio GitHub (backend de datos)
Los JSON de los pasos 1 y 2 (más `banner_ug.jpg`) se alojan en la carpeta `data/` del repositorio:
`https://github.com/JohnnyMeta/PLN-Proyecto-parcial-2-GRUPO-PI-G2.git`
Esto desacopla los datos del código: los notebooks los descargan con `git clone` / `git pull` al arrancar.

### Paso 4 · `01_generar_corpus.ipynb` -> `corpus.json`
Descarga `entidades.json` y `base_conocimiento.json` desde GitHub y construye el **corpus de intenciones** (`corpus.json`): **25 intenciones**, cada una con su `descripcion`, sus `utterances` (frases de entrenamiento) y sus `responses`. Las respuestas se redactan **con datos reales** tomados de la base de conocimiento (fechas, cuotas, materias), de modo que si cambia una fecha oficial basta regenerar el corpus.

### Paso 5 · `PLN_PARCIAL_2_GRUPO-PI-G2.ipynb` -> el chatbot
Es el código main. Descarga los **tres** JSON y ejecuta el pipeline de PLN:

1. **Preprocesamiento** -> minúsculas, eliminación de tildes, tokenización, remoción de stopwords/puntuación/números y **lematización** con `es_core_news_sm`.
2. **TF-IDF desde cero** (NumPy) -> vocabulario, matriz TF, vector IDF y matriz TF-IDF.
3. **Detección de intención** -> **similitud coseno** entre la consulta y cada *utterance*.
4. **Extracción de entidades** -> RegEx generado dinámicamente desde `entidades.json`; desambiguación carrera/materia e inferencia de bloque.
5. **Gestor de diálogo** -> **umbral de confianza 0.35**; respuestas *dinámicas* (usan las entidades para dar el dato exacto), respuestas estáticas del corpus o *fallback*.
6. **Evaluación** -> accuracy, `classification_report` y matriz de confusión sobre un banco de 26 consultas (con errores ortográficos, jerga y preguntas fuera de dominio).
7. **Interfaz** -> consola y panel web **Gradio** con identidad visual de la UG.

---

## Requisitos

- **Python 3.12+** (pensado para **Google Colab**).
- Librerías: `spacy` (+ modelo `es_core_news_sm`), `numpy`, `pandas`, `scikit-learn`, `matplotlib`, `seaborn`, `gradio`.
- `git` disponible en el entorno (para sincronizar la carpeta `data/`).
- Conexión a internet (para clonar el repositorio y para el enlace público de Gradio).

Instalación rápida (si se ejecuta fuera de Colab):

```bash
pip install spacy numpy pandas scikit-learn matplotlib seaborn gradio
python -m spacy download es_core_news_sm
```

---

## Cómo ejecutar

- El sistema está diseñado para **Google Colab**. El orden importa: se deben generar los JSON, `02` y `03`, luego `01`, ejecutando el código respectivo y por último el y por último `PLN_PARCIAL_2_GRUPO-PI-G2.ipynb `.

**Opción A — Solo usar el chatbot (lo más común).**
Como los JSON ya están publicados en GitHub, basta con:

1. Abrir `PLN_PARCIAL_2_GRUPO-PI-G2.ipynb ` en Colab.
2. Ejecutar todas las celdas en orden. La primera fase instala spaCy, la segunda descarga la carpeta `data/` desde GitHub y las siguientes entrenan el modelo TF-IDF.
3. En la última celda, Gradio imprime un **enlace público temporal** (`*.gradio.live`).


**Opción B — Regenerar los datos desde cero en caso que la sincronización con Github fallase**

1. Ejecutar `02_generar_entidades.ipynb` -> descarga `entidades.json`.
2. Ejecutar `03_generar_base_conocimiento.ipynb` -> descarga `base_conocimiento.json`.
3. Subir ambos JSON a la carpeta `data/` del repositorio de Entorno del código: 01_generar_corpus.ipynb 
4. Ejecutar `01_generar_corpus.ipynb` -> genera `corpus.json` (y luego subir los tres archivos a `data/` pero del entorno del código principal).
5. Al ejecutar `PLN_PARCIAL_2_GRUPO-PI-G2.ipynb ` se generará la carpeta `data/` facilitando el proceso, el resto de pasos será como en la Opción A.


---

## Estructura de archivos

```
PLN-Proyecto-parcial-2-GRUPO-PI-G2/
├── data/
│   ├── entidades.json           # generado por 02 (NER)
│   ├── base_conocimiento.json   # generado por 03 (ontología/datos)
│   ├── corpus.json              # generado por 01 (intenciones)
│   └── banner_ug.jpg            # banner para la interfaz Gradio
├── src/
│   ├── 01_generar_corpus.ipynb
│   ├── 02_generar_entidades.ipynb
│   ├── 03_generar_base_conocimiento.ipynb
│   ├── 04_generar_readme.ipynb
│   └── PLN_PARCIAL_2_GRUPO-PI-G2.ipynb
├── README.md                      
└── requirements.txt               
```

---

## Datos clave del sistema

| Elemento                       |          Cantidad         |
|--------------------------------|---------------------------|
| Intenciones en el corpus       |             25            |
| Facultades                     |             17            |
| Carreras                       |             59            |
| Bloques de conocimiento        |              6            |
| Procesos académicos (entidades)|             10            |
| Documentos (entidades)         |              7            |
| Patrones RegEx                 | 3 (cédula, fecha, bloque) |
| Umbral de confianza (coseno)   |           0.35            |

---

## Tecnologías utilizadas

**spaCy** (lematización en español) · **NumPy** (TF-IDF y álgebra vectorial) · **scikit-learn** (métricas de evaluación) · **pandas / seaborn / matplotlib** (reportes y matriz de confusión) · **Gradio** (interfaz web) · **Git + GitHub** (backend de datos) · **Google Colab** (entorno de ejecución).

---

## Fuente oficial

Toda la información se basa en los canales institucionales de la Universidad de Guayaquil.
Este proyecto es de carácter **académico**.