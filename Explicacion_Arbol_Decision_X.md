# Guía Práctica: Entendiendo el Código del Árbol de Decisión

Este documento explica paso a paso cómo funciona el script de entrenamiento del **Árbol de Decisión** (como el que usamos en `Anemia_DT_Local.py` y `Titanic_DT_Local.py`). Está diseñado para que cualquier persona, incluso sin mucha experiencia en Machine Learning, pueda entender qué hace cada bloque de código.

> [!NOTE]
> **¿Qué es un Árbol de Decisión?**
> Imagínalo como un juego de "Adivina Quién". El modelo hace una serie de preguntas de "Sí o No" basadas en los datos (ej. *¿La hemoglobina es menor a 12?*). Dependiendo de las respuestas, sigue dividiendo a los pacientes hasta llegar a una conclusión final (Anémico o No Anémico).

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
Antes de construir una casa, necesitas sacar tus herramientas. Aquí le decimos a Python qué librerías vamos a usar:
* `pandas` y `numpy`: Para abrir y manipular el archivo Excel/CSV con los datos.
* `DecisionTreeClassifier`: El "cerebro" del árbol de decisión.
* `train_test_split`: Una tijera virtual para dividir nuestros datos.
* `joblib`: Una herramienta para empaquetar y guardar nuestro modelo terminado en un archivo para usarlo después en la web.

---

## Bloque 2: Cargar y Preparar los Datos

```python
df = pd.read_csv("anemia.csv")

# Rellenar espacios vacíos si los hay
df = df.fillna(df.median(numeric_only=True))

COLUMNS = ["Gender", "Hemoglobin", "MCH", "MCHC", "MCV"]
X = df[COLUMNS].values  # Las características o "síntomas"
y = df["Result"].values # La respuesta correcta (0 o 1)
```

**Explicación práctica:**
El modelo no lee mentes, necesita ejemplos del pasado para aprender. 
1. Leemos el archivo `anemia.csv`.
2. Si falta algún dato de algún paciente, lo rellenamos con el valor promedio (mediana) para que el modelo no colapse por espacios en blanco.
3. Separamos los datos en dos cajas:
   * **`X` (Las Pistas):** Edad, hemoglobina, sexo, etc.
   * **`y` (La Respuesta):** Si el paciente estaba enfermo o sano, o si sobrevivió o no.

> [!TIP]
> A diferencia de las Redes Neuronales, los Árboles de Decisión **no necesitan que escalemos o normalicemos** los datos. Ellos pueden entender igual de bien si un valor es `0.5` o `15000`.

---

## Bloque 3: Separar para Estudiar y para el Examen

```python
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.20, random_state=42, stratify=y
)
```

**Explicación práctica:**
Si le das a un estudiante las preguntas del examen antes de tiempo, sacará un 100% pero no habrá aprendido realmente. Con la IA pasa lo mismo.
Cortamos nuestros datos en dos pedazos:
* **Entrenamiento (`train` - 80%):** Los datos que el modelo usa para estudiar y encontrar patrones.
* **Prueba (`test` - 20%):** Datos que el modelo **nunca ha visto**, que usaremos al final para hacerle un examen y ver si realmente aprendió a diagnosticar.

---

## Bloque 4: Construir y Entrenar el Cerebro

```python
dt_model = DecisionTreeClassifier(
    max_depth=6,
    min_samples_split=10,
    class_weight="balanced",
    random_state=42
)

# ¡A estudiar!
dt_model.fit(X_train, y_train)
```

**Explicación práctica:**
Aquí creamos el Árbol de Decisión y le ponemos ciertas reglas de configuración:
* `max_depth=6`: Le decimos al árbol que no haga más de 6 preguntas encadenadas. Si hace demasiadas, se memoriza a los pacientes exactos (sobreajuste) en lugar de aprender reglas generales.
* `class_weight="balanced"`: Si tenemos 100 pacientes sanos y solo 10 enfermos, el modelo podría hacerse el "flojo" y decir que todos están sanos. Esto le obliga a darle la misma importancia a ambos grupos.

Finalmente, `dt_model.fit()` es el momento donde la magia ocurre: el modelo analiza el 80% de los datos y crea sus reglas internas.

---

## Bloque 5: El Examen Final (Evaluación)

```python
y_pred = dt_model.predict(X_test)

print(confusion_matrix(y_test, y_pred))
print(classification_report(y_test, y_pred))
```

**Explicación práctica:**
Le pasamos al modelo el 20% de pacientes que reservamos en el Bloque 3 y le pedimos que adivine el resultado (`predict`). 
Luego, comparamos sus respuestas con la realidad usando una **Matriz de Confusión**, que nos dice exactamente en qué se equivocó:
* ¿Cuántos enfermos detectó bien?
* ¿Cuántos sanos detectó bien?
* ¿A cuántos sanos les dijo que estaban enfermos? (Falso Positivo)
* ¿A cuántos enfermos les dijo que estaban sanos? (Falso Negativo)

---

## Bloque 6: Guardar el Trabajo

```python
joblib.dump(dt_model, "anemia_modelo_dt.pkl")
```

**Explicación práctica:**
Una vez que el modelo ya estudió y pasó el examen, no queremos que tenga que volver a estudiar cada vez que alguien entra a la página web. 
Usamos `joblib` para "congelar" y guardar su cerebro en un archivo `.pkl`. 

Luego, en nuestro backend (`app.py`), simplemente abrimos este archivo y el modelo estará listo al instante para atender a nuevos pacientes desde el formulario HTML.

> [!IMPORTANT]
> Nunca modifiques un archivo `.pkl` o `.h5` manualmente. Son archivos binarios que contienen miles de números y reglas matemáticas empaquetadas. Si necesitas cambiar el modelo, debes alterar el código en Python y volver a entrenarlo.
