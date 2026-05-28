# Guía Práctica: Entendiendo el Código del Árbol de Decisión

Este documento explica paso a paso cómo funcionan los scripts de entrenamiento del **Árbol de Decisión** (`Anemia_DT_Local.py` y `Titanic_DT_Local.py`). Está escrito para que cualquier persona, sin importar su experiencia, pueda entender qué hace cada parte del código.

> [!NOTE]
> **¿Qué es un Árbol de Decisión?**
> Imagina que eres médico y tienes que decidir si un paciente tiene anemia. Primero preguntas: *"¿Su hemoglobina es menor a 12?"*. Si la respuesta es Sí, preguntas: *"¿Es mujer?"*. Dependiendo de cada respuesta, llegas a una conclusión. Un Árbol de Decisión hace exactamente esto, pero en lugar de que un médico escriba las preguntas, el modelo las descubre por sí solo analizando cientos o miles de casos del pasado.

---

## Diferencia clave con la Red Neuronal

Antes de empezar, la diferencia más importante:

| Característica | Red Neuronal | Árbol de Decisión |
|---|---|---|
| ¿Necesita normalizar datos? | **Sí** (StandardScaler) | **No** |
| ¿Cómo aprende? | Ajustando millones de pesos matemáticos | Encontrando las mejores preguntas de corte |
| ¿Es fácil de interpretar? | Difícil (caja negra) | Fácil (se puede dibujar como diagrama) |
| Archivos que genera | `.h5` + `.pkl` (scaler) | Solo `.pkl` |

---

## Bloque 1: Importar las Herramientas

```python
import numpy as np
import pandas as pd
from sklearn.tree import DecisionTreeClassifier
from sklearn.model_selection import train_test_split
from sklearn.metrics import confusion_matrix, classification_report
import joblib
```

**Explicación práctica:**
Las herramientas que necesitamos son notablemente más simples que las de la Red Neuronal. No hay TensorFlow ni Keras, porque no estamos construyendo nada que imite un cerebro.

| Librería | Para qué sirve |
|---|---|
| `numpy` / `pandas` | Leer y manipular los datos del CSV |
| `DecisionTreeClassifier` | El árbol de decisión de scikit-learn |
| `train_test_split` | Dividir los datos en entrenamiento y prueba |
| `confusion_matrix` / `classification_report` | Herramientas para medir qué tan bien funcionó |
| `joblib` | Guardar el modelo entrenado en un archivo `.pkl` |

> [!TIP]
> Notarás que no se importa `StandardScaler`. Eso es intencional: los Árboles de Decisión no usan matemáticas que sean sensibles a la magnitud de los números, por lo que normalizar los datos es completamente innecesario.

---

## Bloque 2: Definir el Nombre del Archivo de Salida

```python
DT_MODELO_ARCHIVO = "anemia_modelo_dt.pkl"
```

**Explicación práctica:**
Guardamos el nombre del archivo de salida en una variable al inicio del script. Esto es una buena práctica porque si algún día quisieras cambiar el nombre, solo tendrías que cambiarlo en un lugar, no en cada línea donde se usa.

---

## Bloque 3: Cargar los Datos del CSV

```python
try:
    df = pd.read_csv("anemia.csv")
except FileNotFoundError:
    print("Error: No se encontró 'anemia.csv'.")
    exit()
```

**Explicación práctica:**
Le pedimos a `pandas` que lea el archivo CSV y lo convierta en una tabla (llamada `DataFrame`). El bloque `try / except` es una red de seguridad: si el archivo no existe (por ejemplo, si ejecutas el script desde la carpeta equivocada), el programa no se rompe con un error críptico, sino que imprime un mensaje claro y se detiene ordenadamente.

---

## Bloque 4: Preprocesamiento — Limpiar y Codificar

```python
# Versión Anemia
if df.isnull().sum().any():
    df = df.fillna(df.median(numeric_only=True))

# Versión Titanic (además codifica texto a número)
training["Gender"] = training["Gender"].apply(
    lambda toLabel: 0 if toLabel == "male" else 1
)
training["Age"] = training["Age"].fillna(training["Age"].mean())
```

**Explicación práctica:**
Los modelos de Machine Learning solo entienden números. El CSV puede tener dos tipos de problemas:

1. **Datos faltantes (nulos):** Algunas filas pueden tener celdas vacías. Las rellenamos con la mediana (el valor del medio si ordenaras todos los datos) o la media (el promedio). La mediana es más resistente a valores extremos.

2. **Texto en lugar de números (solo Titanic):** La columna `Gender` tiene `"male"` o `"female"`. El modelo no puede operar con texto. La función `lambda` convierte cada valor: si es `"male"` lo transforma a `0`, si es cualquier otra cosa (femenino) lo transforma a `1`.

---

## Bloque 5: Separar Pistas de Respuestas

