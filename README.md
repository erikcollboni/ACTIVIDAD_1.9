# Actividad 1.9
# Optimización de Rutas Logísticas con Apache Spark

Este proyecto implementa un algoritmo de **Caminos Mínimos (Shortest Path)** distribuido utilizando **Apache Spark (PySpark)**. El objetivo es calcular la distancia mínima desde una ciudad de origen a todas las demás ciudades en una red logística modelada como un grafo dirigido y ponderado.

El algoritmo emplea un enfoque iterativo de **relajación tipo Dijkstra** (*Dijkstra-style relaxation*), adaptado para ejecutarse en paralelo mediante el paradigma MapReduce.

## Descripción del Proyecto

El problema consiste en encontrar la ruta más eficiente (menor coste/distancia) en una red de distribución. Dado que las redes logísticas reales pueden ser masivas, se utiliza Spark para procesar el grafo de manera distribuida.

El sistema modela cada ciudad como un nodo y cada carretera como una arista con peso. A través de iteraciones sucesivas, el algoritmo propaga las distancias conocidas hasta converger en la solución óptima.

---

## Arquitectura del Algoritmo: Map & Reduce

El núcleo de la solución se basa en dos funciones que transforman el RDD (Resilient Distributed Dataset) en cada iteración.

### Estado del Nodo
Cada nodo en el RDD se representa con la siguiente tupla:
` (ID_Nodo, (Lista_Vecinos, Distancia_Acumulada, Es_Activo, Historial_Ruta)) `

### 1. Fase Map: Expansión (`dijkstra_map`)
Esta función actúa como el **"Mensajero"**. Toma el estado actual de un nodo y decide si debe propagar información a sus vecinos.

* **Entrada:** El estado completo de un nodo.
* **Salida:** Una lista de mensajes (tuplas) para ser procesados.

**Mecanismos clave:**
* **Persistencia (Autoconservación):** La función siempre emite una copia del propio nodo. Esto es fundamental porque en Spark, si no re-emitimos el nodo, este desaparecería del RDD en la siguiente iteración. Se emite con el estado `Activo = False` tras ser procesado.
* **Propagación (Relajación):** Si el nodo está marcado como **Activo** (significa que su distancia mejoró en la iteración anterior), calcula la nueva distancia potencial para sus vecinos:
  $$D_{nuevo} = D_{actual} + Peso_{arista}$$
  Posteriormente, envía un "mensaje" de actualización a cada vecino.
  > *Nota:* A los vecinos se les envía una lista de adyacencia vacía `[]` para ahorrar memoria, ya que el objetivo es solo comunicar la nueva distancia propuesta.

### 2. Fase Reduce: Selección (`dijkstra_reduce`)
Esta función actúa como el **"Juez"**. Spark agrupa todos los mensajes destinados al mismo ID de nodo y ejecuta esta función para colapsarlos en un solo estado.

**Mecanismos clave:**
* **Recuperación de Topología:** Reconstruye la estructura del grafo. Dado que los mensajes de actualización llegan con listas de vecinos vacías, la función prioriza la lista que contiene datos (la del mensaje de persistencia).
* **Selección Greedy (Codiciosa):** Compara las distancias recibidas y conserva estrictamente la **menor**.
* **Activación:** Marca el nodo resultante como `Activo = True` **solo si hubo una mejora**. Esto disparará una nueva expansión en la siguiente iteración, simulando la "Active Frontier" de Dijkstra.

### 3. Convergencia
El algoritmo se ejecuta dentro de un bucle que monitoriza la cantidad de nodos activos. El proceso se detiene automáticamente cuando:
` num_active_nodes == 0 `
Esto garantiza que no se desperdicien recursos computacionales una vez hallados todos los caminos mínimos.

---
