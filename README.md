# README — Sistema de Análisis de Transacciones Bancarias Masivas

Este documento explica en detalle el código fuente `main.cpp`, sección por sección, tanto desde el punto de vista **técnico** (por qué está escrito así en C++) como **algorítmico** (qué estructura de datos se usa, su complejidad y su relación con los requisitos del proyecto). El objetivo es que sirva como guion de estudio para la sustentación con el profesor.

> Referencia cruzada con el enunciado: el PDF exige una tabla hash (búsqueda por ID) y un árbol AVL o Red-Black (orden cronológico y consultas por rango), sin usar `map`/`unordered_map`/`set` de la STL. El código cumple esto implementando **ambas estructuras desde cero**: `HashTableEncadenamiento` (hash con encadenamiento mediante listas enlazadas propias) y `AVL` (árbol binario auto-balanceado).

---

## 1. Índice de clases y su rol en el proyecto

| Clase / función | Rol en el proyecto | Sección del PDF que satisface |
|---|---|---|
| `Transaccion` | Modelo de datos de una operación bancaria | Punto 1 (campos obligatorios) |
| `Nodo` + `ListaEnlazada` | Lista enlazada simple, usada como "cubeta" (bucket) de la tabla hash | Base para "Tabla hash" (punto 1, obligatorio) |
| `HashTableEncadenamiento` | Tabla hash con resolución de colisiones por encadenamiento | Funcionalidad 3: Búsqueda por ID |
| `NodoAVL` + `AVL` | Árbol binario de búsqueda auto-balanceado (AVL) | Funcionalidades 4, 5 y 8: orden cronológico, rango de fechas, estadísticas |
| `GestorTransacciones` | Fachada que coordina la Hash y el AVL como una sola unidad lógica | Funcionalidades 1, 2, 6, 7 (carga, registro, actualización, eliminación) |
| Funciones `leer*()` | Validación de entradas del usuario por consola | Robustez del sistema (parte de la evaluación de "consistencia de datos") |
| `main()` | Menú interactivo y medición de tiempos | Funcionalidad 1 (carga automática) y sección 5 del informe (tiempos de ejecución) |

---

## 2. Cabecera del archivo: `#include` y `using namespace std`

```cpp
#include <iostream>
#include <fstream>
#include <sstream>
#include <string>
#include <tuple>
#include <cfloat>  // Para poder utilizar el minimo y maximo float posible
#include <chrono>  // Para poder medir los tiempos de ejecución
using namespace std;
```

Explicación de cada importación (para poder justificarlas ante el profesor, ya que el enunciado prohíbe librerías externas — todas estas son de la **biblioteca estándar de C++**, no estructuras de datos sustitutas):

- **`<iostream>`**: entrada/salida estándar (`cin`, `cout`). Es indispensable para el menú de consola.
- **`<fstream>`**: permite abrir y leer el archivo `.csv` (`ifstream`) para la carga masiva. Es la única forma de "leer un archivo" en C++ estándar.
- **`<sstream>`**: `stringstream` se usa para partir cada línea del CSV por comas (ver `cargarDesdeArchivo`). No reemplaza ninguna estructura de datos exigida; solo sirve para *parsear texto*, algo que el enunciado no prohíbe.
- **`<string>`**: tipo `string` de C++, usado en casi todos los campos (`idTransaccion`, `fecha`, etc.). Es un tipo básico del lenguaje, no una estructura de datos "de más alto nivel" como `map`.
- **`<tuple>`**: se usa dentro de `Nodo` para guardar el par `(id, puntero a Transaccion)` en un solo objeto (`tuple<string, Transaccion*>`). Es equivalente conceptualmente a un `struct` con dos campos; se usó `tuple` en vez de crear un `struct` adicional, pero **no es una estructura de datos de búsqueda** (no ordena, no indexa, no reemplaza al hash ni al árbol), por lo que no infringe la restricción de no usar `map`/`set`.
- **`<cfloat>`**: provee la constante `FLT_MAX` (el valor flotante más grande representable). Se usa en `generarReporteEstadistico()` para inicializar la variable `minM` (monto mínimo) con un valor "infinitamente grande", de modo que la primera transacción comparada siempre sea menor y quede registrada como el mínimo provisional. Es el equivalente de `float minM = FLT_MAX;` en vez de escribir `999999999` a mano.
- **`<chrono>`**: biblioteca estándar para medir tiempo de alta resolución (`high_resolution_clock`). Es la herramienta que exige el punto 5 del informe ("tiempos de ejecución"). Se usa en `main()` alrededor de cada operación del menú y en `mostrarTiempo()`.
- **`using namespace std;`**: evita escribir `std::` antes de cada `string`, `cout`, `vector`, etc. Es una práctica común en proyectos académicos pequeños (en proyectos grandes se evita por posibles colisiones de nombres, pero aquí no representa un riesgo real).

---

## 3. Clase `Transaccion`

```cpp
class Transaccion
{
private:
    string idTransaccion, cuentaOrigen, cliente, tipo, fecha, hora, estado;
    float monto;
public:
    Transaccion(string id, string origin, string client, string type,
                float amount, string date, string hour, string state) { ... }
    string getId() { return idTransaccion; }
    ...
    void imprimir() { ... }
};
```

### Qué representa
Es el **modelo de dominio**: cada objeto `Transaccion` corresponde exactamente a una fila del CSV (`idTransaccion, cuentaOrigen, cliente, tipo, monto, fecha, hora, estado`), cumpliendo el mínimo de campos exigido en el punto 1 del PDF.

### Detalles técnicos
- Todos los atributos son **privados** y se accede a ellos mediante *getters* (`getId()`, `getMonto()`, etc.) y un único *setter* (`setEstado()`). Esto es **encapsulamiento**: la única forma permitida de modificar el estado de una transacción es a través de `setEstado`, que es exactamente la única operación de actualización que exige el punto 6 del enunciado ("actualizar el estado de una transacción"). No existen setters para el monto, la fecha, el cliente, etc., porque el sistema no los necesita modificar tras la creación.
- El constructor recibe **por valor** (`string id`, no `string &id`) porque son parámetros pequeños/temporales usados una sola vez para inicializar los atributos; no hay necesidad de evitar la copia aquí porque solo ocurre una vez por transacción creada (no es una operación repetida millones de veces sobre la misma variable).
- `imprimir()` no retorna nada (`void`); solamente hace `cout` con el formato de tarjeta ("----- Transaccion [ID] -----"). Es la función reutilizada en **todas** las funcionalidades que muestran una transacción: búsqueda por ID, orden cronológico, rango de fechas, mayor/menor monto en estadísticas.