```python
COLUMNS = ["Gender", "Hemoglobin", "MCH", "MCHC", "MCV"]
TARGET   = "Result"

X = df[COLUMNS].values   # Las "pistas" que recibirá el modelo
y = df[TARGET].values    # La respuesta correcta (0 o 1)
```

**Explicación práctica:**
Dividimos la tabla en dos partes:
- **`X` (entrada):** Todo lo que el médico conoce del paciente antes del diagnóstico: Género, Hemoglobina, MCH, MCHC, MCV.
- **`y` (salida esperada):** La respuesta correcta ya conocida, que el modelo usará para saber si acertó durante el aprendizaje.

> [!IMPORTANT]
> El orden de las columnas en `COLUMNS` es **crítico**. Debe ser exactamente el mismo orden que se usa cuando el formulario web envía datos a `app.py`. Si cambias el orden aquí, debes cambiarlo también allá, o las predicciones serán incorrectas.

---

## Bloque 6: Dividir para Estudiar y para el Examen

```python
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.20, random_state=42, stratify=y
)
```

**Explicación práctica:**
Imaginemos que tienes 1000 fichas de pacientes. No puedes usar todas para que el modelo aprenda, porque después no sabrás si realmente aprendió o solo memorizó. Separamos:

- **80% → Entrenamiento:** El modelo estudia estos casos y descubre sus reglas.
- **20% → Prueba:** Casos que el modelo **nunca verá** hasta el examen final.

**Parámetros importantes:**
- `random_state=42`: Garantiza que la división sea siempre la misma. Si lo cambias, obtendrás una mezcla diferente de pacientes en cada grupo.
- `stratify=y`: Asegura que la proporción de enfermos vs. sanos sea similar en ambos grupos. Sin esto, podría ocurrir que la mayoría de los pacientes enfermos queden en un solo grupo por azar.

---

## Bloque 7: Construir y Entrenar el Árbol de Decisión ⭐

```python
dt_model = DecisionTreeClassifier(
    max_depth=6,
    min_samples_split=10,
    min_samples_leaf=5,
    class_weight="balanced",
    random_state=42
)

dt_model.fit(X_train, y_train)
```

**Explicación práctica:**
Este es el bloque más importante. Aquí creamos el árbol y le pasamos sus "reglas de comportamiento" antes de que empiece a aprender.

### ¿Qué hace cada parámetro?

**`max_depth=6` — Profundidad máxima:**
Limita cuántas preguntas encadenadas puede hacer el árbol. Sin este límite, el árbol podría volverse enormemente complejo y "memorizar" a cada paciente individual en lugar de aprender reglas generales. Esto se llama **sobreajuste** (*overfitting*).

```
                 [Hemoglobina < 12?]           ← Nivel 1
                /                  \
        [MCH < 25?]           [MCHC < 30?]    ← Nivel 2
        /       \              /       \
    ...         ...         ...       ...      ← Nivel 3
                                               ← ...hasta nivel 6
```

**`min_samples_split=10` — Mínimo para dividir:**
Un nodo (una pregunta) solo se crea si hay al menos 10 pacientes que llegan a ese punto. Evita que el árbol haga preguntas basadas en 2 o 3 casos, lo cual sería estadísticamente poco confiable.

**`min_samples_leaf=5` — Mínimo en una hoja:**
Cada "hoja" (conclusión final del árbol) debe tener al menos 5 pacientes que la respalden. Una conclusión basada en un solo caso sería muy frágil.

**`class_weight="balanced"` — Peso por clase:**
Imagina que tienes 900 pacientes sanos y solo 100 enfermos. Un árbol sin este parámetro podría aprender a decir "todos están sanos" y tener un 90% de precisión, pero estaría fallando completamente en detectar enfermos. Con `balanced`, el modelo le da más importancia a los casos de la clase minoritaria.

**`dt_model.fit(X_train, y_train)` — ¡A aprender!:**
El árbol analiza los datos de entrenamiento y encuentra cuáles son las mejores preguntas (y en qué valores cortarlas) para separar a los enfermos de los sanos de la forma más eficiente posible.

---

## Bloque 8: Evaluación con los Datos de Prueba

```python
y_pred = dt_model.predict(X_test)

print(confusion_matrix(y_test, y_pred))
print(classification_report(y_test, y_pred, target_names=["No anémico", "Anémico"]))
```

**Explicación práctica:**
Le mostramos al modelo el 20% de pacientes que reservamos y le pedimos sus predicciones.

`dt_model.predict()` devuelve directamente la clase (`0` o `1`), no una probabilidad como la Red Neuronal. El árbol recorre sus preguntas y llega a una hoja que dice: "Anémico" o "No anémico".

**Leyendo la Matriz de Confusión:**

```
                   Predicho: Sano    Predicho: Enfermo
Real: Sano       [  Correcto ✓    |  Falso Positivo ✗  ]
Real: Enfermo    [  Falso Neg. ✗  |  Correcto ✓        ]
```

