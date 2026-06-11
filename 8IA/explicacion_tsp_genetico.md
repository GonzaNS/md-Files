# Explicación Detallada del Código: TSP con Algoritmo Genético

Este documento proporciona una guía detallada y estructurada para comprender el script de Python [`tsp_genetico.py`](file:///c:/Users/USUARIO/Desktop/UNTELS/CICLO%208/4.%20INTELIGENCIA%20ARTIFICIAL/SEM%2010/tarea%20huarote%20genetica/tsp_genetico.py), el cual resuelve el **Problema del Agente Viajero (Traveling Salesperson Problem - TSP)** para los **25 departamentos del Perú** utilizando un **Algoritmo Genético Generacional**.

---

## 1. ¿Qué es y qué hace el código?

El **Problema del Agente Viajero (TSP)** es un problema de optimización combinatoria clásico. Consiste en encontrar la ruta más corta posible que visite un conjunto de ciudades exactamente una vez y regrese a la ciudad de origen.

Dado que el espacio de búsqueda crece de manera factorial según el número de ciudades ($N!$), para $N = 25$ ciudades existen:
$$25! \approx 1.55 \times 10^{25} \text{ posibles rutas}$$

Hacer una búsqueda exhaustiva (fuerza bruta) tomaría millones de años. Por ello, este script implementa una heurística inspirada en la evolución biológica: un **Algoritmo Genético (AG)**. 

### ¿Qué hace el script?
1. **Define las coordenadas** geográficas reales (latitud y longitud) de los 25 departamentos del Perú.
2. **Precalcula las distancias** reales en kilómetros entre cada par de departamentos mediante la fórmula de Haversine.
3. **Genera una población inicial** de rutas aleatorias.
4. **Itera evolutivamente** aplicando operadores de selección (torneo), cruce de orden (OX1) y mutación (intercambio) para optimizar la distancia global.
5. **Aplica un criterio de parada** por estancamiento o límite de generaciones.
6. **Muestra y guarda los resultados** finales tanto de forma textual como gráfica (un mapa de la ruta óptima y la curva de convergencia).

---

## 2. Parámetros del Algoritmo Genético

El comportamiento del algoritmo se controla mediante las siguientes variables:

*   **Población Inicial (`PI = 50`)**: Número de rutas alternativas que se mantienen y evolucionan en cada generación.
*   **Generaciones Máximas (`MAX_GEN = 500`)**: Límite de iteraciones que dará el algoritmo.
*   **Probabilidad de Cruce (`PC = 0.8`)**: Probabilidad (80%) de que dos padres seleccionados combinen su información para crear descendencia.
*   **Probabilidad de Mutación (`PM = 0.01`)**: Probabilidad (1%) de que un gen (ciudad) cambie de lugar aleatoriamente en un individuo, introduciendo diversidad para evitar óptimos locales.
*   **Estancamiento (`ESTANCAMIENTO = 100`)**: Criterio de parada temprana. Si tras 100 generaciones consecutivas no se encuentra una ruta más corta, el algoritmo asume convergencia y finaliza.

---

## 3. Tecnologías y Librerías Utilizadas

El desarrollo del script se apoya en módulos estándar y externos de Python:

*   `random`: Permite realizar operaciones estocásticas como barajar las rutas iniciales (`random.shuffle`), seleccionar individuos para el torneo (`random.sample`), tomar decisiones probabilísticas (`random.random`) e intercambiar índices para mutaciones.
*   `math`: Proporciona funciones trigonométricas y operaciones aritméticas de bajo nivel (como `radians`, `sin`, `cos`, `atan2` y `sqrt`) esenciales para el cálculo de distancias sobre una esfera.
*   `matplotlib` (y su módulo `pyplot`): Utilizado para generar la interfaz visual del resultado.
*   `matplotlib.use('TkAgg')`: Especifica de manera explícita el backend gráfico de Matplotlib, garantizando compatibilidad y correcto despliegue de ventanas interactivas en entornos Windows/Desktop.

---

## 4. Análisis del Código Paso a Paso

A continuación, se describe detalladamente el funcionamiento de cada sección y función del archivo [`tsp_genetico.py`](file:///c:/Users/USUARIO/Desktop/UNTELS/CICLO%208/4.%20INTELIGENCIA%20ARTIFICIAL/SEM%2010/tarea%20huarote%20genetica/tsp_genetico.py):

### A. Estructura de Datos
*   **`DEPARTAMENTOS`**: Lista de tuplas conteniendo el nombre del departamento, su latitud y su longitud.
*   **`NOMBRES`** y **`COORDS`**: Listas derivadas para un acceso indexado rápido.
*   **`DIST`**: Matriz de distancias bidimensional prepoblada de tamaño $25 \times 25$. Calcular la distancia entre ciudades dinámicamente durante la evaluación del fitness consumiría valioso tiempo de cómputo; al precalcularla al inicio, el cálculo del fitness se reduce a una consulta directa en memoria de costo $O(1)$.

### B. Funciones Core

#### 1. Cálculo de Distancias: [`haversine`](file:///c:/Users/USUARIO/Desktop/UNTELS/CICLO%208/4.%20INTELIGENCIA%20ARTIFICIAL/SEM%2010/tarea%20huarote%20genetica/tsp_genetico.py#L68-L92)
La Tierra no es plana, por lo que la distancia euclidiana tradicional en coordenadas cartesianas daría errores significativos. La fórmula de Haversine calcula la distancia de círculo máximo entre dos puntos en una esfera a partir de sus longitudes y latitudes:
$$a = \sin^2\left(\frac{\Delta \text{lat}}{2}\right) + \cos(\text{lat}_1) \cdot \cos(\text{lat}_2) \cdot \sin^2\left(\frac{\Delta \text{lon}}{2}\right)$$
$$c = 2 \cdot \text{atan2}\left(\sqrt{a}, \sqrt{1-a}\right)$$
$$d = R \cdot c$$
*(Donde $R = 6371.0\text{ km}$ es el radio medio de la Tierra)*.

#### 2. Inicialización: [`inicializar_poblacion`](file:///c:/Users/USUARIO/Desktop/UNTELS/CICLO%208/4.%20INTELIGENCIA%20ARTIFICIAL/SEM%2010/tarea%20huarote%20genetica/tsp_genetico.py#L104-L121)
Crea `tam_poblacion` (50) cromosomas. Cada cromosoma representa una ruta única e independiente. Se genera llenando una lista con los números de `0` a `24` y desordenándola aleatoriamente con `random.shuffle`.

#### 3. Evaluación de Aptitud: [`calcular_fitness`](file:///c:/Users/USUARIO/Desktop/UNTELS/CICLO%208/4.%20INTELIGENCIA%20ARTIFICIAL/SEM%2010/tarea%20huarote%20genetica/tsp_genetico.py#L124-L140)
El fitness (o aptitud) en este problema es la distancia total recorrida. A menor distancia, mejor es el individuo. La función suma las distancias acumuladas de ir de la ciudad en la posición `i` a la ciudad `i + 1`. Al final, conecta la última ciudad con la primera (`(i + 1) % len(cromosoma)`) para cerrar el ciclo del viaje de ida y vuelta.

#### 4. Selección: [`seleccion_torneo`](file:///c:/Users/USUARIO/Desktop/UNTELS/CICLO%208/4.%20INTELIGENCIA%20ARTIFICIAL/SEM%2010/tarea%20huarote%20genetica/tsp_genetico.py#L143-L159)
Elige `k` (por defecto 3) individuos aleatorios de la población y los hace competir. El que tenga el menor fitness (distancia más corta) gana el torneo y es elegido como padre. Este método garantiza que los mejores individuos tengan mayor probabilidad de reproducirse, pero mantiene la oportunidad de que individuos mediocres transmitan sus genes, preservando la diversidad genética.

#### 5. Reproducción (Cruce): [`cruce_ox1`](file:///c:/Users/USUARIO/Desktop/UNTELS/CICLO%208/4.%20INTELIGENCIA%20ARTIFICIAL/SEM%2010/tarea%20huarote%20genetica/tsp_genetico.py#L161-L195)
En el TSP, no se puede utilizar un operador de cruce de punto único convencional porque produciría cromosomas inválidos (ciudades repetidas y omitidas). Por ello se utiliza el **Cruce de Orden 1 (OX1)** simplificado:
1. Se define un punto de corte aleatorio.
2. Para el `Hijo 1`, se copia el segmento izquierdo del `Padre 1` tal cual.
3. Los elementos restantes se rellenan en orden secuencial según aparecen en el `Padre 2`, ignorando los elementos que ya se agregaron del segmentado del `Padre 1`.
4. El proceso se repite de forma inversa para generar el `Hijo 2`.

*Ejemplo conceptual con letras:*
```text
Padre 1: [A, B, C | D, E, F]   (Punto de corte en 3)
Padre 2: [B, D, F | A, C, E]

Hijo 1 (Fase 1): [A, B, C | ?, ?, ?]
Recorrido Padre 2 para completar:
- B (ya está)
- D (no está -> agregar) -> [A, B, C, D]
- F (no está -> agregar) -> [A, B, C, D, F]
- A (ya está)
- C (ya está)
- E (no está -> agregar) -> [A, B, C, D, F, E]
Hijo 1 final:    [A, B, C, D, F, E] (Válido y sin duplicados)
```

#### 6. Mutación: [`mutar`](file:///c:/Users/USUARIO/Desktop/UNTELS/CICLO%208/4.%20INTELIGENCIA%20ARTIFICIAL/SEM%2010/tarea%20huarote%20genetica/tsp_genetico.py#L198-L213)
Aplica la **Mutación por Intercambio (Swap Mutation)**. Con una baja probabilidad (`PM = 0.01`), se seleccionan dos posiciones aleatorias de la ruta y se intercambian sus ciudades. Esto simula errores de copia que pueden resultar en rutas más eficientes.

#### 7. Bucle Principal: [`algoritmo_genetico`](file:///c:/Users/USUARIO/Desktop/UNTELS/CICLO%208/4.%20INTELIGENCIA%20ARTIFICIAL/SEM%2010/tarea%20huarote%20genetica/tsp_genetico.py#L216-L296)
Es el orquestador evolutivo:
1. Inicializa la población y evalúa el fitness inicial.
2. Bucle iterativo de generaciones:
    * Selecciona parejas mediante torneos.
    * Cruza parejas (con probabilidad `PC`) y muta los hijos (con probabilidad `PM`).
    * Actualiza la población con la nueva descendencia (estrategia generacional pura).
    * Evalúa y registra las estadísticas (mejor distancia global, distancia media de la población).
    * Comprueba si hay mejora: si el mejor global actualiza su valor, se reinicia el contador de estancamiento. Si no, se incrementa.
    * Detiene la ejecución si se alcanza `MAX_GEN` o si el estancamiento llega a `ESTANCAMIENTO` (100 generaciones sin mejora).

#### 8. Visualización de Resultados: [`graficar`](file:///c:/Users/USUARIO/Desktop/UNTELS/CICLO%208/4.%20INTELIGENCIA%20ARTIFICIAL/SEM%2010/tarea%20huarote%20genetica/tsp_genetico.py#L299-L368)
Dibuja dos gráficos en una sola ventana:
*   **Izquierda**: Gráfico lineal de la distancia (en km) a lo largo de las generaciones, mostrando tanto el récord absoluto (línea azul) como la media de la población (línea roja). Permite visualizar cómo el algoritmo converge rápidamente en las primeras generaciones y luego se estabiliza.
*   **Derecha**: Un gráfico bidimensional simulando el mapa del Perú, donde los departamentos se marcan como puntos rojos etiquetados, y la mejor ruta encontrada se dibuja conectando estos puntos de manera secuencial mediante líneas azules. El punto de inicio de la ruta se resalta con una estrella verde gigante.
*   Al final, guarda la gráfica como un archivo de alta calidad `tsp_resultados.png`.

---

## 5. Ejecución del Programa

Cuando el script se ejecuta directamente a través de `if __name__ == "__main__":`:
1. Fija una semilla aleatoria (`random.seed(42)`) para que los resultados sean reproducibles en cualquier máquina (siempre dará la misma secuencia evolutiva).
2. Llama a `algoritmo_genetico()`.
3. Imprime por consola los resultados del procesamiento paso a paso, mostrando el progreso cada 50 generaciones.
4. Muestra un resumen del recorrido óptimo indicando el orden exacto de los departamentos y la distancia total del trayecto.
5. Abre la ventana interactiva de Matplotlib y exporta la imagen de resultados `tsp_resultados.png`.