### Por qué NO se usa un `struct` en su lugar
Podría haberse usado un `struct` con campos públicos, pero se prefirió `class` con atributos privados para forzar que cualquier modificación pase por métodos controlados (buena práctica de POO exigida implícitamente por la rúbrica de "modularidad y claridad del código").

---

## 4. `Nodo` y `ListaEnlazada`: la base de la tabla hash

```cpp
class Nodo
{
private:
    tuple<string, Transaccion *> tupla;
    Nodo *nextNodo;
    ...
};

class ListaEnlazada
{
private:
    Nodo *head;
    ...
};
```

### Por qué existen
El enunciado exige una tabla hash **implementada por el grupo**, sin usar `unordered_map`. Toda tabla hash necesita resolver **colisiones** (dos IDs distintos que producen la misma posición). El grupo eligió la técnica de **encadenamiento (chaining)**: cada posición del arreglo de la tabla hash no almacena una sola transacción, sino una **lista enlazada** de transacciones que colisionaron en esa posición. `Nodo` es el nodo de esa lista, y `ListaEnlazada` es la lista en sí.

### Punteros (`Transaccion *`, `Nodo *`)
- `tuple<string, Transaccion *>`: se guarda un **puntero** a la transacción (no una copia del objeto completo) para que la misma instancia de `Transaccion` sea compartida entre la tabla hash y el árbol AVL. Esto es clave: **cada transacción existe una sola vez en memoria**, pero es *referenciada* desde dos estructuras distintas (hash y AVL). Si se guardaran copias, actualizar el estado de una transacción (`setEstado`) requeriría actualizarla en ambas estructuras por separado; al compartir el puntero, un solo `setEstado()` sobre el objeto es visible desde cualquiera de las dos estructuras.
- `Nodo *nextNodo`: puntero al siguiente nodo de la lista, el mecanismo clásico de una lista enlazada simple.

### Métodos y su complejidad
| Método | Qué hace | Complejidad |
|---|---|---|
| `insertar(Transaccion*)` | Recorre hasta el final de la lista y agrega ahí (inserción al final) | O(n) en el peor caso, donde n = elementos en ese *bucket* |
| `buscar(id)` / `buscarPorId(id)` | Recorre la lista comparando IDs | O(n) en el peor caso |
| `eliminar(id)` | Recorre la lista manteniendo un puntero `anterior` para "saltar" el nodo eliminado | O(n) en el peor caso |

En la práctica, si la tabla hash está bien dimensionada (ver sección 5), cada *bucket* tiene muy pocos elementos (idealmente 0 o 1), por lo que estas operaciones se comportan como **O(1) amortizado**.

### El destructor (`~ListaEnlazada`)
```cpp
~ListaEnlazada()
{
    Nodo *current = head;
    while (current != nullptr)
    {
        Nodo *next = current->getNextNodo();
        delete current->getTransaccion();
        delete current;
        current = next;
    }
}
```
Este destructor libera **dos niveles de memoria**: primero la `Transaccion*` que contiene el nodo (`delete current->getTransaccion()`), y luego el nodo mismo (`delete current`). Esto es importante porque el proyecto usa `new` manualmente en toda la aplicación (no hay `smart pointers` como `unique_ptr`), así que la responsabilidad de liberar memoria es 100% manual. Como la clase `HashTableEncadenamiento` es la "dueña última" de las transacciones (es la primera en insertarlas), su destrucción en cascada es la que efectivamente libera toda la memoria de las transacciones al final del programa.

> **Nota de diseño importante para la sustentación**: el AVL también contiene punteros `Transaccion*`, pero su destructor (`AVL::destruir`) **no** hace `delete` sobre la transacción, solo sobre el `NodoAVL`. Esto es intencional: si ambas estructuras intentaran liberar el mismo puntero `Transaccion*`, se produciría un **double free** (error grave de memoria). Por diseño, la tabla hash es la única responsable de liberar las transacciones.

---

## 5. `HashTableEncadenamiento`: la tabla hash

```cpp
class HashTableEncadenamiento
{
private:
    int size;
    ListaEnlazada *tabla;
public:
    HashTableEncadenamiento(int n) { size = n; tabla = new ListaEnlazada[size]; }
    ...
    int funcionHash(string id) { ... }
    ...
};
```

### Estructura interna
`tabla` es un **arreglo dinámico de listas enlazadas** (`ListaEnlazada tabla[size]`), creado con `new ListaEnlazada[size]`. Este es el corazón de la tabla hash: un arreglo de tamaño fijo `size`, donde cada casilla es un *bucket* (una lista enlazada) para resolver colisiones.

### La función hash: DJB2

```cpp
int funcionHash(string id)
{
    unsigned long hash = 5381;
    for (char c : id)
    {
        hash = (hash * 33) + c;
    }
    return static_cast<int>(hash % size);
}
```

- **DJB2** es un algoritmo hash clásico y muy usado (creado por Daniel J. Bernstein), elegido por ser rápido y tener buena distribución. Se inicia en `5381` (número primo elegido empíricamente por el autor original por su buen comportamiento estadístico) y en cada carácter se hace `hash = hash*33 + c`. El `33` también es una constante empírica del algoritmo original (equivalente a `hash*32 + hash + c`, que se puede optimizar a nivel de bits, aunque aquí se deja en su forma legible).
- El bucle `for (char c : id)` es un **range-based for loop** (C++11): recorre cada carácter del string sin necesidad de índices manuales (`for(int i=0;...)`), haciendo el código más limpio.
- Al final, `hash % size` comprime el número (que puede ser enorme) al rango `[0, size-1]`, que son las posiciones válidas del arreglo `tabla`.
- **Por qué IDs parecidos como `TX-000001` y `TX-000002` terminan en posiciones distintas**: como el hash multiplica progresivamente por 33 y suma el valor ASCII de cada carácter, un cambio de un solo carácter al final (`1` vs `2`, diferencia de 1 en ASCII) se propaga y termina produciendo números muy distintos tras el módulo, evitando que IDs secuenciales se agrupen en el mismo *bucket* (lo cual sí ocurriría con una función hash más ingenua, como sumar solo los códigos ASCII).