- ✓ **Verdadero Positivo:** Dijo "Enfermo" y era enfermo.
- ✓ **Verdadero Negativo:** Dijo "Sano" y era sano.
- ✗ **Falso Positivo:** Dijo "Enfermo" pero estaba sano. (Alarma innecesaria)
- ✗ **Falso Negativo:** Dijo "Sano" pero estaba enfermo. *(El más peligroso en medicina)*

---

## Bloque 9: Importancia de Variables 🔍

```python
importances = dt_model.feature_importances_
for col, imp in sorted(zip(COLUMNS, importances), key=lambda x: -x[1]):
    bar = "█" * int(imp * 40)
    print(f"  {col:<12} {imp:.4f}  {bar}")
```

**Explicación práctica:**
Esta es una de las grandes ventajas del Árbol de Decisión sobre la Red Neuronal: **podemos preguntarle qué tan importante fue cada variable para tomar sus decisiones**.

`feature_importances_` devuelve un número entre 0 y 1 para cada columna. Un valor cercano a 1 significa que esa variable fue decisiva; un valor cercano a 0 significa que el árbol casi no la usó.

El código genera una mini barra visual en la consola como esta:
```
Hemoglobin   0.9212  ████████████████████████████████████
Gender       0.0523  ██
MCH          0.0134  
MCHC         0.0089  
MCV          0.0042  
```

Esto confirma algo que ya sabíamos médicamente: la **hemoglobina** es el indicador dominante de la anemia.

---

## Bloque 10: Guardar el Modelo

```python
joblib.dump(dt_model, "anemia_modelo_dt.pkl")
```

**Explicación práctica:**
`joblib.dump()` "congela" el árbol entrenado y lo guarda en un archivo `.pkl` (formato *Pickle* de Python). Este archivo contiene todas las preguntas y reglas de corte que el árbol descubrió durante el entrenamiento.

A diferencia de la Red Neuronal, **el Árbol de Decisión solo genera un archivo**. No necesita un scaler separado porque puede trabajar directamente con los datos sin normalizar.

```
Red Neuronal:     mimodelo.h5  +  mi_scaler.pkl   (2 archivos)
Árbol de Decisión: modelo_dt.pkl                  (1 archivo)
```

---

## Bloque 11: Predicción Individual de Ejemplo

```python
# Datos: [Gender, Hemoglobin, MCH, MCHC, MCV]
paciente_ejemplo = np.array([[1, 9.5, 22.0, 29.0, 78.0]])

pred_clase = dt_model.predict(paciente_ejemplo)[0]
pred_proba = dt_model.predict_proba(paciente_ejemplo)[0]  # [prob_no, prob_anemia]

estado = "Anémico" if pred_clase == 1 else "No anémico"
print(f"Probabilidad de anemia: {pred_proba[1]*100:.2f}%")
```

**Explicación práctica:**
Probamos el modelo con un paciente ficticio (mujer con hemoglobina de 9.5, claramente baja).

El árbol ofrece **dos métodos de predicción**:

| Método | Qué devuelve | Uso |
|---|---|---|
| `predict()` | La clase directa: `0` o `1` | Decisión binaria simple |
| `predict_proba()` | Lista con las probabilidades: `[0.12, 0.88]` | Para mostrar el porcentaje en la web |

`predict_proba()` devuelve siempre dos valores que suman 1.0:
- `[0]` → Probabilidad de que sea **No anémico**
- `[1]` → Probabilidad de que sea **Anémico**

En la web (`app.py`), usamos `predict_proba()[0][1] * 100` para obtener el porcentaje de anemia que se muestra en la barra de progreso del formulario.

---

## Resumen del Flujo Completo

```
CSV con datos reales
        │
        ▼
Preprocesamiento
 ├── Rellenar valores nulos con mediana/media
 └── Convertir texto a número (ej: "male" → 0)
        │
        ▼
Separar columnas: X (pistas) y y (respuesta)
        │
        ▼
División 80% / 20%  →  train_test_split(stratify=y)
        │
        ▼
DecisionTreeClassifier(max_depth, min_samples, class_weight)
        │
        ▼
dt_model.fit(X_train, y_train)   ← El árbol aprende las reglas
        │
        ▼
Evaluar con X_test
 ├── Matriz de Confusión
 ├── Classification Report
 └── Importancia de Variables (feature_importances_)
        │
        ▼
joblib.dump(dt_model, "modelo.pkl")  →  app.py lo carga al iniciar
```

> [!TIP]
> Si quisieras visualizar el árbol completo como un diagrama (con todas sus preguntas y ramas), puedes usar `sklearn.tree.plot_tree(dt_model, feature_names=COLUMNS)` con `matplotlib`. Es muy útil para entender exactamente qué reglas aprendió el modelo.