### Por qué `size = 15013`
En `main()`: `GestorTransacciones gestor(15013);`. El sistema debe soportar al menos 10,000 transacciones. Se eligió un tamaño **mayor** a 10,000 y, específicamente, **un número primo** (15013 es primo). Esto es una práctica estándar en tablas hash: usar un tamaño primo para el módulo reduce patrones de colisión que aparecerían con tamaños potencia de 2 o múltiplos de números comunes, mejorando la distribución uniforme de los datos entre los *buckets*.

### Complejidad
- **Inserción**: O(1) para calcular la posición (`funcionHash`) + O(k) para insertar en la lista de ese *bucket* (k = elementos en ese bucket). Con buena distribución, k es pequeño y constante en promedio → **O(1) amortizado**.
- **Búsqueda por ID** (`buscarPorId`): igual razonamiento → **O(1) amortizado**, que es exactamente lo que exige la Funcionalidad 3 del PDF ("buscar rápidamente por su identificador").
- **Eliminación**: igual, **O(1) amortizado**.
- **Peor caso teórico**: O(n) si todas las transacciones colisionaran en el mismo *bucket* (extremadamente improbable con DJB2 y tamaño primo bien elegido).

### Relación con el punto 3 del PDF
Esta clase es exactamente la "tabla hash, para buscar transacciones rápidamente por su identificador" exigida como estructura obligatoria.

---

## 6. `NodoAVL` y `AVL`: el árbol balanceado

Esta es la estructura más compleja del proyecto y satisface **tres** funcionalidades del PDF a la vez: orden cronológico (func. 4), consulta por rango de fechas (func. 5) y, de forma indirecta, las estadísticas (func. 8, mediante un recorrido completo).

### 6.1 La clave compuesta

```cpp
NodoAVL(Transaccion *tx)
{
    key = tx->getFecha() + "_" + tx->getHora() + "_" + tx->getId();
    ...
}
```

**Decisión de diseño clave**: en vez de ordenar por fecha (`string fecha`) directamente, se concatenan `fecha + "_" + hora + "_" + id` en un solo string, por ejemplo:
```
2026-06-01_14:35:20_TX-000001
```

Razones técnicas:
1. **Orden lexicográfico = orden cronológico**, porque las fechas están en formato `YYYY-MM-DD` y las horas en `HH:MM:SS` (ambos formatos son "ordenables como texto": los años más grandes son textualmente "mayores", igual con meses, días, horas, etc. — esto solo funciona porque los campos tienen ancho fijo con ceros a la izquierda). Al comparar dos strings de este tipo con los operadores `<` y `>` de C++ (comparación lexicográfica carácter por carácter), el resultado coincide exactamente con "cuál transacción ocurrió antes".
2. **Unicidad garantizada**: si dos transacciones ocurrieran exactamente en la misma fecha y hora, agregar el `id` al final asegura que la clave nunca se repita (evitando que el árbol trate a dos transacciones distintas como si fueran la misma clave).
3. Esto permite reutilizar un **árbol binario de búsqueda ordinario basado en comparación de strings**, sin necesitar convertir fechas a timestamps numéricos ni escribir un comparador personalizado.

### 6.2 Estructura del nodo

```cpp
class NodoAVL
{
private:
    string key;
    Transaccion *transaccion;
    int h;              // altura del subárbol
    NodoAVL *left;
    NodoAVL *right;
    ...
};
```

- `h`: la **altura** del nodo (usada para calcular el factor de equilibrio). Es la variable clásica que distingue un AVL de un BST simple.
- `left` / `right`: punteros a los hijos, como en cualquier árbol binario.
- Igual que en la tabla hash, `transaccion` es un **puntero compartido**, no una copia.

### 6.3 Cálculo de altura y factor de equilibrio

```cpp
int altura(NodoAVL *n) { if (n == nullptr) return -1; return n->getH(); }
int getFE(NodoAVL *n) { if (n == nullptr) return 0; return altura(n->getLeft()) - altura(n->getRight()); }
void updateH(NodoAVL *n) { ... n->setH(1 + max(altIzq, altDer)); }
```

- `altura(nullptr) == -1`: convención estándar en AVL (un árbol vacío tiene altura -1, una hoja tiene altura 0). Esto simplifica las fórmulas de balanceo.
- **Factor de equilibrio (FE)** = altura(izquierda) − altura(derecha). Es la métrica central del algoritmo AVL: si `|FE| > 1`, el árbol está desbalanceado en ese nodo y se necesita una rotación.
- `updateH` recalcula la altura de un nodo como `1 + max(altura_izq, altura_der)` cada vez que se inserta o elimina algo debajo de él — se llama después de cada inserción/eliminación recursiva, "subiendo" por la recursión.

### 6.4 Rotaciones

```cpp
NodoAVL *rotateRight(NodoAVL *y) { ... }
NodoAVL *rotateLeft(NodoAVL *x) { ... }
```

Son las dos rotaciones básicas del AVL. Toda rotación simple se compone reutilizando estas dos; los casos "doble rotación" (Izquierda-Derecha y Derecha-Izquierda) simplemente encadenan una rotación con la otra:

```cpp
// Caso Derecha - Izquierda
if (bf < -1 && nuevaClave < nodo->getRight()->getKey())
{
    nodo->setRight(rotateRight(nodo->getRight()));
    return rotateLeft(nodo);
}
```

Los **4 casos clásicos de desbalance AVL** (Izquierda-Izquierda, Derecha-Derecha, Izquierda-Derecha, Derecha-Izquierda) están implementados explícitamente tanto en `insertR` (inserción) como en `eliminarNodo` (eliminación), siguiendo el algoritmo estándar de los libros de estructuras de datos.

### 6.5 Inserción (`insertR`)

```cpp
NodoAVL *insertR(NodoAVL *nodo, Transaccion *transaccion, string nuevaClave)
{
    if (nodo == nullptr) return new NodoAVL(transaccion);
    if (nuevaClave < nodo->getKey())
        nodo->setLeft(insertR(nodo->getLeft(), transaccion, nuevaClave));
    else if (nuevaClave > nodo->getKey())
        nodo->setRight(insertR(nodo->getRight(), transaccion, nuevaClave));
    else
        return nodo; // clave duplicada, no se inserta
    updateH(nodo);
    int bf = getFE(nodo);
    // ... 4 casos de rotación ...
    return nodo;
}
```

**Por qué es recursivo y retorna `NodoAVL*`**: es el patrón estándar de inserción en BST/AVL. Cada llamada recursiva **reconstruye el puntero del hijo** correspondiente (`nodo->setLeft(insertR(...))`), de modo que, si una rotación cambia la raíz de un subárbol, el nodo padre automáticamente apunta al nuevo nodo raíz de ese subárbol. Esto es lo que permite que el rebalanceo se propague correctamente "subiendo" desde la hoja insertada hasta la raíz del árbol completo.

**Complejidad**: O(log n), porque el AVL garantiza que su altura nunca exceda ~1.44·log₂(n). Esta es la ganancia principal de usar AVL en vez de un BST normal: un BST desbalanceado podría degenerar a O(n) (por ejemplo, si los datos llegan ya ordenados cronológicamente, como es el caso típico del CSV, que ya viene ordenado por fecha).

### 6.6 Recorrido in-order limitado (`inorderR`) — Funcionalidad 4

```cpp
void inorderR(NodoAVL *nodo, int &contador, int limite)
{
    if (nodo == nullptr || contador >= limite) return;
    inorderR(nodo->getLeft(), contador, limite);
    if (contador < limite) { nodo->getTransaccion()->imprimir(); contador++; }
    inorderR(nodo->getRight(), contador, limite);
}
```

- Un **recorrido in-order** (izquierda → nodo → derecha) de un árbol binario de búsqueda visita los nodos en **orden ascendente de la clave**. Como la clave es `fecha_hora_id`, el recorrido in-order entrega automáticamente las transacciones **ordenadas cronológicamente**, sin necesidad de un `sort()` adicional. Esto responde exactamente a la Funcionalidad 4 del PDF.
- **`int &contador`** (paso por referencia): esta es la razón central para usar `&` aquí. Como la función es recursiva y se llama a sí misma en las dos ramas (izquierda y derecha), el contador debe ser **una sola variable compartida** a través de todas las llamadas recursivas, no una copia local por cada llamada. Si se pasara por valor (`int contador`), cada llamada recursiva tendría su propia copia independiente y el conteo nunca se acumularía correctamente entre subárboles distintos. Al pasarlo por referencia, todas las llamadas leen y modifican la **misma** variable ubicada en el `main()`/función llamante original.
- El corte temprano `contador >= limite` evita recorrer todo el árbol cuando ya se mostraron los `k` elementos pedidos (por ejemplo, 20 o 50, como sugiere el PDF), aunque en el peor caso (si `limite` es mayor al tamaño del árbol) sigue siendo O(n) por naturaleza de un recorrido in-order.

### 6.7 Consulta por rango de fechas (`consultarRangoFechasR`) — Funcionalidad 5

```cpp
void consultarRangoFechasR(NodoAVL *nodo, string desde, string hasta, int &encontrados, bool limite)
{
    if (nodo == nullptr) return;
    string fechaNodo = nodo->getKey().substr(0, 10);
    if (fechaNodo >= desde)
        consultarRangoFechasR(nodo->getLeft(), desde, hasta, encontrados, limite);
    if (fechaNodo >= desde && fechaNodo <= hasta)
    {
        if (!limite || encontrados < 30) nodo->getTransaccion()->imprimir();
        encontrados++;
    }
    if (fechaNodo <= hasta)
        consultarRangoFechasR(nodo->getRight(), desde, hasta, encontrados, limite);
}
```

Este es el algoritmo clásico de **búsqueda por rango en un BST/AVL**, y es más eficiente que recorrer todo el árbol:

- `nodo->getKey().substr(0, 10)`: como la clave completa es `"fecha_hora_id"` y la fecha siempre ocupa exactamente los primeros 10 caracteres (`YYYY-MM-DD`), `substr(0,10)` extrae solo la parte de fecha para comparar contra `desde`/`hasta`, ignorando hora e ID.
- **Poda del recorrido (pruning)**: 
  - Solo se visita el **subárbol izquierdo** si `fechaNodo >= desde` (porque si la fecha actual ya es menor que el límite inferior, todo lo que esté a la derecha del nodo actual —incluyendo el propio nodo— podría estar en rango, pero lo que está más a la izquierda seguramente seguirá siendo menor a `desde`, aunque el código conservadoramente entra igual salvo que la condición falle. En términos generales, esta condición evita bajar innecesariamente por la izquierda cuando ya se sabe que esos valores son menores al rango buscado).
  - Simétricamente, solo se visita el **subárbol derecho** si `fechaNodo <= hasta`.
  - Esto significa que el algoritmo **no recorre todo el árbol**, solo la porción relevante al rango, siendo más eficiente que O(n) en la práctica (aunque su cota superior teórica en el peor caso, si el rango cubre todos los datos, sigue siendo O(n)).
- **`int &encontrados`** (otra vez por referencia): mismo razonamiento que en `inorderR`. Es un contador compartido entre todas las llamadas recursivas para acumular el total de coincidencias, independientemente de en qué rama del árbol se encuentren.
- **Parámetro `bool limite`**: controla si se imprimen *todas* las coincidencias o solo las primeras 30 (para no saturar la consola con miles de líneas si el rango es muy amplio), mientras que el contador `encontrados` siempre sigue acumulando el conteo real total, cumpliendo el requisito del PDF de "mostrar la cantidad de transacciones encontradas y una muestra de los resultados".

### 6.8 Eliminación (`eliminarNodo`) — Funcionalidad 7

```cpp
NodoAVL *eliminarNodo(NodoAVL *nodo, string key)
{
    if (nodo == nullptr) return nullptr;
    if (key < nodo->getKey()) nodo->setLeft(eliminarNodo(nodo->getLeft(), key));
    else if (key > nodo->getKey()) nodo->setRight(eliminarNodo(nodo->getRight(), key));
    else
    {
        // caso hoja, caso 1 hijo (der), caso 1 hijo (izq), caso 2 hijos
        ...
    }
    updateH(nodo);
    int bf = getFE(nodo);
    // ... rebalanceo con los 4 casos AVL ...
    return nodo;
}
```

Sigue el algoritmo estándar de eliminación en BST, con los **4 casos** típicos:
1. **Nodo hoja** (sin hijos): se elimina directamente.
2. **Un solo hijo derecho**: el nodo se reemplaza por su hijo derecho.
3. **Un solo hijo izquierdo**: el nodo se reemplaza por su hijo izquierdo.
4. **Dos hijos**: se busca el **"mayor de los menores"** (`nodoMayorMenor`, el nodo más a la derecha del subárbol izquierdo — equivalente al predecesor in-order), se copian su clave y su transacción al nodo actual, y luego se elimina recursivamente ese nodo predecesor de su posición original (que por construcción tiene a lo sumo un hijo, cayendo en los casos 1-3).

Tras cualquier eliminación, se recalcula la altura (`updateH`) y se aplican las mismas 4 rotaciones AVL que en la inserción, para que el árbol **nunca pierda su propiedad de balance** incluso después de eliminar nodos.

**Complejidad**: O(log n), igual que la inserción, gracias al balance garantizado por AVL.

**Importante**: `eliminarNodo` **no** hace `delete` sobre la `Transaccion*` (solo sobre el `NodoAVL`), por la razón de gestión de memoria explicada en la sección 4 (evitar doble liberación, ya que la tabla hash es la responsable de eso).

### 6.9 Estadísticas generales (`calcularEstadisticasR`) — Funcionalidad 8

```cpp
void calcularEstadisticasR(NodoAVL *nodo, int &total, float &montoTotal, float &maxM, float &minM,
                            Transaccion *&mayor, Transaccion *&menor,
                            int &transferencias, int &retiros, int &pagos, int &compras, int &depositos,
                            int &aprobadas, int &pendientes, int &rechazadas, int &observadas, int &anuladas)
{
    if (nodo == nullptr) return;
    Transaccion *transaccion = nodo->getTransaccion();
    total++;
    montoTotal += transaccion->getMonto();
    if (transaccion->getMonto() > maxM) { maxM = transaccion->getMonto(); mayor = transaccion; }
    if (transaccion->getMonto() < minM) { minM = transaccion->getMonto(); menor = transaccion; }
    // ... conteos por tipo y por estado ...
    calcularEstadisticasR(nodo->getLeft(), ...);
    calcularEstadisticasR(nodo->getRight(), ...);
}
```

- Es un **recorrido completo del árbol** (no importa el orden, aquí no se usa in-order estrictamente, aunque de hecho sí visita izquierda y luego derecha) que acumula, en una sola pasada, **todos** los indicadores que pide la Funcionalidad 8 del PDF: total de transacciones, monto total, monto máximo/mínimo (y sus transacciones asociadas), y conteos por tipo y por estado.
- **Explosión de parámetros por referencia (`int &total`, `float &montoTotal`, `Transaccion *&mayor`, etc.)**: aquí es donde el uso de `&` se vuelve crítico y merece explicación detallada porque hay **16 parámetros de referencia**. Al ser una función **recursiva** que se llama dos veces por nodo (una para el hijo izquierdo, otra para el derecho), cada una de esas 16 variables debe ser la **misma variable física** en cada nivel de la recursión para que los acumulados (sumas, conteos, comparaciones de máximo/mínimo) sean correctos a través de todo el árbol. Si cualquiera de esos parámetros se pasara por valor, cada llamada recursiva operaría sobre su propia copia y, al terminar esa llamada, el cambio se perdería (el `total++` de un subárbol nunca se sumaría al total global).
- `Transaccion *&mayor`: nótese que esta es una **referencia a un puntero** (`&` aplicado sobre `Transaccion *`), no una referencia a un objeto `Transaccion`. Esto permite que la función interna reasigne *cuál puntero* apunta a la transacción de mayor/menor monto, y que ese cambio sea visible en la variable original declarada en `generarReporteEstadistico()`.
- **Complejidad**: O(n), porque no hay forma de calcular estadísticas globales (total, promedio, máximos) sin visitar cada transacción exactamente una vez. Es el único método de la clase `AVL` que es intrínsecamente O(n) por naturaleza del problema, no por una limitación del árbol.
- `generarReporteEstadistico()` es el método público que declara las 16 variables locales, las inicializa (`maxM = -1.0`, `minM = FLT_MAX`, contadores en 0, punteros en `nullptr`) y llama a `calcularEstadisticasR` pasándolas por referencia; luego imprime el reporte formateado.

---

## 7. `GestorTransacciones`: la fachada que une Hash + AVL

```cpp
class GestorTransacciones
{
private:
    HashTableEncadenamiento *tablaHash;
    AVL *arbolAVL;
public:
    ...
};
```

Esta clase implementa el **patrón Fachada (Facade)**: ninguna otra parte del código (ni `main()`) accede directamente a `HashTableEncadenamiento` o `AVL`; todo pasa por `GestorTransacciones`. Esto es importante porque **cada operación de negocio debe mantener sincronizadas ambas estructuras** (hash y árbol), y centralizar esa sincronización en un solo lugar evita bugs de inconsistencia (por ejemplo, insertar en el hash pero olvidarse de insertar en el árbol).

### 7.1 `nuevaTransaccion` — Funcionalidad 2 (registro manual)

```cpp
bool nuevaTransaccion(string id, ..., string estado)
{
    if (tablaHash->buscarPorId(id))
    {
        cout << "\nError: El ID de la transaccion " << id << " ya existe." << endl;
        return false;
    }
    Transaccion *nuevaTX = new Transaccion(id, ...);
    tablaHash->insertar(nuevaTX);
    arbolAVL->insert(nuevaTX);
    ...
    return true;
}
```

Implementa literalmente la Funcionalidad 2 del PDF: *"Antes de insertarla, debe verificarse que el idTransaccion no exista previamente"*. Nótese el flujo:
1. Verificar duplicado usando la **tabla hash** (rápido, O(1) amortizado) — no el árbol, porque buscar por ID es precisamente para qué existe la tabla hash.
2. Crear el objeto `Transaccion` en el heap (`new`).
3. Insertar el **mismo puntero** en ambas estructuras (`tablaHash->insertar` y `arbolAVL->insert`) — de nuevo, el objeto vive una sola vez en memoria, referenciado desde dos estructuras.

### 7.2 `cargarDesdeArchivo` — Funcionalidad 1 (carga masiva)

```cpp
void cargarDesdeArchivo(string nombreArchivo, char separador = ',', bool tieneCabecera = true)
{
    ifstream archivo(nombreArchivo);
    if (!archivo.is_open()) { ... return; }
    string linea;
    int contadorHash = 0, contadorAVL = 0;
    while (getline(archivo, linea))
    {
        if (tieneCabecera) { tieneCabecera = false; continue; }
        if (linea.empty()) continue;
        stringstream ss(linea);
        string id, origen, cliente, tipo, fecha, hora, estado, montoStr;
        getline(ss, id, separador);
        getline(ss, origen, separador);
        ...
        getline(ss, estado, separador);
        if (!estado.empty() && estado.back() == '\r') estado.pop_back();
        float monto = stof(montoStr);
        if (tablaHash->buscarPorId(id) == nullptr)
        {
            Transaccion *nuevaTx = new Transaccion(id, origen, cliente, tipo, monto, fecha, hora, estado);
            tablaHash->insertar(nuevaTx);
            arbolAVL->insert(nuevaTx);
            contadorHash++; contadorAVL++;
        }
    }
    archivo.close();
    ...
}
```

Detalles clave:
- **Parámetros con valores por defecto** (`char separador = ','`, `bool tieneCabecera = true`): permiten llamar a la función simplemente como `gestor.cargarDesdeArchivo("archivo.csv")` para el caso más común (CSV separado por comas con cabecera), sin tener que especificar todos los argumentos siempre. En `main()` se llama explícitamente con `,` para dejar claro el separador usado.
- **`ifstream archivo(nombreArchivo)`**: abre el archivo en modo lectura de texto. Se valida `archivo.is_open()` para manejar el caso de archivo inexistente o ruta incorrecta (robustez exigida en la rúbrica).
- **`tieneCabecera` como variable mutable dentro del bucle**: es un truco simple para "consumir" solo la primera línea del CSV (el encabezado `id,origen,cliente,...`) y luego dejar de aplicar ese filtro en las siguientes iteraciones — se reutiliza el mismo parámetro como si fuera una bandera de "primera vez".
- **`stringstream ss(linea)` + múltiples `getline(ss, campo, separador)`**: es la técnica estándar en C++ para partir ("tokenizar") una línea de texto por un delimitador, ya que C++ no tiene un `split()` nativo como Python. Cada `getline(ss, variable, ',')` extrae el siguiente fragmento de texto hasta la próxima coma.
- **Limpieza del carácter `\r`**: los archivos CSV generados o editados en sistemas Windows usan terminación de línea `\r\n`, mientras que en Linux/Mac es solo `\n`. Al leer con `getline` (que solo separa por `\n`), el `\r` puede quedar pegado al final del último campo (`estado`). El código detecta y elimina ese carácter invisible para evitar que, por ejemplo, el estado quede guardado como `"Aprobada\r"` en vez de `"Aprobada"` (lo cual rompería comparaciones como `estado == "Aprobada"` en las estadísticas).
- **Verificación de duplicados también en la carga masiva** (`tablaHash->buscarPorId(id) == nullptr`): garantiza que, si el CSV tuviera un ID repetido por error, solo se inserte la primera ocurrencia — consistencia de datos entre hash y árbol, tal como pide la rúbrica ("consistencia de los datos entre la hash y el árbol").
- **Contadores `contadorHash` y `contadorAVL`**: el PDF pide explícitamente mostrar "la cantidad de registros insertados en la hash y en el árbol" tras la carga masiva (Escenario de prueba 1). Aunque en este código siempre son iguales (porque cada inserción exitosa va a ambas estructuras a la vez), mantenerlos separados documenta claramente para el lector/evaluador que ambas estructuras están sincronizadas.

**Complejidad total de la carga masiva**: si hay `n` transacciones en el archivo, cada una realiza: 1 búsqueda hash O(1) amortizado + 1 inserción hash O(1) amortizado + 1 inserción AVL O(log n). Por lo tanto, la carga completa es **O(n log n)** — dominada por las `n` inserciones en el árbol AVL, ya que las operaciones de hash son más rápidas en promedio.

### 7.3 `eliminarTransaccion` — Funcionalidad 7, con auto-verificación

```cpp
void eliminarTransaccion(string id)
{
    Transaccion *transaccion = tablaHash->buscarPorId(id);
    if (transaccion == nullptr) { cout << "Transaccion no encontrada" << endl; return; }
    string key = transaccion->getFecha() + "_" + transaccion->getHora() + "_" + transaccion->getId();
    arbolAVL->eliminar(key);
    tablaHash->eliminar(id);
    cout << "Transaccion eliminada correctamente" << endl;
    if (tablaHash->buscarPorId(id) == nullptr)
        cout << "Verificacion correcta: La transacciones ya no existe" << endl;
    else
        cout << "ERROR: La transacciones sigue existiendo" << endl;
}
```

Puntos importantes:
- **Se busca primero en la tabla hash para reconstruir la clave del AVL**: el árbol está indexado por `fecha_hora_id`, no por `id` solo, así que para eliminar del árbol primero hay que **reconstruir esa clave compuesta** a partir de los datos de la transacción encontrada en el hash. Este es un ejemplo claro de cómo ambas estructuras se **complementan**: la hash sirve para localizar rápidamente por ID, y esa información se usa para poder operar sobre el árbol (que está indexado por otro criterio).
- **Orden de eliminación**: primero se elimina del árbol (`arbolAVL->eliminar(key)`) y **después** de la tabla hash (`tablaHash->eliminar(id)`). Esto es crítico porque `tablaHash->eliminar` es lo que hace `delete transaccion` internamente (a través de `ListaEnlazada::eliminar`), liberando la memoria del objeto. Si se eliminara primero del hash, el puntero `transaccion` (usado para construir `key`) ya habría sido liberado y el string `key` ya estaría calculado de todas formas (así que en este caso específico no causaría un error, pero el orden mostrado es la práctica correcta y segura: usar el objeto para leer sus datos antes de que cualquier estructura lo destruya).
- **Auto-verificación** (`if (tablaHash->buscarPorId(id) == nullptr) ...`): responde directamente al Escenario de prueba / Funcionalidad 7 del PDF: *"Luego debe verificarse que ya no aparezca en las consultas"*. El propio código ejecuta esa verificación automáticamente tras cada eliminación, en vez de dejarlo solo como una prueba manual separada.

### 7.4 Resto de métodos de fachada

- `ActualizarEstadoTransaccion(id, nuevoEstado)`: busca en el hash (O(1) amortizado) y llama a `setEstado()` sobre el mismo objeto compartido — como la transacción es un puntero único, **no es necesario actualizar nada en el árbol AVL**, porque el árbol no está indexado por estado y el objeto referenciado es el mismo. Esto ilustra la ventaja de compartir punteros en vez de copias.
- `buscarTransaccion(id)`: delega directamente en `tablaHash->buscarPorId(id)` e imprime o muestra error — Funcionalidad 3 pura.
- `MostrarPorCronologico(k)` / `MostrarPorRangoFechas(fechaI, fechaF, limitado)`: simples *wrappers* que delegan en los métodos correspondientes de `AVL`, añadiendo únicamente un mensaje de contexto en consola.
- `mostrarEstadisticas()`: delega en `arbolAVL->generarReporteEstadistico()`.

---

## 8. Funciones auxiliares fuera de las clases

### 8.1 `mostrarTiempo` — medición de rendimiento (sección 5 del informe)

```cpp
void mostrarTiempo(const string &operacion,
                    chrono::high_resolution_clock::time_point inicio,
                    chrono::high_resolution_clock::time_point fin)
{
    chrono::duration<double, std::milli> tiempo = fin - inicio;
    cout << "Tiempo de " << operacion << ": " << tiempo.count() << " ms" << endl;
}
```

- **`const string &operacion`**: aquí `&` se usa por **eficiencia**, no por necesidad de modificar el valor. Pasar un `string` por valor implica copiar todo su contenido cada vez que se llama a la función; pasarlo por **referencia constante** (`const string&`) evita esa copia (se pasa solo la dirección de memoria) y el `const` garantiza que la función no pueda modificar accidentalmente el string original del que la llama. Es el patrón recomendado en C++ para pasar objetos "de solo lectura" que no son tipos primitivos.
- Los parámetros `inicio` y `fin` sí se pasan **por valor** porque son `time_point`, un tipo pequeño (esencialmente un número/entero interno de `chrono`) — copiarlos no tiene costo relevante, a diferencia de un `string`.
- `chrono::duration<double, std::milli>`: crea una duración expresada en **milisegundos** como número de punto flotante (`double`), a partir de la resta de dos `time_point` (`fin - inicio`). `.count()` extrae ese número para poder imprimirlo. Este mecanismo es el que llena exactamente la columna "Tiempo de ejecución" de la tabla exigida en la sección 5 del informe (carga masiva, búsqueda por ID, inserción, consulta ordenada, consulta por rango, actualización, eliminación).
- **Uso en `main()`**: cada opción del menú envuelve la llamada al `GestorTransacciones` correspondiente entre `auto inicio = chrono::high_resolution_clock::now();` y `auto fin = ...::now();`, y luego llama a `mostrarTiempo(...)`. `high_resolution_clock` se eligió (en vez de, por ejemplo, `system_clock`) porque está diseñado específicamente para medir intervalos cortos con la mayor precisión disponible en el sistema, ideal para medir operaciones que pueden tardar microsegundos o milisegundos (como una búsqueda por hash).

### 8.2 Funciones de validación de entrada (`leerEntero`, `leerMonto`, `leerTexto`, `leerTipo`, `leerEstado`, `leerFecha`, `leerHora`, `leerId`, `esDigito`)

Todas siguen el mismo patrón defensivo: **bucle infinito (`while(true)`) que solo termina con `return` cuando la entrada es válida**, mostrando un mensaje de error y volviendo a pedir el dato en caso contrario. Esto asegura que el programa **nunca crashee ni continúe con datos corruptos** por una mala entrada del usuario, lo cual es relevante para la rúbrica ("consistencia de los datos").

Detalles técnicos específicos:

- **`leerEntero` / `leerMonto`**: usan `cin.fail()` para detectar si el usuario ingresó algo que no es un número (por ejemplo, letras), y `cin.peek() == '\n'` para asegurar que **no haya caracteres extra después del número** en la misma línea (por ejemplo, si el usuario escribe `"5abc"`, `cin >> valor` leería `5` pero dejaría `"abc"` pendiente en el buffer; el chequeo de `peek()` detecta que después del número no viene inmediatamente un salto de línea, y por tanto rechaza la entrada). Si falla, se limpia el estado de error (`cin.clear()`) y se descarta el resto del buffer hasta el siguiente `'\n'` (`while (cin.get() != '\n') {}`), para evitar que el bucle vuelva a leer la misma entrada corrupta infinitamente.
- **`leerTexto`**: usa `getline(cin >> ws, texto)` en vez de `cin >> texto`. `cin >> ws` primero descarta cualquier espacio en blanco pendiente en el buffer (incluyendo el `\n` que quedó de una lectura anterior con `cin >>`), y luego `getline` permite leer nombres de cliente con posibles espacios internos completos (aunque en el CSV se usan guiones bajos `Ana_Torres`, esta función soporta ambos casos con robustez).
- **`leerTipo` / `leerEstado`**: comparan la entrada contra una lista fija de valores válidos (`"Transferencia", "Retiro", "PagoServicio", "CompraWeb", "Deposito"` y `"Aprobada", "Pendiente", "Rechazada", "Observada", "Anulada"` respectivamente), que corresponden exactamente a los valores permitidos sugeridos por el PDF.
- **`leerFecha`**: valida formato `YYYY-MM-DD` verificando: longitud exacta (10 caracteres), guiones en las posiciones 4 y 7, que el resto de caracteres sean dígitos (`esDigito`), y rangos lógicos de mes (1-12) y día (1-31). No valida días exactos por mes (por ejemplo, no rechaza el 31 de febrero) para mantener la validación simple, algo que se puede mencionar como una limitación conocida ante el profesor.
- **`leerHora`**: análogo a `leerFecha`, pero para formato `HH:MM:SS`, validando rangos de horas (0-23), minutos y segundos (0-59).
- **`leerId`**: valida el formato exacto `TX-XXXXXX` (9 caracteres: "TX-" + 6 dígitos), coincidiendo con el formato de ID usado en el CSV de ejemplo del PDF (`TX-000001`).
- **`esDigito(char c)`**: función pequeña y reutilizada (`return c >= '0' && c <= '9';`) que verifica si un carácter es un dígito, usando comparación directa de valores ASCII/char en vez de la función estándar `isdigit()` de `<cctype>` — una decisión de implementar manualmente hasta este detalle mínimo, consistente con el espíritu del proyecto de "no usar utilidades de más alto nivel de lo necesario".

---

## 9. `main()`: el flujo completo del programa

```cpp
int main()
{
    cout << "--- Sistema de Analisis de Transacciones ---" << endl;
    GestorTransacciones gestor(15013);

    // CARGA AUTOMATICA AL INICIAR EL PROGRAMA
    auto inicio = high_resolution_clock::now();
    gestor.cargarDesdeArchivo("transacciones_masivo.csv", ',');
    auto fin = high_resolution_clock::now();
    mostrarTiempo("Carga", inicio, fin);

    int opcion = 0;
    ...
    while (opcion != 8)
    {
        // Imprime el menú
        opcion = leerEntero("Seleccione una opcion: ");
        switch (opcion)
        {
        case 1: /* registrar */ break;
        case 2: /* buscar */ break;
        case 3: /* mostrar cronológico */ break;
        case 4: /* rango de fechas */ break;
        case 5: /* actualizar estado */ break;
        case 6: /* eliminar */ break;
        case 7: /* estadísticas */ break;
        case 8: /* salir */ break;
        default: /* opción inválida */ 
        }
    }
    return 0;
}
```

### Flujo de la información (visión de alto nivel)

```
Archivo .csv (transacciones_masivo.csv)
        │
        ▼
GestorTransacciones::cargarDesdeArchivo()
        │
   ┌────┴────┐
   ▼         ▼
HashTable   AVL Tree     ← cada Transaccion* insertada en AMBAS estructuras
(por ID)    (por fecha_hora_id)
   │         │
   └────┬────┘
        ▼
   Menú interactivo (main)
   1. Registrar   → Hash.buscar (evitar duplicado) → new Transaccion → Hash.insertar + AVL.insert
   2. Buscar      → Hash.buscarPorId               → imprimir()
   3. Cronológico → AVL.mostrarCronologico (inorder)→ imprimir() x k
   4. Rango       → AVL.consultarRangoFechas        → imprimir() x resultados
   5. Actualizar  → Hash.buscarPorId → setEstado()  (mismo puntero, visible en ambas estructuras)
   6. Eliminar    → Hash.buscarPorId (leer datos) → AVL.eliminar(key) → Hash.eliminar(id) (libera memoria)
   7. Estadísticas→ AVL.generarReporteEstadistico (recorrido completo O(n))
   8. Salir       → destructores de GestorTransacciones → AVL::destruir + HashTable::~HashTableEncadenamiento
```

### Puntos a resaltar en la sustentación
1. **Carga automática al inicio** (antes de mostrar el menú): el PDF exige demostrar el funcionamiento con al menos 10,000 transacciones desde el primer momento, y medir el tiempo de esa carga — el código lo hace automáticamente apenas se ejecuta el programa, sin necesidad de una opción de menú separada.
2. **Cada opción del menú mide su propio tiempo** con `chrono`, cumpliendo exactamente la tabla de tiempos exigida en la sección 5 del informe.
3. **`switch (opcion)` con bloques `{ }` en cada `case`**: los bloques (`case 1: { ... break; }`) son necesarios porque dentro de cada `case` se declaran variables locales como `inicio`/`fin`; sin las llaves, C++ compartiría el mismo *scope* entre todos los `case` del `switch`, lo que generaría errores de redeclaración de variables entre distintos casos.
4. **Gestión de memoria al salir**: al terminar `main()`, `gestor` (una variable local, no un puntero) se destruye automáticamente al salir de *scope*, lo que dispara `~GestorTransacciones()` → `delete tablaHash` y `delete arbolAVL` → destructores en cascada de `HashTableEncadenamiento` (que libera todas las `Transaccion*`) y de `AVL` (que libera todos los `NodoAVL`, sin duplicar la liberación de las transacciones, como se explicó en la sección 4).

---

## 10. Resumen de complejidades (para la tabla del informe)

| Operación | Estructura usada | Complejidad teórica | Justificación |
|---|---|---|---|
| Carga masiva (n transacciones) | Hash + AVL | O(n log n) | n inserciones AVL a O(log n) cada una (dominante); hash es O(1) amortizado por inserción |
| Búsqueda por ID | Tabla Hash | O(1) amortizado | Acceso directo por posición hash + recorrido corto del *bucket* |
| Inserción individual | Hash + AVL | O(log n) | Dominada por la inserción balanceada en AVL; hash es O(1) amortizado |
| Consulta ordenada (cronológico, k elementos) | AVL (in-order) | O(k + log n) aprox. | Recorrido in-order con corte temprano al llegar a k elementos |
| Consulta por rango de fechas | AVL (poda de ramas) | O(log n + m) | m = elementos en el rango; se descartan subárboles fuera de rango |
| Actualización de estado | Tabla Hash | O(1) amortizado | Solo requiere localizar el puntero en el hash; el AVL no se modifica |
| Eliminación | Hash + AVL | O(log n) | Dominada por el rebalanceo AVL; hash es O(1) amortizado |
| Estadísticas generales | AVL (recorrido completo) | O(n) | Debe visitar cada transacción exactamente una vez |

---

## 11. Resumen general del proyecto

El sistema implementa un **gestor de transacciones bancarias** que combina dos estructuras de datos clásicas, ambas construidas desde cero (sin STL de alto nivel), cada una optimizada para un tipo de consulta distinto:

- Una **tabla hash con encadenamiento** (`HashTableEncadenamiento`, usando `funcionHash` con el algoritmo DJB2 y `ListaEnlazada` para resolver colisiones), que ofrece **búsqueda, inserción y eliminación por ID en tiempo casi constante (O(1) amortizado)** — ideal para las operaciones de "búsqueda por ID", "verificar duplicados al registrar" y "actualizar estado".
- Un **árbol AVL auto-balanceado** (`AVL`/`NodoAVL`, indexado por la clave compuesta `fecha_hora_id`), que garantiza **inserción y eliminación en O(log n)** y permite **recorridos en orden cronológico** y **consultas eficientes por rango de fechas** con poda de subárboles irrelevantes — cumpliendo las funcionalidades de "consulta ordenada" y "consulta por rango".

Ambas estructuras **comparten los mismos punteros `Transaccion*`** en lugar de duplicar los datos, lo que garantiza consistencia automática cuando se actualiza el estado de una transacción (un solo `setEstado()` es visible desde ambas estructuras) y evita errores de memoria como el *double free* (solo la tabla hash libera la memoria de las transacciones; el AVL solo libera sus propios nodos).

La clase `GestorTransacciones` actúa como **fachada** que sincroniza ambas estructuras en cada operación de negocio (registrar, cargar, actualizar, eliminar), y el `main()` provee un menú interactivo que además **mide el tiempo de ejecución de cada operación con `<chrono>`**, generando exactamente los datos que exige la tabla de resultados del informe técnico. Un conjunto de funciones de validación de entrada (`leer*`) garantiza que el sistema nunca reciba datos corruptos desde el teclado, reforzando la robustez y consistencia de los datos exigida en la rúbrica de evaluación.

En conjunto, el código cubre las 8 funcionalidades obligatorias del PDF (carga masiva, registro manual, búsqueda por ID, consulta cronológica, consulta por rango, actualización de estado, eliminación y estadísticas generales), respetando la restricción de no usar `map`/`set`/`unordered_map` ni librerías externas, y dejando trazabilidad clara entre cada clase/función del código y el requisito correspondiente del enunciado — lo cual debería facilitar responder con seguridad cualquier pregunta técnica del profesor sobre "por qué se hizo así" en la sustentación.
