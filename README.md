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
public:
    Nodo(string id, Transaccion *transaccion) { ... }
    string getId() { return get<0>(tupla); }
    Transaccion *getTransaccion() { return get<1>(tupla); }
    Nodo *getNextNodo() { return nextNodo; }
    void setNextNodo(Nodo *siguiente) { nextNodo = siguiente; }
};

class ListaEnlazada
{
private:
    Nodo *head;
public:
    ListaEnlazada() { head = nullptr; }
    ~ListaEnlazada() { ... }
    void insertar(Transaccion *transaccion) { ... }
    bool buscar(string id) { ... }
    Transaccion *buscarPorId(string id) { ... }
    void eliminar(string id) { ... }
};
```

> **Precisión importante sobre a quién pertenece cada método**: `Nodo` es una clase **puramente pasiva** — solo tiene *getters/setters* (`getId`, `getTransaccion`, `getNextNodo`, `setNextNodo`). No sabe "buscarse a sí mismo" ni recorrer nada. **Todos** los métodos de recorrido (`insertar`, `buscar`, `buscarPorId`, `eliminar`) pertenecen a `ListaEnlazada`, que es quien usa los *getters* de `Nodo` para navegar la lista con un puntero `tmp`/`actual` que avanza con `tmp = tmp->getNextNodo()`.

### Por qué existen
El enunciado exige una tabla hash **implementada por el grupo**, sin usar `unordered_map`. Toda tabla hash necesita resolver **colisiones** (dos IDs distintos que producen la misma posición). El grupo eligió la técnica de **encadenamiento (chaining)**: cada posición del arreglo de la tabla hash no almacena una sola transacción, sino una **lista enlazada** de transacciones que colisionaron en esa posición. `Nodo` es el nodo de esa lista, y `ListaEnlazada` es la lista en sí.

### Punteros (`Transaccion *`, `Nodo *`)
- `tuple<string, Transaccion *>`: se guarda un **puntero** a la transacción (no una copia del objeto completo) para que la misma instancia de `Transaccion` sea compartida entre la tabla hash y el árbol AVL. Esto es clave: **cada transacción existe una sola vez en memoria**, pero es *referenciada* desde dos estructuras distintas (hash y AVL). Si se guardaran copias, actualizar el estado de una transacción (`setEstado`) requeriría actualizarla en ambas estructuras por separado; al compartir el puntero, un solo `setEstado()` sobre el objeto es visible desde cualquiera de las dos estructuras.
- `Nodo *nextNodo`: puntero al siguiente nodo de la lista, el mecanismo clásico de una lista enlazada simple.

### Métodos de `ListaEnlazada` y su complejidad
| Método | Qué hace | Complejidad |
|---|---|---|
| `insertar(Transaccion*)` | Recorre hasta el final de la lista y agrega ahí (inserción al final) | O(n) en el peor caso, donde n = elementos en ese *bucket* |
| `buscar(id)` | Recorre la lista comparando IDs, devuelve `bool` | O(n) en el peor caso |
| `buscarPorId(id)` | Recorre la lista comparando IDs, devuelve `Transaccion*` (o `nullptr`) | O(n) en el peor caso |
| `eliminar(id)` | Recorre la lista manteniendo un puntero `anterior` para "saltar" el nodo eliminado | O(n) en el peor caso |

En la práctica, si la tabla hash está bien dimensionada (ver sección 5), cada *bucket* tiene muy pocos elementos (idealmente 0 o 1), por lo que estas operaciones se comportan como **O(1) amortizado**.

### `buscar` vs. `buscarPorId`: código duplicado y una observación de código muerto

Comparando ambos métodos:

```cpp
bool buscar(string id)
{
    Nodo *tmp = head;
    while (tmp != nullptr)
    {
        if (tmp->getId() == id) return true;
        tmp = tmp->getNextNodo();
    }
    return false;
}
Transaccion *buscarPorId(string id)
{
    Nodo *tmp = head;
    while (tmp != nullptr)
    {
        if (tmp->getId() == id) return tmp->getTransaccion();
        tmp = tmp->getNextNodo();
    }
    return nullptr;
}
```

Ambos recorren la lista con el **mismo bucle**, solo que uno devuelve `true`/`false` y el otro el puntero (o `nullptr`). `buscar` podría reescribirse para **reutilizar** `buscarPorId` en vez de duplicar el recorrido:

```cpp
bool buscar(string id)
{
    return buscarPorId(id) != nullptr;   // o, de forma más compacta: return buscarPorId(id);
}
```

Esto funciona gracias a la **conversión implícita puntero → bool** de C++: cualquier puntero distinto de `nullptr` se evalúa como `true`, y `nullptr` como `false` — el mismo mecanismo que permite escribir `if (puntero)` en vez de `if (puntero != nullptr)`.

**Dato adicional para la sustentación**: revisando todo `main.cpp`, **ni `HashTableEncadenamiento::buscar` ni `ListaEnlazada::buscar` son llamados jamás** en ninguna parte del programa — todas las operaciones (`nuevaTransaccion`, `cargarDesdeArchivo`, `ActualizarEstadoTransaccion`, `eliminarTransaccion`, `buscarTransaccion`) usan exclusivamente `buscarPorId`. Es decir, hoy `buscar()` es **código muerto** en ambas clases. Esto se puede mencionar como una observación de limpieza de código en la sección de Discusión/Conclusiones del informe: o se elimina por no usarse, o se refactoriza para reutilizar `buscarPorId` (evitando mantener dos bucles idénticos sincronizados manualmente, principio DRY — *Don't Repeat Yourself*).

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

### `buscar` y `buscarPorId` a nivel de `HashTableEncadenamiento`

```cpp
bool buscar(string id)
{
    int posicion = funcionHash(id);
    return tabla[posicion].buscar(id);        // delega en ListaEnlazada::buscar
}
Transaccion *buscarPorId(string id)
{
    int posicion = funcionHash(id);
    return tabla[posicion].buscarPorId(id);   // delega en ListaEnlazada::buscarPorId
}
```

Se repite exactamente el mismo patrón de la sección 4: `buscar` podría reescribirse para reutilizar `buscarPorId` en vez de delegar en `ListaEnlazada::buscar` (que internamente duplica el mismo recorrido):

```cpp
bool buscar(string id)
{
    int posicion = funcionHash(id);
    return tabla[posicion].buscarPorId(id) != nullptr;
}
```

Como ya se señaló: **ni `HashTableEncadenamiento::buscar` ni `ListaEnlazada::buscar` se llaman jamás** en `main.cpp` — todas las operaciones del programa (`nuevaTransaccion`, `cargarDesdeArchivo`, `ActualizarEstadoTransaccion`, `eliminarTransaccion`, `buscarTransaccion`) usan siempre `buscarPorId`. Es una observación de "código muerto" válida para mencionar en la Discusión del informe.

### Relación con el punto 3 del PDF
Esta clase es exactamente la "tabla hash, para buscar transacciones rápidamente por su identificador" exigida como estructura obligatoria.

---

## 6. `NodoAVL` y `AVL`: el árbol balanceado

Esta es la estructura más compleja del proyecto y satisface **tres** funcionalidades del PDF a la vez: orden cronológico (func. 4), consulta por rango de fechas (func. 5) y, de forma indirecta, las estadísticas (func. 8, mediante un recorrido completo).

### 6.1 `NodoAVL`: atributos privados

```cpp
class NodoAVL
{
private:
    string key;               // "fecha_hora_id" para orden cronologico
    Transaccion *transaccion; // puntero compartido con la tabla hash
    int h;                    // altura del subárbol que cuelga de este nodo
    NodoAVL *left;
    NodoAVL *right;
```

- **`key`**: no es la fecha sola — es la concatenación `fecha + "_" + hora + "_" + id` (ver 6.1.1). Es la clave que el árbol usa para comparar y ordenar.
- **`transaccion`**: puntero, no copia — el mismo objeto que también referencia la tabla hash.
- **`h`**: la altura de **este nodo dentro de su propio subárbol** (no la profundidad desde la raíz). Es la variable que hace posible calcular el factor de equilibrio sin recorrer todo el árbol cada vez.
- **`left`, `right`**: punteros a los hijos, como cualquier árbol binario.

### 6.1.1 La clave compuesta (constructor de `NodoAVL`)

```cpp
NodoAVL(Transaccion *tx)
{
    key = tx->getFecha() + "_" + tx->getHora() + "_" + tx->getId();
    transaccion = tx;
    h = 0;
    left = nullptr;
    right = nullptr;
}
```

Un nodo nuevo siempre nace como **hoja**: `h = 0` (una hoja tiene altura 0, con la convención de que un árbol vacío tiene altura -1) y ambos hijos en `nullptr`.

**Decisión de diseño clave**: en vez de ordenar por fecha (`string fecha`) directamente, se concatenan `fecha + "_" + hora + "_" + id` en un solo string, por ejemplo:
```
2026-06-01_14:35:20_TX-000001
```

Razones técnicas:
1. **Orden lexicográfico = orden cronológico**, porque las fechas están en formato `YYYY-MM-DD` y las horas en `HH:MM:SS` (ambos formatos son "ordenables como texto": los años más grandes son textualmente "mayores", igual con meses, días, horas, etc. — esto solo funciona porque los campos tienen ancho fijo con ceros a la izquierda). Al comparar dos strings de este tipo con los operadores `<` y `>` de C++ (comparación lexicográfica carácter por carácter), el resultado coincide exactamente con "cuál transacción ocurrió antes".
2. **Unicidad garantizada**: si dos transacciones ocurrieran exactamente en la misma fecha y hora, agregar el `id` al final asegura que la clave nunca se repita.
3. Esto permite reutilizar un **árbol binario de búsqueda ordinario basado en comparación de strings**, sin necesitar convertir fechas a timestamps numéricos ni escribir un comparador personalizado.

### 6.1.2 Getters/setters de `NodoAVL`

Son *accessors* simples (`getKey`, `getTransaccion`, `getH`/`setH`, `getLeft`/`setLeft`, `getRight`/`setRight`), salvo dos que merecen atención especial:

```cpp
// Para la eliminacion
void setKey(string nuevaKey) { key = nuevaKey; }
void setTransaccion(Transaccion *nuevaTransaccion) { transaccion = nuevaTransaccion; }
```

Estos dos *setters* existen **exclusivamente** para el caso de eliminación con dos hijos (sección 6.9), donde en vez de mover punteros físicamente, se **copia** la clave y el puntero de transacción del nodo predecesor hacia el nodo que se quiere eliminar. No se usan en ningún otro lugar del código.

### 6.2 `AVL`: el único atributo privado

```cpp
class AVL
{
private:
    NodoAVL *root;
```

Solo hay **un atributo**: el puntero a la raíz. Todo lo demás es comportamiento (métodos). Cuando el árbol está vacío, `root == nullptr`.

### 6.3 Cálculo de altura y factor de equilibrio

```cpp
int altura(NodoAVL *n)
{
    if (n == nullptr) return -1;
    return n->getH();
}

int getFE(NodoAVL *n)
{
    if (n == nullptr) return 0;
    return altura(n->getLeft()) - altura(n->getRight());
}

void updateH(NodoAVL *n)
{
    if (n != nullptr)
    {
        int altIzq = altura(n->getLeft());
        int altDer = altura(n->getRight());
        n->setH(1 + max(altIzq, altDer));
    }
}
```

- **`altura(n)`**: envoltorio seguro sobre el atributo `h`. `altura(nullptr) == -1` es la convención estándar en AVL (un árbol vacío tiene altura -1, una hoja tiene altura 0), lo que simplifica las fórmulas de balanceo.
- **`getFE(n)`** (factor de equilibrio) = `altura(izquierda) - altura(derecha)`. Es la métrica central del algoritmo AVL: `FE > 1` desbalance a la izquierda, `FE < -1` desbalance a la derecha, `-1 ≤ FE ≤ 1` balanceado.
- **`updateH(n)`**: recalcula la altura de un nodo **a partir de sus hijos** ("1 + la altura del hijo más alto"). Se llama **después** de cualquier cambio estructural debajo de ese nodo, propagando la información de altura correctamente "subiendo" por la recursión.

### 6.4 Rotaciones: `rotateRight` y `rotateLeft`

```cpp
NodoAVL *rotateRight(NodoAVL *y)
{
    NodoAVL *x = y->getLeft();
    NodoAVL *T2 = x->getRight();

    x->setRight(y);
    y->setLeft(T2);

    updateH(y);
    updateH(x);

    return x;
}
```

Antes:
```
        y
       / \
      x   T3
     / \
    T1  T2
```
Después de `rotateRight(y)`:
```
      x
     / \
    T1  y
       / \
      T2  T3
```

`x` (el hijo izquierdo de `y`) sube a ocupar el lugar de `y`; `T2` (antiguo hijo derecho de `x`) pasa a ser el hijo izquierdo de `y`; `y` pasa a ser el hijo derecho de `x`. Se actualizan las alturas de `y` **primero** y de `x` **después** (el orden importa: `x` depende de la altura ya actualizada de `y`). Se retorna `x`, la **nueva raíz de este subárbol**.

`rotateLeft` es el espejo exacto. Los casos de **doble rotación** (Izquierda-Derecha, Derecha-Izquierda) encadenan una rotación con la otra:

```cpp
// Caso Derecha - Izquierda
if (bf < -1 && nuevaClave < nodo->getRight()->getKey())
{
    nodo->setRight(rotateRight(nodo->getRight()));
    return rotateLeft(nodo);
}
```

Los **4 casos clásicos de desbalance AVL** (Izq-Izq, Der-Der, Izq-Der, Der-Izq) están implementados tanto en `insertR` como en `eliminarNodo`, con condiciones de decisión distintas entre ambos (ver 6.9).

### 6.5 Inserción recursiva (`insertR`)

```cpp
NodoAVL *insertR(NodoAVL *nodo, Transaccion *transaccion, string nuevaClave)
{
    if (nodo == nullptr)
        return new NodoAVL(transaccion);

    if (nuevaClave < nodo->getKey())
        nodo->setLeft(insertR(nodo->getLeft(), transaccion, nuevaClave));
    else if (nuevaClave > nodo->getKey())
        nodo->setRight(insertR(nodo->getRight(), transaccion, nuevaClave));
    else
        return nodo; // clave duplicada, no se inserta

    updateH(nodo);
    int bf = getFE(nodo);

    if (bf > 1 && nuevaClave < nodo->getLeft()->getKey())        // Izq-Izq
        return rotateRight(nodo);
    if (bf < -1 && nuevaClave > nodo->getRight()->getKey())      // Der-Der
        return rotateLeft(nodo);
    if (bf < -1 && nuevaClave < nodo->getRight()->getKey())      // Der-Izq
    {
        nodo->setRight(rotateRight(nodo->getRight()));
        return rotateLeft(nodo);
    }
    if (bf > 1 && nuevaClave > nodo->getLeft()->getKey())        // Izq-Der
    {
        nodo->setLeft(rotateLeft(nodo->getLeft()));
        return rotateRight(nodo);
    }
    return nodo;
}
```

1. **Caso base**: llegar a `nullptr` es donde debe ir la nueva transacción → se crea el `NodoAVL` y se retorna.
2. **Descenso recursivo**: se compara `nuevaClave` contra la clave del nodo actual (comportamiento estándar de un BST).
3. **Por qué `nodo->setLeft(insertR(nodo->getLeft(), ...))` y no solo `insertR(nodo->getLeft(), ...)`**: `insertR` **reconstruye y retorna la nueva raíz del subárbol** tras insertar y posiblemente rebalancear. La variable local `nodo` dentro de la llamada recursiva es solo una copia del puntero recibido; la única forma en que el llamador se entera de "cuál es ahora la raíz de ese subárbol" es a través del **valor de retorno**. Sin la asignación, si en algún nivel inferior ocurre una rotación, el padre seguiría apuntando al nodo *viejo*, dejando el árbol estructuralmente inconsistente.
4. **Clave duplicada** (`else return nodo`): no se inserta nada (en la práctica casi nunca ocurre, porque el `id` único forma parte de la clave).
5. **Actualizar altura y factor de equilibrio**, ya con el nuevo elemento insertado en algún descendiente.
6. **Los 4 casos de rotación**, decididos comparando `bf` y la posición de `nuevaClave` respecto al hijo relevante — en inserción se sabe exactamente por dónde se insertó el nuevo elemento, así que se puede decidir el tipo de rotación comparando esa clave directamente.

**Complejidad**: Θ(log n), porque el AVL garantiza que su altura nunca exceda ~1.44·log₂(n) — a diferencia de un BST sin balanceo, que degeneraría a O(n) si los datos llegan ya ordenados (como el CSV, que viene ordenado por fecha).

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

- Un **recorrido in-order** (izquierda → nodo → derecha) visita los nodos en **orden ascendente de la clave**. Como la clave es `fecha_hora_id`, el recorrido in-order entrega automáticamente las transacciones **ordenadas cronológicamente**, sin necesidad de un `sort()` adicional.
- **`int &contador`** (por referencia): al ser recursiva en dos ramas, el contador debe ser **una sola variable compartida** en todas las llamadas. Si se pasara por valor, cada llamada tendría su propia copia y el conteo nunca se acumularía correctamente entre subárboles.
- El corte temprano `contador >= limite` evita recorrer todo el árbol una vez alcanzados los `k` elementos pedidos.

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

- `nodo->getKey().substr(0, 10)`: extrae solo la parte de fecha (los primeros 10 caracteres de la clave) para comparar contra `desde`/`hasta`, ignorando hora e ID.
- **Los tres `if` son independientes, no un `if/else if`** — esto es clave: un nodo que **sí está dentro del rango** cumple las **tres condiciones simultáneamente** (`>= desde`, el AND completo, y `<= hasta`), por lo que las tres acciones ocurren para ese nodo: recorrer la izquierda, imprimir/contar, y recorrer la derecha. El orden en que aparecen estos tres `if` en el código **no afecta el resultado**, porque no dependen entre sí ni comparten un `else`. Si en cambio fuera un `if/else if/else` (mutuamente excluyentes), un nodo dentro del rango entraría solo en la primera rama que le calzara y nunca llegaría a ejecutar las siguientes — eso sí sería un bug real, perdiendo resultados y dejando de explorar la mitad derecha del árbol.
- **Poda del recorrido (*pruning*)**: solo se visita el subárbol izquierdo si `fechaNodo >= desde` (si el nodo actual ya es menor que `desde`, todo su subárbol izquierdo también lo sería); simétricamente para la derecha con `<= hasta`. El algoritmo **no recorre todo el árbol**, solo la porción relevante al rango — más eficiente que O(n) en la práctica, aunque en el peor caso (rango que cubre todo) se degrada a Θ(n).
- **`int &encontrados`** (por referencia): mismo razonamiento que en `inorderR`, contador compartido entre todas las llamadas recursivas.
- **`bool limite`**: controla si se imprimen *todas* las coincidencias o solo las primeras 30, mientras `encontrados` siempre acumula el conteo real total.

### 6.8 `nodoMayorMenor` — encontrar el predecesor in-order

```cpp
NodoAVL *nodoMayorMenor(NodoAVL *nodo)
{
    NodoAVL *actual = nodo;
    while (actual->getRight() != nullptr)
        actual = actual->getRight();
    return actual;
}
```

Dado un subárbol (típicamente el hijo izquierdo de un nodo a eliminar), encuentra el nodo con la clave más grande de ese subárbol (el "mayor de los menores", equivalente al predecesor in-order), yendo siempre a la derecha hasta que no haya más hijo derecho. Es **iterativo**, no recursivo — no hace falta reconstruir punteros en el camino, solo "caminar" hasta el final.

### 6.9 Eliminación recursiva (`eliminarNodo`) — Funcionalidad 7

```cpp
NodoAVL *eliminarNodo(NodoAVL *nodo, string key)
{
    if (nodo == nullptr) return nullptr;

    if (key < nodo->getKey())
        nodo->setLeft(eliminarNodo(nodo->getLeft(), key));
    else if (key > nodo->getKey())
        nodo->setRight(eliminarNodo(nodo->getRight(), key));
    else
    {
        // Nodo Hoja
        if (nodo->getLeft() == nullptr && nodo->getRight() == nullptr)
        {
            delete nodo;
            return nullptr;
        }
        // Un Hijo Derecho
        if (nodo->getLeft() == nullptr)
        {
            NodoAVL *tmp = nodo->getRight();
            delete nodo;
            return tmp;
        }
        // Un Hijo Izquierdo
        if (nodo->getRight() == nullptr)
        {
            NodoAVL *tmp = nodo->getLeft();
            delete nodo;
            return tmp;
        }
        // Dos Hijos
        NodoAVL *tmp = nodoMayorMenor(nodo->getLeft());
        nodo->setKey(tmp->getKey());
        nodo->setTransaccion(tmp->getTransaccion());
        nodo->setLeft(eliminarNodo(nodo->getLeft(), tmp->getKey()));
    }

    updateH(nodo);
    int bf = getFE(nodo);
    if (bf > 1 && getFE(nodo->getLeft()) >= 0)   return rotateRight(nodo);
    if (bf < -1 && getFE(nodo->getRight()) <= 0) return rotateLeft(nodo);
    if (bf < -1 && getFE(nodo->getRight()) > 0)  { nodo->setRight(rotateRight(nodo->getRight())); return rotateLeft(nodo); }
    if (bf > 1 && getFE(nodo->getLeft()) < 0)    { nodo->setLeft(rotateLeft(nodo->getLeft())); return rotateRight(nodo); }
    return nodo;
}
```

**Descenso**: igual que en `insertR`, comparando la clave a eliminar y reasignando el resultado al hijo correspondiente.

**Los 4 sub-casos cuando se encuentra el nodo**:
1. **Nodo hoja**: se libera (`delete nodo`) y se retorna `nullptr`.
2. **Solo hijo derecho**: se guarda el puntero al hijo, se libera el nodo actual, se retorna ese hijo.
3. **Solo hijo izquierdo**: simétrico.
4. **Dos hijos**: se busca el predecesor in-order (`nodoMayorMenor` sobre el hijo izquierdo), se **copian** su `key` y su `transaccion` hacia el nodo a eliminar (con los *setters* especiales de `NodoAVL`), y se elimina recursivamente al predecesor de su posición original — como ese predecesor nunca tiene hijo derecho, esa eliminación recursiva siempre cae en el caso 1 o 2 (sin riesgo de recursión infinita).

**Por qué es imprescindible `nodo->setLeft(eliminarNodo(nodo->getLeft(), tmp->getKey()))` y no solo `eliminarNodo(nodo->getLeft(), tmp->getKey())`**: si se omitiera la asignación, en el caso más común (el predecesor `tmp` es una hoja) ocurriría lo siguiente: `eliminarNodo` encuentra `tmp`, hace `delete nodo` (liberando físicamente esa memoria) y retorna `nullptr` — si ese `nullptr` se descarta, `nodo->left` **seguiría apuntando a memoria ya liberada**: un **puntero colgante (dangling pointer)**. Cualquier operación futura que intente leer `nodo->getLeft()->getKey()` accedería a memoria inválida → comportamiento indefinido (puede corromper datos o causar un *segmentation fault*, posiblemente mucho después y en un lugar sin relación aparente con la eliminación real). Incluso si `tmp` no fuera hoja, cualquier rotación en niveles inferiores cambia la raíz de ese subárbol, y ese cambio **solo se comunica hacia arriba a través del `return`** — sin la asignación, el árbol quedaría con el balance AVL perdido silenciosamente.

**Rebalanceo tras eliminar — condiciones distintas a `insertR`**: en inserción se compara contra `nuevaClave` (se sabe por dónde se insertó); en eliminación no hay una "clave nueva" que guíe la decisión, así que se consulta el **factor de equilibrio del hijo relevante** (`getFE(nodo->getLeft())` / `getFE(nodo->getRight())`) para decidir si la rotación debe ser simple o doble. Es la diferencia clásica entre rebalanceo de inserción vs. eliminación en un AVL.

**Complejidad**: Θ(log n), igual que la inserción.

**Importante**: `eliminarNodo` **no** hace `delete` sobre la `Transaccion*` (solo sobre el `NodoAVL`) — evita el *double free*, ya que la tabla hash es la responsable de liberar las transacciones (sección 4).

### 6.10 Estadísticas generales (`calcularEstadisticasR`) — Funcionalidad 8

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

Es un **recorrido completo del árbol** (visita izquierda y derecha sin aprovechar el orden — para estadísticas globales no hay atajo que dependa de que el árbol esté ordenado) que acumula en una sola pasada todos los indicadores de la Funcionalidad 8.

#### 6.10.1 Por qué 16 parámetros por referencia (`int &`, `float &`)

La razón de fondo es la **recursión de árbol binario en dos ramas**: la función se llama a sí misma dos veces por nodo (izquierda y derecha). Con n = 10,000 transacciones hay ~10,000 marcos de llamada distintos, cada uno con sus propias variables locales. Los 16 acumuladores deben ser **una sola variable física** compartida por todos esos marcos para que las sumas y conteos se acumulen correctamente. Al declarar `int &total`, el parámetro es un **alias** de la variable original declarada en `generarReporteEstadistico()` — cualquier `total++` en cualquier nivel modifica esa única variable.

**Qué pasaría sin `&` (por valor)**: cada llamada recursiva operaría sobre su propia copia local; al hacer `total++` solo se incrementa esa copia, y como la función es `void` (no hay `return valor;` que la rescate), esa copia se destruye al salir. La variable original en `generarReporteEstadistico()` **nunca se modificaría** — el reporte final mostraría `Total: 0`, `Monto total: 0`, etc., aunque el árbol tuviera miles de transacciones reales.

#### 6.10.2 Por qué `Transaccion *&mayor` (referencia a un puntero, no solo un puntero)

`mayor` ya es un puntero; lo que cambia no es el **contenido** del objeto apuntado, sino **a cuál objeto apunta** la variable (`mayor = transaccion;` reasigna el puntero mismo). Si `mayor` fuera un puntero **por valor**, esa reasignación solo afectaría la copia local — el `mayor` real en `generarReporteEstadistico()` seguiría en `nullptr` para siempre. Se necesita una referencia **al puntero mismo**: `Transaccion *&mayor` se lee "`mayor` es una referencia a un puntero a `Transaccion`" — un alias de la variable-puntero completa.

| Declaración | Qué permite modificar de forma visible para el llamador |
|---|---|
| `Transaccion *mayor` (por valor) | El contenido apuntado, pero **no** a cuál objeto apunta `mayor` |
| `Transaccion *&mayor` (referencia al puntero) | **Ambas cosas** — y sobre todo, a cuál objeto apunta la variable, que es justo lo que necesita este algoritmo |

#### 6.10.3 Complejidad

**Θ(n)** — no hay forma de calcular estadísticas globales sin visitar cada transacción exactamente una vez. Es el único método de `AVL` cuya complejidad no depende de la altura del árbol, sino directamente del número total de nodos.

`generarReporteEstadistico()` es el método público que declara las 16 variables locales, las inicializa (`maxM = -1.0`, `minM = FLT_MAX`, contadores en 0, punteros en `nullptr`), llama a `calcularEstadisticasR` pasándolas por referencia, y luego imprime el reporte formateado.

#### 6.10.4 Alternativa de diseño: agrupar los 16 parámetros en un `struct`

Los 16 parámetros por referencia se pueden reemplazar por **un solo parámetro por referencia** a un `struct` acumulador, con el mismo comportamiento y mucho menos riesgo de error al declarar la firma:

```cpp
struct EstadisticasAcumulador
{
    int total = 0;
    float montoTotal = 0.0f;
    float maxM = -1.0f;
    float minM = FLT_MAX;
    Transaccion *mayor = nullptr;
    Transaccion *menor = nullptr;
    int transferencias = 0, retiros = 0, pagos = 0, compras = 0, depositos = 0;
    int aprobadas = 0, pendientes = 0, rechazadas = 0, observadas = 0, anuladas = 0;
};

void calcularEstadisticasR(NodoAVL *nodo, EstadisticasAcumulador &acc)
{
    if (nodo == nullptr) return;
    Transaccion *tx = nodo->getTransaccion();
    acc.total++;
    acc.montoTotal += tx->getMonto();
    if (tx->getMonto() > acc.maxM) { acc.maxM = tx->getMonto(); acc.mayor = tx; }
    if (tx->getMonto() < acc.minM) { acc.minM = tx->getMonto(); acc.menor = tx; }
    if (tx->getTipo() == "Transferencia") acc.transferencias++;
    if (tx->getTipo() == "Retiro")        acc.retiros++;
    if (tx->getTipo() == "PagoServicio")  acc.pagos++;
    if (tx->getTipo() == "CompraWeb")     acc.compras++;
    if (tx->getTipo() == "Depósito")      acc.depositos++;
    if (tx->getEstado() == "Aprobada")   acc.aprobadas++;
    if (tx->getEstado() == "Pendiente")  acc.pendientes++;
    if (tx->getEstado() == "Rechazada")  acc.rechazadas++;
    if (tx->getEstado() == "Observada")  acc.observadas++;
    if (tx->getEstado() == "Anulada")    acc.anuladas++;
    calcularEstadisticasR(nodo->getLeft(), acc);
    calcularEstadisticasR(nodo->getRight(), acc);
}

void generarReporteEstadistico()
{
    EstadisticasAcumulador acc;   // todos los campos ya nacen inicializados por el struct
    calcularEstadisticasR(root, acc);
    if (acc.total == 0) { cout << "No hay datos para mostrar." << endl; return; }
    // ... imprimir usando acc.total, acc.montoTotal, acc.mayor, etc. ...
}
```

`EstadisticasAcumulador &acc` es una referencia, así que `acc` es un alias del **mismo struct físico** en todos los niveles de la recursión. Los inicializadores de miembro por defecto del struct (`int total = 0;`, `float minM = FLT_MAX;`) hacen que `EstadisticasAcumulador acc;` ya deje todos los campos correctamente inicializados, eliminando las 16 líneas de inicialización manual que existían antes.

**Otras alternativas posibles** (menos recomendables): (a) que cada llamada recursiva **retorne** su propio struct y el nodo padre los combine con los de sus hijos (estilo funcional, sin `&`, pero con copias adicionales por nivel); (b) convertir los acumuladores en **atributos de `AVL`** (evita parámetros, pero acopla la función a un estado oculto que habría que resetear manualmente); (c) variables globales (descartable). La opción del `struct` por referencia es la que ofrece la mejor relación entre preservar el comportamiento actual y reducir el riesgo de errores — una mejora válida para mencionar en la sección de Conclusiones del informe.

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

**Detalle técnico sobre `if (tablaHash->buscarPorId(id))`**: `buscarPorId` no devuelve un `bool` — devuelve un **puntero** (`Transaccion *`): la transacción encontrada, o `nullptr` si no existe. Que esto funcione dentro de un `if` se debe a la **conversión implícita puntero → bool** de C++: cualquier puntero distinto de `nullptr` se evalúa como `true`, y `nullptr` se evalúa como `false`. La línea es exactamente equivalente, en significado, a `if (tablaHash->buscarPorId(id) != nullptr)`. Existe también un método `bool buscar(string id)` en `HashTableEncadenamiento` que hace lo mismo de forma más explícita (ver sección 5) — el grupo probablemente usó `buscarPorId` de forma uniforme en toda la clase porque, en la mayoría de los demás métodos de `GestorTransacciones` (`ActualizarEstadoTransaccion`, `eliminarTransaccion`, `buscarTransaccion`), sí se necesita el puntero real, no solo el booleano.

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
- **`ifstream archivo(nombreArchivo)`**: abre el archivo en modo lectura de texto. Se valida `archivo.is_open()` para manejar el caso de archivo inexistente o ruta incorrecta (robustez exigida en la rúbrica). Si falla, se hace `return;` inmediato, antes de tocar `tablaHash` o `arbolAVL`.

#### Cómo `linea` recibe su valor sin un `=` visible: `getline` por referencia

`string linea;` solo **reserva** la variable (la crea vacía) — no tiene contenido del archivo todavía. El contenido llega en cada `while (getline(archivo, linea))`. La firma real de `getline` (simplificada) es:

```cpp
istream& getline(istream& flujo, string& variable);
```

Nótese el `&` después de `string`: `getline` recibe `variable` **por referencia** — no una copia, sino un alias directo de la variable que se le pasa. Cuando `getline` internamente asigna el texto leído a `variable`, en realidad está modificando **tu** `linea` original, porque ambas son la misma caja de memoria (el mismo mecanismo de referencia que ya vimos con `int &contador` en `inorderR`, solo que aquí ya viene implementado dentro de la función de la biblioteca estándar). Concretamente, en cada vuelta del `while`, `getline(archivo, linea)`: (1) lee todos los caracteres desde la posición actual del cursor del archivo hasta el próximo `\n`; (2) escribe ese texto directamente en `linea` (gracias a la referencia); (3) avanza el cursor del archivo; (4) devuelve el propio `archivo`, que se interpreta como `true`/`false` según si la lectura tuvo éxito — así el `while` sabe cuándo detenerse (al llegar al final del archivo) sin necesitar una condición explícita como `while (!archivo.eof())` (que de hecho sería una práctica incorrecta y propensa a errores).

`linea` es la **misma variable reutilizada y sobrescrita** en cada vuelta del bucle, no una nueva cada vez.

#### `tieneCabecera` — un parámetro reutilizado como "interruptor de una sola vez"

```cpp
if (tieneCabecera)
{
    tieneCabecera = false; // Ignora solo la primera linea
    continue;
}
```

A diferencia de la "asignación invisible" de `getline`, aquí `tieneCabecera = false;` es una asignación normal y explícita. `tieneCabecera` es un parámetro de la función (variable local, como si se hubiera declarado dentro del cuerpo). En la primera vuelta del `while`, si llegó como `true` (valor por defecto), entra al bloque, se pone en `false`, y `continue` salta el resto del cuerpo para esa vuelta (sin tokenizar esa línea como datos) y pasa a leer la siguiente línea. En todas las vueltas siguientes, `tieneCabecera` ya vale `false`, así que la condición nunca se vuelve a cumplir. Es, en esencia, un interruptor que nace encendido (si el archivo trae cabecera) y se apaga solo, una única vez.

- **Descarte de líneas vacías** (`if (linea.empty()) continue;`): protección contra líneas en blanco al final del archivo — sin esto, `stof("")` sobre un campo vacío lanzaría una excepción.

#### `stringstream ss(linea)` y la tokenización con `getline(ss, campo, separador)`

```cpp
stringstream ss(linea);
string id, origen, cliente, tipo, fecha, hora, estado, montoStr;
getline(ss, id, separador);
getline(ss, origen, separador);
...
```

`stringstream` convierte un `string` ya existente en memoria en un **flujo de entrada** (*input stream*), exactamente como `ifstream` trata un archivo en disco como un flujo — la diferencia es que `ss` "lee" del contenido de `linea`, no del disco, y mantiene su **propio cursor**, independiente del cursor de `archivo`.

Aquí se usa la versión de **tres argumentos** de `getline` — una firma distinta a la de dos argumentos usada para leer del archivo:

```cpp
istream& getline(istream& flujo, string& variable, char delimitador);
```

Mientras la versión de 2 argumentos siempre usa `\n` como delimitador, la de 3 argumentos permite elegir **cualquier carácter** como delimitador — aquí, `separador` (la coma). Cada llamada consecutiva `getline(ss, campo, ',')` lee desde donde quedó el cursor de `ss` hasta la próxima coma, guarda ese fragmento en `campo`, y avanza el cursor — por eso el orden de las 8 llamadas debe coincidir exactamente con el orden real de las columnas del CSV (`id, origen, cliente, tipo, monto, fecha, hora, estado`): cada llamada agarra secuencialmente el siguiente fragmento, sin "buscar" posiciones.

`montoStr` se extrae como texto (nunca directamente como `float`, porque `getline` solo puede extraer `string`) y se convierte a número recién en el siguiente paso, con `stof`.

- **Limpieza del carácter `\r`**: los archivos generados/editados en Windows usan terminación `\r\n`; `getline(archivo, linea)` solo separa por `\n`, dejando el `\r` pegado al final del último campo (`estado`). El código detecta y elimina ese carácter invisible con `estado.pop_back()` (tras verificar `!estado.empty()` para no acceder a `.back()` de un string vacío) — sin esto, `estado` podría quedar como `"Aprobada\r"` en vez de `"Aprobada"`, rompiendo silenciosamente comparaciones como `estado == "Aprobada"` en las estadísticas.
- **Verificación de duplicados también en la carga masiva** (`tablaHash->buscarPorId(id) == nullptr`): garantiza que, si el CSV tuviera un ID repetido por error, solo se inserte la primera ocurrencia — consistencia de datos entre hash y árbol.
- **`stof(montoStr)` sin `try/catch`**: si `montoStr` no fuera un número válido, `stof` lanzaría una excepción que, al no estar capturada, terminaría el programa abruptamente — una posible mejora a mencionar en la Discusión del informe sería envolver esto en `try/catch` para mayor robustez ante archivos malformados.
- **Contadores `contadorHash` y `contadorAVL`**: el PDF pide explícitamente mostrar "la cantidad de registros insertados en la hash y en el árbol" tras la carga masiva (Escenario de prueba 1). Aunque en este código siempre son iguales (cada inserción exitosa va a ambas estructuras a la vez), mantenerlos separados documenta claramente que ambas estructuras están sincronizadas.

**Complejidad total de la carga masiva**: si hay `n` transacciones en el archivo, cada una realiza: 1 búsqueda hash O(1) amortizado + 1 inserción hash O(1) amortizado + 1 inserción AVL Θ(log n). Por lo tanto, la carga completa es **Θ(n log n)** — dominada por las `n` inserciones en el árbol AVL.

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

### 8.1 `mostrarTiempo` y el mecanismo de medición con `<chrono>`

```cpp
void mostrarTiempo(const string &operacion,
                    chrono::high_resolution_clock::time_point inicio,
                    chrono::high_resolution_clock::time_point fin)
{
    chrono::duration<double, std::milli> tiempo = fin - inicio;
    cout << "Tiempo de " << operacion << ": " << tiempo.count() << " ms" << endl;
}
```

#### `high_resolution_clock` — el reloj usado

```cpp
auto inicio = high_resolution_clock::now();
```

- `high_resolution_clock` es uno de los tres relojes de `<chrono>` (junto a `system_clock` y `steady_clock`), elegido por ofrecer la **mayor precisión posible** del sistema — ideal para medir operaciones muy rápidas, como una búsqueda hash que puede tardar microsegundos.
- `now()` es una función **estática** que devuelve **el instante actual** — una "foto" del reloj. Su tipo de retorno es `chrono::high_resolution_clock::time_point`: un "punto en el tiempo" (un instante, no una duración).
- **`auto`**: le pide al compilador que infiera el tipo a partir de lo que devuelve `now()`, evitando escribir el nombre completo `chrono::high_resolution_clock::time_point` a mano. `auto inicio = high_resolution_clock::now();` es exactamente equivalente a escribir el tipo completo explícitamente.

#### El patrón "sándwich": medir solo la operación, no la espera del usuario

```cpp
auto inicio = chrono::high_resolution_clock::now();   // (1) foto del reloj ANTES
gestor.buscarTransaccion(id);                          // (2) la operación que se quiere medir
auto fin = chrono::high_resolution_clock::now();       // (3) foto del reloj DESPUÉS
mostrarTiempo("Busqueda", inicio, fin);                // (4) calcular e imprimir la diferencia
```

Es crítico que `inicio` y `fin` envuelvan **exactamente** la operación y nada más. En el `main()` real, cualquier lectura de datos por teclado (`leerId(...)`, `leerEstado()`, etc.) ocurre **antes** de `auto inicio = ...`, precisamente para que el tiempo que el usuario tarda en escribir no contamine la medición del costo algorítmico real.

#### Cómo `mostrarTiempo` calcula la diferencia

- **`const string &operacion`**: referencia constante — evita copiar el string (eficiencia) y garantiza que la función no lo modifique. Mismo patrón recomendado para pasar cualquier objeto de "solo lectura" que no sea un tipo primitivo.
- **`inicio` y `fin` se pasan por valor** (sin `&`): a diferencia de `operacion`, un `time_point` es un tipo pequeño internamente (esencialmente un número), así que copiarlo no tiene costo relevante.
- **`fin - inicio`**: aunque `fin` e `inicio` son instantes (no duraciones), `chrono` tiene sobrecargado el operador `-` para que restar dos instantes dé como resultado una **duración** — el tiempo transcurrido entre ambos (análogo a "las 3:00 PM menos las 2:45 PM son 15 minutos").
- **`chrono::duration<double, std::milli>`**: el tipo al que se convierte esa resta. `duration` es una plantilla que recibe (a) el tipo numérico interno (`double`, para tener precisión fraccionaria) y (b) la unidad (`std::milli` = milisegundos; también existen `std::micro`, `std::nano`, `std::ratio<1>` para segundos). El resultado crudo de `fin - inicio` viene en la unidad nativa del reloj (usualmente nanosegundos), y `chrono` hace la conversión de unidades automáticamente al asignarlo a esta variable — no hay ninguna división manual escrita en el código.
- **`.count()`**: `tiempo` no es un número "puro" — es un objeto `chrono::duration<...>` que encapsula cantidad y unidad (para evitar sumar milisegundos con segundos por error). `.count()` extrae el valor numérico interno (`double`) para poder imprimirlo con `cout <<`.

**Ejemplo numérico**: si `fin - inicio` valen 3,400 nanosegundos (la unidad nativa del reloj), al convertir a `duration<double, std::milli>`, `chrono` divide automáticamente entre 1,000,000 → `tiempo.count()` devuelve `0.0034`, y se imprime `"Tiempo de Busqueda: 0.0034 ms"`.

#### `using namespace chrono;` dentro de `main()`

Justo antes de la carga masiva automática, el código escribe `using namespace chrono;`, permitiendo usar `high_resolution_clock::now()` sin el prefijo `chrono::`. Más adelante, dentro del `switch` del menú, el código vuelve a escribir el prefijo completo (`chrono::high_resolution_clock::now()`) en cada `case` — ambas formas son válidas y equivalentes; es solo una inconsistencia de estilo, sin efecto en el comportamiento.

#### Relación exacta con la tabla de tiempos del PDF

Cada fila de la tabla de la sección 5 del informe corresponde a una llamada específica a `mostrarTiempo` dispersa en `main()`:

| Fila de la tabla del informe | Llamada en el código |
|---|---|
| Carga masiva | `mostrarTiempo("Carga", inicio, fin)` (antes del menú) |
| Búsqueda por ID | `mostrarTiempo("Busqueda", inicio, fin)` (case 2) |
| Inserción individual | `mostrarTiempo("Insercion", inicio, fin)` (case 1) |
| Consulta ordenada | `mostrarTiempo("Consulta Cronologica", inicio, fin)` (case 3) |
| Consulta por rango | `mostrarTiempo("Consulta por Rango", inicio, fin)` (case 4) |
| Actualización | `mostrarTiempo("Actualizacion", inicio, fin)` (case 5) |
| Eliminación | `mostrarTiempo("Eliminacion", inicio, fin)` (case 6) |

Nótese que el **case 7 (estadísticas)** es el único que **no** mide tiempo con `chrono` — la tabla del PDF tampoco lo exige explícitamente, lo cual explica la omisión, aunque sería una mejora sencilla agregarlo de todas formas (es una operación Θ(n) interesante de reportar en la Discusión).

### 8.2 Funciones de validación de entrada

Todas las funciones `leer*` comparten el mismo patrón defensivo:

```cpp
TipoDeRetorno leerAlgo(...)
{
    TipoDeRetorno valor;
    while (true)          // bucle infinito
    {
        // pedir el dato al usuario
        // validar
        if (/* es inválido */) { mostrar error; continue; }
        return valor;      // única forma de salir del bucle
    }
}
```

No hay un número fijo de intentos — la función sigue pidiendo el dato **hasta que sea válido**. El único punto de salida es el `return` dentro del `if` de éxito; `continue` hace que el `while` vuelva a evaluar su condición (`true`, siempre se cumple) y repita el cuerpo desde el inicio. Esto garantiza que **el programa nunca continúa con un dato inválido**, la base de la robustez exigida en la rúbrica ("consistencia de los datos").

#### `leerEntero` — `cin.fail()`, `cin.peek()` y limpieza de buffer

```cpp
int leerEntero(string mensaje)
{
    int valor;
    while (true)
    {
        cout << mensaje;
        cin >> valor;
        if (!cin.fail() && cin.peek() == '\n')
            return valor;
        cout << "Error: ingrese un numero entero valido." << endl;
        cin.clear();
        while (cin.get() != '\n') {}
    }
}
```

- **`cin.fail()`**: devuelve `true` si la última lectura falló (por ejemplo, el usuario escribió `"abc"` donde se esperaba un número). `!cin.fail()` significa "la lectura fue exitosa".
- **`cin.peek() == '\n'`**: `peek()` mira el siguiente carácter pendiente en el buffer **sin consumirlo**. Esto comprueba que, justo después del número leído, venga inmediatamente un salto de línea. Es necesario porque si el usuario escribe `"25abc"`, `cin >> valor` extrae `25` (sin fallar) y deja `"abc"` pendiente en el buffer — sin este chequeo, se aceptaría `25` como válido, dejando `"abc"` para contaminar la **siguiente** lectura de `cin` en el programa.
- **`cin.clear()`**: cuando `cin` entra en estado de error, queda "bloqueado" — cualquier lectura posterior fallará automáticamente hasta resetear ese estado. `clear()` limpia esas banderas de error.
- **`while (cin.get() != '\n') {}`**: descarta, carácter por carácter, todo el contenido corrupto que quedó en el buffer, hasta encontrar el `\n` (que también se consume). Es imprescindible: sin esto, la siguiente vuelta del `while` externo volvería a leer sobre el mismo texto corrupto sin limpiar, entrando en un **bucle infinito** que repite el mismo error para siempre.

#### `leerMonto` — mismo patrón + validación de regla de negocio

```cpp
if (!cin.fail() && cin.peek() == '\n' && monto > 0)
    return monto;
```

Idéntico a `leerEntero`, cambiando `int` por `float` y agregando `monto > 0` — combina validación de **formato** (que sea un número bien formado) con validación de **regla de negocio** (que tenga sentido en el dominio: un monto bancario no puede ser negativo ni cero). Si `monto` fuera `-50` (formato válido), la condición completa igual sería `false` por culpa de `monto > 0`.

#### `leerTexto` — `getline(cin >> ws, texto)`

```cpp
string leerTexto(string mensaje)
{
    string texto;
    while (true)
    {
        cout << mensaje;
        getline(cin >> ws, texto);
        if (!texto.empty()) return texto;
        cout << "Error: el campo no puede estar vacio." << endl;
    }
}
```

- No se usa `cin >> texto` porque se detiene en el primer espacio en blanco (un nombre con espacio como `"Ana Torres"` solo capturaría `"Ana"`). `getline` lee la línea completa, espacios incluidos.
- **`cin >> ws`**: `ws` es un manipulador de flujo que descarta cualquier espacio en blanco pendiente (incluyendo saltos de línea residuales de una lectura anterior con `cin >>`). Resuelve un bug clásico: si justo antes se usó `cin >> algo` (como en `leerEntero`/`leerId`), queda un `\n` sin consumir; sin `cin >> ws`, la primera llamada a `getline` leería inmediatamente ese `\n` residual y devolvería una cadena vacía, sin darle al usuario oportunidad real de escribir.
- Solo valida `!texto.empty()` — no exige ningún formato particular, a diferencia de fecha/hora/ID.

#### `leerTipo` y `leerEstado` — validación contra una lista fija

```cpp
if (tipo == "Transferencia" || tipo == "Retiro" || tipo == "PagoServicio" ||
    tipo == "CompraWeb" || tipo == "Deposito")
    return tipo;
```

No necesitan el chequeo `cin.fail()`/`peek()` porque `cin >> string` nunca falla por formato — cualquier secuencia de caracteres sin espacios es válida como `string`. El único criterio aquí es semántico: coincidencia **exacta** (sensible a mayúsculas/minúsculas) con uno de los 5 valores permitidos. `leerEstado` es análogo, con la lista `Aprobada, Pendiente, Rechazada, Observada, Anulada` — ambas listas coinciden exactamente con los valores sugeridos en el PDF.

#### `esDigito` — auxiliar reutilizada por tres validadores

```cpp
bool esDigito(char c)
{
    return c >= '0' && c <= '9';
}
```

Compara directamente los valores ASCII: como `'0'`-`'9'` son consecutivos en la tabla ASCII (48-57), esta comparación de rango es equivalente a verificar si `c` es un dígito. No se usó `isdigit()` de `<cctype>` (la función estándar equivalente) — decisión de estilo consistente con implementar los detalles "desde cero", aunque usar `isdigit()` no violaría ninguna restricción del PDF. Se reutiliza dentro de `leerFecha`, `leerHora` y `leerId`.

#### `leerFecha` — validación por capas: formato + rangos lógicos

```cpp
if (fecha.size() != 10) { ...error...; continue; }               // Capa 1: longitud exacta
if (fecha[4] != '-' || fecha[7] != '-') { ...error...; continue; } // Capa 2: guiones en posición
// Capa 3: recorrido con esDigito, saltando posiciones 4 y 7
int anio = stoi(fecha.substr(0,4));
int mes  = stoi(fecha.substr(5,2));
int dia  = stoi(fecha.substr(8,2));
// Capa 4: anio<=0, mes 1-12, dia 1-31
```

- **Capa 1**: el formato `YYYY-MM-DD` siempre tiene 10 caracteres — detecta, por ejemplo, `"2026-6-1"` (sin ceros a la izquierda).
- **Capa 2**: acceso directo por índice (`fecha[4]`, `fecha[7]`) para verificar los guiones — detecta formatos como `"2026/06/01"`.
- **Capa 3**: recorre cada posición saltando (con `continue`) las de los guiones, usando `esDigito` en el resto; corta con `break` en el primer carácter inválido.
- **Capa 4**: `fecha.substr(inicio, cantidad)` extrae cada componente como texto, `stoi(...)` lo convierte a entero, y se validan rangos: año positivo, mes 1-12, día 1-31.
- **Limitación reconocida**: el día se valida contra el rango fijo `1-31` sin considerar cuántos días tiene realmente cada mes — por ejemplo, `"2026-02-31"` (31 de febrero, inexistente) pasaría la validación. Es una simplificación consciente, válida para mencionar proactivamente en la sustentación.

#### `leerHora` — mismo patrón que `leerFecha`, aplicado a `HH:MM:SS`

Réplica estructural de `leerFecha`: longitud 8, dos puntos (`:`) en posiciones 2 y 5, recorrido con `esDigito`, y rangos exactos y correctos (horas 0-23, minutos y segundos 0-59) — a diferencia de la fecha, aquí no hay ninguna simplificación pendiente.

#### `leerId` — formato fijo con prefijo literal

```cpp
if (id.size() != 9) ...
if (id[0] != 'T' || id[1] != 'X' || id[2] != '-') ...
// bucle desde la posición 3 hasta la 8, verificando esDigito
```

Longitud 9 (`TX-XXXXXX` = 2 letras + 1 guion + 6 dígitos), verificación de las letras específicas `T`, `X` y el guion, y verificación de que las 6 posiciones restantes sean dígitos.

#### Resumen de la sección de validaciones

| Función | Qué valida | Mecanismo | Reutiliza |
|---|---|---|---|
| `leerEntero` | Número entero completo, sin basura después | `cin.fail()` + `cin.peek() == '\n'` + limpieza de buffer | — |
| `leerMonto` | Igual que `leerEntero`, más `> 0` | Igual + condición de negocio extra | — |
| `leerTexto` | No vacío (acepta espacios internos) | `getline(cin >> ws, texto)` + `.empty()` | — |
| `leerTipo` / `leerEstado` | Coincide con 1 de 5 valores permitidos | Cadena de `||` con `==` | — |
| `esDigito` | Carácter `'0'`-`'9'` | Comparación de rango ASCII | Usada por `leerFecha`, `leerHora`, `leerId` |
| `leerFecha` | `YYYY-MM-DD` + rangos de mes/día | Longitud + posiciones + `esDigito` + `stoi` + rangos | `esDigito` |
| `leerHora` | `HH:MM:SS` + rangos de hora/min/seg | Misma estructura que `leerFecha` | `esDigito` |
| `leerId` | `TX-XXXXXX` | Longitud + prefijo literal + `esDigito` | `esDigito` |

El principio unificador: **nunca confiar en que el usuario escribirá algo bien formado**, validar en capas progresivas, y **nunca dejar avanzar el programa** con un dato que no pasó todas las validaciones.

---

## 9. `main()`: el flujo completo del programa

### 9.1 Preparación inicial (antes del menú)

```cpp
int main()
{
    cout << "--- Sistema de Analisis de Transacciones ---" << endl;
    GestorTransacciones gestor(15013);

    cout << "Iniciando carga automatica de datos desde el archivo masivo..." << endl;
    using namespace chrono;
    auto inicio = high_resolution_clock::now();
    gestor.cargarDesdeArchivo("transacciones_masivo.csv", ',');
    auto fin = high_resolution_clock::now();
    mostrarTiempo("Carga",inicio,fin);

    int opcion = 0;
    string id, origen, cliente, tipo, fecha, hora, estado, fInicio, fFin;
    float monto;
```

- `GestorTransacciones gestor(15013);`: **una sola instancia**, viviendo en la pila (no un puntero, sin `new`) durante toda la ejecución — todas las operaciones del menú actúan sobre este mismo objeto.
- **Carga automática**: se ejecuta el patrón "sándwich" de `chrono` (sección 8.1) alrededor de `cargarDesdeArchivo`, cumpliendo el requisito del PDF de demostrar el sistema con al menos 10,000 transacciones desde el primer momento, sin depender de que el usuario recuerde ejecutar una opción de menú.
- **Variables declaradas una sola vez, fuera del `switch`** (`id, origen, cliente, ...`): se reutilizan entre distintos `case` (por ejemplo, `id` en los case 1, 2, 5 y 6), en vez de redeclararlas dentro de cada uno.
- `opcion = 0;`: valor inicial que no corresponde a ninguna opción válida, para que la condición del `while` (`opcion != 8`) sea verdadera en la primera vuelta.

### 9.2 El bucle principal: `while (opcion != 8)`

```cpp
while (opcion != 8)
{
    // ... imprime las 8 opciones del menú ...
    opcion = leerEntero("Seleccione una opcion: ");
    switch (opcion) { ... }
}
```

El menú se **redibuja por completo en cada vuelta** — no hay una versión estática que se muestre una sola vez. La condición de salida se evalúa **al principio** de cada vuelta: aunque el usuario elija `8` dentro del `switch`, el `case 8` se ejecuta una vez (imprime el mensaje de despedida) antes de que el `while` vuelva a evaluar su condición y decida terminar — no hay ningún `return`/`break` que rompa el `while` directamente desde dentro del `case 8`.

### 9.3 Por qué cada `case` tiene sus propias llaves `{ }`

```cpp
switch (opcion)
{
case 1:
    {
        id = leerId(...);
        ...
        break;
    }
case 2:
    {
        ...
        break;
    }
...
}
```

En un `switch`, todos los `case` que no tienen llaves propias **comparten el mismo scope**. Si `case 1` y `case 2` declararan ambos, por ejemplo, `auto inicio = ...;` sin llaves, el compilador marcaría un **error de redeclaración**, porque sin llaves ambas declaraciones estarían técnicamente en el mismo bloque. Al envolver cada `case` en su propio `{ }`, cada uno obtiene un **scope aislado**: la variable `inicio` de `case 1` es una variable completamente distinta a la `inicio` de `case 2`, aunque compartan nombre. Esto explica por qué `auto inicio = chrono::high_resolution_clock::now();` se repite en casi todos los `case` sin que el compilador se queje.

### 9.4 Recorrido de cada `case`

#### `case 1` — Registrar (Funcionalidad 2)
```cpp
id = leerId(...); origen = leerTexto(...); cliente = leerTexto(...); tipo = leerTipo();
monto = leerMonto(...); fecha = leerFecha(...); hora = leerHora(...); estado = leerEstado();
auto inicio = ...; gestor.nuevaTransaccion(...); auto fin = ...;
mostrarTiempo("Insercion",inicio,fin);
```
Los 8 campos se piden y validan **antes** de iniciar la medición de tiempo — coherente con el principio de no incluir el tiempo de tecleo del usuario dentro de la medición de "Inserción" (sección 8.1).

#### `case 2` — Buscar por ID (Funcionalidad 3)
El más simple: pide el ID validado, mide el tiempo alrededor de `buscarTransaccion` (que delega en `tablaHash->buscarPorId`, O(1) amortizado).

#### `case 3` — Consulta cronológica (Funcionalidad 4)
```cpp
int k;
k = leerEntero(...);
while(k<=0) { cout<<"Debe ser mayor que cero"<<endl; k = leerEntero(...); }
```
Aquí aparece una **validación adicional manual**, fuera de las funciones `leer*` reutilizables: `leerEntero` garantiza formato numérico, pero un `while (k<=0)` propio de este `case` garantiza además que `k` sea positivo. Es una asimetría de diseño notable: `leerMonto` incorpora su validación de "mayor que cero" **dentro** de la función reutilizable, mientras que aquí se dejó como un bucle suelto en `main()` en vez de crear una función `leerEnteroPositivo` — funcionalmente correcto, pero inconsistente en estilo.

#### `case 4` — Consulta por rango (Funcionalidad 5)
```cpp
fInicio = leerFecha(...); fFin = leerFecha(...);
while (fInicio > fFin) { ...; fInicio = leerFecha(...); fFin = leerFecha(...); }
...
gestor.MostrarPorRangoFechas(fInicio, fFin, false);
```
- Validación de coherencia entre fechas (`fInicio > fFin`, comparación de strings que funciona por el mismo principio que la clave del AVL: ancho fijo → orden lexicográfico = orden cronológico). Si falla, se vuelven a pedir **ambas** fechas desde cero, aunque solo una estuviera mal.
- El tercer argumento `false` corresponde al parámetro `limite`/`limitacion` de `consultarRangoFechasR`: indica que se muestren **todos** los resultados (no solo los primeros 30), aunque el conteo total siempre se calcula completo.

#### `case 5` — Actualizar estado (Funcionalidad 6)
Pide ID y nuevo estado (validados), mide el tiempo de `ActualizarEstadoTransaccion` — O(1) amortizado porque solo localiza el puntero en el hash y llama `setEstado()`, sin tocar el AVL.

#### `case 6` — Eliminar (Funcionalidad 7)
Pide el ID, mide el tiempo de `eliminarTransaccion` — reconstruye la clave del AVL desde el hash, elimina primero del árbol y luego del hash, con auto-verificación.

#### `case 7` — Estadísticas (Funcionalidad 8)
```cpp
case 7:
    gestor.mostrarEstadisticas();
    break;
```
El **único** `case` sin llaves propias (no declara variables locales) y el **único que no mide tiempo con `chrono`** — la tabla de tiempos del PDF no incluye explícitamente "estadísticas" como fila obligatoria, lo que podría explicar la omisión, aunque sería sencillo agregar el mismo patrón de medición.

#### `case 8` — Salir
Solo imprime un mensaje; el `while` termina en la **siguiente** evaluación de `opcion != 8`, no por ningún mecanismo dentro del propio `case`.

#### `default`
Captura cualquier valor fuera de 1-8. `leerEntero` garantiza formato numérico pero no rango válido — este `default` es la capa final que evita ejecutar código para una opción inexistente.

### 9.5 Cierre del programa y destrucción automática (RAII)

```cpp
    }
    return 0;
}
```

Al terminar `main()` (cuando `opcion == 8`), la variable local `gestor` sale de *scope* y se **destruye automáticamente** — esto dispara en cascada: `~GestorTransacciones()` → `delete tablaHash` (destruye cada `ListaEnlazada`, liberando cada `Nodo` y cada `Transaccion*`) y `delete arbolAVL` (recorre en post-order liberando cada `NodoAVL`, sin duplicar la liberación de las transacciones). Esto ocurre por **RAII** (*Resource Acquisition Is Initialization*): como `gestor` es una variable de pila, no un puntero manual, su destructor se garantiza que se ejecute al salir de *scope*, sin necesitar un `delete gestor;` explícito.

### 9.6 Flujo de la información (visión de alto nivel)

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
   1. Registrar   → Hash.buscarPorId (evitar duplicado) → new Transaccion → Hash.insertar + AVL.insert
   2. Buscar      → Hash.buscarPorId               → imprimir()
   3. Cronológico → AVL.mostrarCronologico (inorder)→ imprimir() x k
   4. Rango       → AVL.consultarRangoFechas        → imprimir() x resultados
   5. Actualizar  → Hash.buscarPorId → setEstado()  (mismo puntero, visible en ambas estructuras)
   6. Eliminar    → Hash.buscarPorId (leer datos) → AVL.eliminar(key) → Hash.eliminar(id) (libera memoria)
   7. Estadísticas→ AVL.generarReporteEstadistico (recorrido completo, sin medir tiempo)
   8. Salir       → return 0 → destrucción RAII de "gestor" → AVL::destruir + ~HashTableEncadenamiento
```

### 9.7 Puntos a resaltar en la sustentación
1. **Carga automática al inicio** (antes de mostrar el menú): el PDF exige demostrar el funcionamiento con al menos 10,000 transacciones desde el primer momento, y medir el tiempo de esa carga — el código lo hace automáticamente apenas se ejecuta el programa.
2. **Cada opción del menú mide su propio tiempo** con `chrono` (salvo `case 7`), cumpliendo la tabla de tiempos de la sección 5 del informe, siempre envolviendo solo la operación algorítmica, no la espera del teclado.
3. **`switch (opcion)` con bloques `{ }` en cada `case`**: necesarios para dar *scope* aislado a variables locales como `inicio`/`fin` en cada `case`, evitando errores de redeclaración.
4. **Inconsistencia menor de diseño**: la validación de `k > 0` en `case 3` se hace con un `while` manual en `main()`, en vez de una función `leer*` reutilizable — funciona igual de bien, pero rompe el patrón que sí siguen `leerMonto` o las demás funciones de validación.
5. **Gestión de memoria al salir**: al terminar `main()`, `gestor` (variable local, no puntero) se destruye automáticamente por RAII, disparando en cascada `~GestorTransacciones()` → `delete tablaHash` y `delete arbolAVL`, liberando toda la memoria sin duplicar la liberación de las transacciones (sección 4).

---

## 10. Análisis detallado de complejidad (temporal y espacial)

Esta sección profundiza en el análisis de complejidad de **cada estructura** y de **cada operación**, distinguiendo mejor caso, caso promedio y peor caso, y separando siempre el costo en **tiempo** del costo en **memoria (espacio)**. Es la sección que sustenta directamente el punto 3 ("Análisis... y análisis de complejidad") y el punto 6 ("Discusión") del informe técnico exigido por el PDF.

### 10.1 Conceptos previos necesarios para la sustentación

- **n** = número total de transacciones almacenadas en el sistema (mínimo 10,000 según el PDF).
- **m** = tamaño de la tabla hash (`size`), fijado en el código como `15013` (constante, no crece con `n`).
- **k** = número de elementos que se piden mostrar en una consulta acotada (ej. las primeras 20 o 50 transacciones cronológicas).
- **factor de carga (load factor)** de una tabla hash: `α = n / m`. Es la métrica más importante para predecir el rendimiento real de una tabla hash con encadenamiento. Con `n = 10,000` y `m = 15013`, `α ≈ 0.666`, es decir, cada *bucket* (lista enlazada) tiene, en promedio, **menos de un elemento**. Esto es deliberado: el enunciado exige mínimo 10,000 registros, y el tamaño de la tabla (número primo mayor a n) se eligió precisamente para mantener `α < 1` y así garantizar que las listas de colisión sean muy cortas.
- **Notación usada**: O(·) para cota superior (peor caso o caso general cuando no se distingue), Θ(·) cuando el mejor y el peor caso coinciden exactamente (por ejemplo, el AVL siempre garantiza altura Θ(log n), a diferencia de un BST simple que en el peor caso sería O(n)).

### 10.2 Complejidad de `HashTableEncadenamiento` (tabla hash con encadenamiento)

**Complejidad temporal**

| Operación | Mejor caso | Caso promedio (con buena distribución) | Peor caso |
|---|---|---|---|
| `funcionHash(id)` | O(L) | O(L) | O(L) |
| `insertar` | O(1) | O(1 + α) ≈ O(1) | O(n) |
| `buscarPorId` / `buscar` | O(1) | O(1 + α) ≈ O(1) | O(n) |
| `eliminar` | O(1) | O(1 + α) ≈ O(1) | O(n) |

- `L` = longitud del string `id` (constante y muy pequeña en este proyecto: siempre 9 caracteres, formato `TX-XXXXXX`). Como `L` es constante y acotado, `funcionHash` es efectivamente **O(1)** en la práctica, aunque formalmente dependa de la longitud del string recorrido carácter por carácter.
- El **caso promedio O(1 + α)** es el resultado clásico de teoría de tablas hash con encadenamiento: se paga 1 unidad por calcular la posición y, en promedio, `α` comparaciones adicionales por recorrer el *bucket* correspondiente. Con `α ≈ 0.666` en este proyecto, en la práctica cada operación hace **menos de una comparación adicional en promedio**, comportándose como una operación prácticamente constante.
- El **peor caso teórico O(n)** ocurriría solo si la función hash distribuyera muy mal los datos y **todas** las transacciones cayeran en el mismo *bucket* (degenerando la lista enlazada de ese *bucket* a contener las n transacciones). Con DJB2 y un tamaño de tabla primo, este escenario es extremadamente improbable con datos reales como los IDs secuenciales `TX-000001...TX-010000`, pero sigue siendo la cota superior formal que debe mencionarse en el informe.
- **Importante para la discusión del informe**: a diferencia del AVL, la tabla hash **no ofrece ninguna garantía determinística** de rendimiento — su buen comportamiento depende enteramente de la calidad de la función hash y de la relación `n/m`. Esto es una limitación intrínseca de las tablas hash que vale la pena mencionar al comparar ambas estructuras en la sección de Discusión del informe.

**Complejidad espacial**

| Componente | Espacio ocupado |
|---|---|
| Arreglo `tabla` (buckets) | Θ(m) — fijo, independiente de cuántos elementos se inserten |
| Nodos de las listas enlazadas (todos los *buckets* combinados) | Θ(n) — un `Nodo` por cada transacción insertada |
| Cada `Nodo` individual | O(1) — contiene una `tuple<string, Transaccion*>` (un string corto + un puntero de 8 bytes) y un puntero `nextNodo` (8 bytes) |
| **Total tabla hash** | **Θ(n + m)** |

- El espacio del arreglo `tabla` se reserva **por adelantado** con `new ListaEnlazada[size]` en el constructor, y es **constante** (no crece ni decrece aunque se inserten o eliminen transacciones) — son m = 15,013 objetos `ListaEnlazada` (cada uno solo con un puntero `head`, es decir, 8 bytes cada uno más overhead del objeto).
- El espacio de los nodos **sí crece linealmente con n**, porque por cada transacción insertada se crea exactamente un `Nodo` nuevo en el heap (`new Nodo(...)`).
- Nótese que la tabla hash **no almacena una copia de la `Transaccion`**, solo un puntero (`Transaccion*`) de 8 bytes (en una arquitectura de 64 bits). El objeto `Transaccion` real se cuenta como parte del costo compartido con el AVL (ver sección 10.4), no se duplica.
- **Trade-off espacio/tiempo explícito**: el tamaño de tabla `m = 15013` fue elegido deliberadamente **más grande** que el mínimo de 10,000 transacciones exigido, sacrificando algo de memoria (buckets vacíos o con pocos elementos) a cambio de mantener `α < 1` y así garantizar tiempos de acceso casi constantes. Es un ejemplo clásico y citable de decisión de diseño espacio-vs-tiempo.

### 10.3 Complejidad de `AVL` (árbol binario auto-balanceado)

**Complejidad temporal**

| Operación | Mejor caso | Caso promedio | Peor caso |
|---|---|---|---|
| `insert` (`insertR`) | Θ(log n) | Θ(log n) | Θ(log n) |
| `eliminar` (`eliminarNodo`) | Θ(log n) | Θ(log n) | Θ(log n) |
| Búsqueda de una clave puntual | Θ(log n) | Θ(log n) | Θ(log n) |
| `mostrarCronologico(k)` (`inorderR`) | Θ(k) | Θ(k) | Θ(k), acotado por Θ(n) si k ≥ n |
| `consultarRangoFechas` (`consultarRangoFechasR`) | Θ(log n) (rango vacío) | Θ(log n + m_r) | Θ(n) (rango cubre todo el árbol) |
| `generarReporteEstadistico` (`calcularEstadisticasR`) | Θ(n) | Θ(n) | Θ(n) |
| `rotateLeft` / `rotateRight` | Θ(1) | Θ(1) | Θ(1) |

Donde `m_r` = número de transacciones que efectivamente caen dentro del rango de fechas consultado.

- **Por qué el AVL garantiza Θ(log n) y no solo O(log n) o "O(n) en el peor caso" como un BST normal**: la propiedad AVL fuerza, después de **cada** inserción y eliminación, que el factor de equilibrio de todo nodo sea `|FE| ≤ 1` (mediante las 4 rotaciones implementadas). Esto matemáticamente garantiza que la altura del árbol nunca exceda `1.44 · log₂(n + 2) - 0.328` (cota demostrada formalmente en la teoría de árboles AVL), es decir, la altura está **acotada por ambos lados** por una función logarítmica — de ahí que se use la notación Θ (ajustada, no solo cota superior) en vez de O. Este es la diferencia crítica frente a un BST sin balanceo: si las 10,000 transacciones del CSV llegaran ya ordenadas cronológicamente (que es justamente el caso, ya que el archivo viene ordenado por fecha), un BST simple degeneraría en una lista enlazada de altura n, con búsquedas/inserciones O(n) — el AVL evita esto por construcción.
- **Costo de las rotaciones**: cada rotación (`rotateLeft`/`rotateRight`) es Θ(1) porque solo reasigna un puñado fijo de punteros (no recorre subárboles). Sin embargo, en el peor caso una sola operación de inserción/eliminación puede requerir **hasta O(log n) actualizaciones de altura** (`updateH`) mientras la recursión "sube" desde la hoja hasta la raíz — pero como cada una de esas actualizaciones y su posible rotación asociada es Θ(1), el costo total de la operación completa sigue siendo Θ(log n), no Θ(log² n).
- **`inorderR` con límite `k`**: gracias al corte temprano (`if (nodo == nullptr || contador >= limite) return;`), la función deja de descender por nuevas ramas en cuanto se alcanzan `k` elementos impresos. Sin embargo, cabe una precisión importante para la sustentación: como el árbol está ordenado por `fecha_hora_id` (no por profundidad), las primeras `k` claves cronológicas **no necesariamente están todas cerca de la raíz** — en el peor caso estructural, mostrar las primeras k transacciones cronológicas puede requerir descender hasta la hoja más a la izquierda del árbol (profundidad Θ(log n)) y luego recorrer hacia la derecha k veces, por lo que el costo real es **Θ(k + log n)**: el término `log n` cubre el descenso inicial hasta el nodo con la clave mínima, y el término `k` cubre la impresión de los k elementos siguientes en orden.
- **`consultarRangoFechasR` — por qué la poda mejora el caso promedio**: el algoritmo solo desciende por la izquierda si `fechaNodo >= desde` y solo desciende por la derecha si `fechaNodo <= hasta`. Esto significa que las ramas completas del árbol que quedan totalmente fuera del rango **nunca se visitan**, ahorrando trabajo proporcional al tamaño de esas ramas descartadas. Formalmente, el costo es **Θ(log n + m_r)**, donde el término `log n` corresponde al camino desde la raíz hasta el primer nodo dentro del rango, y `m_r` corresponde a los nodos efectivamente visitados dentro (o adyacentes) al rango. En el peor caso (rango que cubre todas las fechas del sistema), `m_r = n` y la complejidad se degrada a Θ(n), equivalente a un recorrido completo — esto es esperable, porque no hay forma de reportar n resultados en menos de Θ(n).
- **`generarReporteEstadistico` es intrínsecamente Θ(n)**, sin mejor ni peor caso distinguibles: no existe ningún atajo posible para calcular sumas, promedios, máximos/mínimos y conteos globales sin inspeccionar cada una de las n transacciones exactamente una vez. Es el único método de la clase `AVL` cuya complejidad no depende de la altura del árbol, sino directamente del número total de nodos.

**Complejidad espacial**

| Componente | Espacio ocupado |
|---|---|
| Cada `NodoAVL` | O(1) — un `string key` (tamaño fijo ~30 caracteres: `YYYY-MM-DD_HH:MM:SS_TX-XXXXXX`), un puntero `Transaccion*` (8 bytes), un `int h` (4 bytes) y dos punteros `left`/`right` (8 bytes cada uno) |
| Todos los `NodoAVL` del árbol | Θ(n) — un nodo por cada transacción insertada |
| Pila de llamadas recursivas (`insertR`, `eliminarNodo`, `inorderR`, `consultarRangoFechasR`, `calcularEstadisticasR`) | Θ(log n) en operaciones de inserción/eliminación/búsqueda puntual; hasta Θ(n) en el peor caso teórico para recorridos completos como `calcularEstadisticasR` (aunque en la práctica, gracias al balance AVL, la profundidad real de la recursión de un recorrido completo también está acotada por Θ(log n) por rama, sumando Θ(n) solo en número total de llamadas, no en profundidad simultánea de la pila) |
| **Total árbol AVL** | **Θ(n)** en memoria de nodos, más Θ(log n) de espacio adicional (pila) durante cualquier operación individual de inserción/eliminación/búsqueda |

- **Punto importante sobre la recursión y la memoria de pila**: todas las funciones del AVL (`insertR`, `eliminarNodo`, `inorderR`, `consultarRangoFechasR`) están implementadas de forma **recursiva**, lo cual consume espacio en la **pila de llamadas (call stack)** proporcional a la **profundidad de la recursión**, no al número total de nodos visitados. Gracias a la garantía de balance AVL, esa profundidad nunca excede Θ(log n) en las operaciones de inserción, eliminación y descenso hacia una clave puntual. Esto es una ventaja adicional del AVL frente a otras estructuras: **el espacio extra usado durante cualquier operación de modificación es logarítmico**, no lineal — algo que no estaría garantizado en un BST sin balanceo (donde la pila de recursión podría crecer hasta Θ(n) en el peor caso, con riesgo real de *stack overflow* con decenas de miles de transacciones si el árbol se desbalancea).
- El `string key` de cada `NodoAVL` es una **cadena adicional** (no reutiliza el string `fecha`/`hora`/`id` de `Transaccion`, los concatena en un nuevo objeto `string`). Esto significa que, estrictamente, el AVL sí duplica una pequeña cantidad de información textual (aproximadamente 30 caracteres por nodo) además de guardar el puntero a la transacción — un costo de memoria adicional pero acotado y constante por nodo, aceptado como *trade-off* para simplificar la comparación de claves a una simple comparación de strings.

### 10.4 Complejidad espacial global del sistema

Es importante distinguir, para el informe, que **la memoria de las transacciones en sí (`Transaccion`) se cuenta una sola vez**, aunque sea referenciada desde dos estructuras:

| Componente | Espacio |
|---|---|
| n objetos `Transaccion` (datos reales: 6 strings + 1 float) | Θ(n) — memoria "de dominio", contada una sola vez |
| Estructura de la tabla hash (arreglo `tabla` + n `Nodo`) | Θ(n + m) |
| Estructura del AVL (n `NodoAVL`, cada uno con su `string key` propio) | Θ(n) |
| **Total del sistema (`GestorTransacciones` completo)** | **Θ(n + m)**, que con `m` constante (15,013) se simplifica a **Θ(n)** |

- **Por qué no se cuenta el espacio de `Transaccion` dos veces**: tanto `HashTableEncadenamiento` como `AVL` almacenan únicamente **punteros** (`Transaccion*`, 8 bytes en arquitectura de 64 bits) hacia el mismo objeto en el heap, nunca una copia del objeto completo. Si en cambio cada estructura guardara una copia completa de `Transaccion` (con sus 6 `string` y su `float`), el costo de memoria del sistema se **duplicaría** innecesariamente. Este es uno de los argumentos técnicos más sólidos para justificar, en la sustentación, la decisión de diseño de compartir punteros en lugar de duplicar datos: se ahorra memoria proporcional a n y, adicionalmente (como ya se explicó en la sección 7.4), se garantiza consistencia automática entre ambas estructuras al actualizar el estado de una transacción.
- **Overhead de punteros de la doble estructura**: aunque los datos no se duplican, sí existe un costo de memoria por **mantener dos estructuras de indexación en paralelo** (los `Nodo` de la lista hash + los `NodoAVL`), cada uno con su propio conjunto de punteros de navegación. Este es el costo esperado y aceptado de cualquier sistema que necesita **dos criterios de acceso distintos** (por ID y por fecha/hora) sobre el mismo conjunto de datos — es exactamente el mismo principio detrás de los índices secundarios en bases de datos relacionales, donde cada índice adicional consume espacio propio a cambio de acelerar un tipo de consulta específico.

### 10.5 Complejidad de la carga masiva y del ciclo completo del CSV

| Fase dentro de `cargarDesdeArchivo` (por cada línea leída) | Complejidad |
|---|---|
| Lectura de la línea (`getline(archivo, linea)`) | O(L), L = longitud de la línea (constante y pequeña) |
| Tokenización con `stringstream` (8 campos) | O(L) |
| Conversión `stof(montoStr)` | O(L) |
| Verificación de duplicado (`tablaHash->buscarPorId`) | O(1) amortizado |
| Inserción en hash (`tablaHash->insertar`) | O(1) amortizado |
| Inserción en AVL (`arbolAVL->insert`) | Θ(log n) |
| **Costo por línea** | **Θ(log n)** (dominado por la inserción AVL) |
| **Costo total para n líneas** | **Θ(n log n)** |

- Con `n = 10,000` (el archivo `transacciones_masivo.csv` incluido tiene exactamente 10,000 registros de datos más 1 línea de cabecera), `n log₂ n ≈ 10,000 × 13.3 ≈ 133,000` operaciones elementales estimadas para las inserciones en el árbol — esta es la cifra de referencia que debería ser consistente con el tiempo medido empíricamente por `mostrarTiempo("Carga", ...)` al ejecutar el programa, y es el tipo de razonamiento que se espera desarrollar en la sección de Discusión del informe (comparar la complejidad teórica contra el tiempo real medido).
- **Espacio adicional durante la carga**: además del espacio final Θ(n) de las estructuras, durante la ejecución de `cargarDesdeArchivo` se usan variables locales de tamaño constante por iteración (`linea`, `id`, `origen`, etc., y el `stringstream ss`), que se reutilizan/destruyen en cada vuelta del `while` — es decir, **no se acumula memoria temporal proporcional a n durante la carga**, solo la memoria permanente de las estructuras finales.

### 10.6 Tabla resumen de complejidades (para la tabla del informe)

| Operación | Estructura(s) usada(s) | Complejidad temporal | Complejidad espacial adicional (aparte del almacenamiento fijo Θ(n)) | Justificación breve |
|---|---|---|---|---|
| Carga masiva (n transacciones) | Hash + AVL | Θ(n log n) | O(log n) de pila por cada inserción AVL (no acumulativo) | n inserciones AVL a Θ(log n) cada una (dominante); hash es O(1) amortizado por inserción |
| Búsqueda por ID | Tabla Hash | O(1) amortizado / O(n) peor caso | O(1) | Acceso directo por posición hash + recorrido corto del *bucket* |
| Inserción individual | Hash + AVL | Θ(log n) | O(log n) de pila (recursión AVL) | Dominada por la inserción balanceada en AVL; hash es O(1) amortizado |
| Consulta ordenada (cronológico, k elementos) | AVL (in-order) | Θ(k + log n) | O(log n) de pila | Descenso inicial al mínimo + recorrido de k elementos con corte temprano |
| Consulta por rango de fechas | AVL (poda de ramas) | Θ(log n + m_r) | O(log n) de pila | m_r = elementos en el rango; se descartan subárboles fuera de rango |
| Actualización de estado | Tabla Hash | O(1) amortizado | O(1) | Solo requiere localizar el puntero en el hash; el AVL no se modifica |
| Eliminación | Hash + AVL | Θ(log n) | O(log n) de pila (recursión AVL) | Dominada por el rebalanceo AVL; hash es O(1) amortizado |
| Estadísticas generales | AVL (recorrido completo) | Θ(n) | O(log n) de pila (profundidad, no nodos totales) | Debe visitar cada transacción exactamente una vez; no hay atajo posible |

**Complejidad espacial total del sistema (almacenamiento permanente):** Θ(n) para los datos de dominio (`Transaccion`) + Θ(n) para los nodos AVL + Θ(n + m) para la tabla hash (con m constante) = **Θ(n)** en total, ya que m es una constante fija (15,013) que no depende de n una vez elegida. En otras palabras: **la memoria total del sistema crece linealmente con la cantidad de transacciones**, con una pequeña constante multiplicativa por mantener dos estructuras de indexación (hash + AVL) en paralelo sobre el mismo conjunto de datos.

---

## 11. Resumen general del proyecto

El sistema implementa un **gestor de transacciones bancarias** que combina dos estructuras de datos clásicas, ambas construidas desde cero (sin STL de alto nivel), cada una optimizada para un tipo de consulta distinto:

- Una **tabla hash con encadenamiento** (`HashTableEncadenamiento`, usando `funcionHash` con el algoritmo DJB2 y `ListaEnlazada` para resolver colisiones), que ofrece **búsqueda, inserción y eliminación por ID en tiempo casi constante (O(1) amortizado)** — ideal para las operaciones de "búsqueda por ID", "verificar duplicados al registrar" y "actualizar estado".
- Un **árbol AVL auto-balanceado** (`AVL`/`NodoAVL`, indexado por la clave compuesta `fecha_hora_id`), que garantiza **inserción y eliminación en O(log n)** y permite **recorridos en orden cronológico** y **consultas eficientes por rango de fechas** con poda de subárboles irrelevantes — cumpliendo las funcionalidades de "consulta ordenada" y "consulta por rango".

Ambas estructuras **comparten los mismos punteros `Transaccion*`** en lugar de duplicar los datos, lo que garantiza consistencia automática cuando se actualiza el estado de una transacción (un solo `setEstado()` es visible desde ambas estructuras), evita errores de memoria como el *double free* (solo la tabla hash libera la memoria de las transacciones; el AVL solo libera sus propios nodos) y **ahorra memoria**: el costo espacial total del sistema es Θ(n) — lineal respecto al número de transacciones — en lugar de duplicarse por mantener dos índices independientes (ver sección 10.4). En cuanto al tiempo, la combinación de ambas estructuras logra que las operaciones más frecuentes (búsqueda, actualización) sean O(1) amortizado gracias al hash, mientras que las que requieren orden (consulta cronológica, por rango) se resuelven en Θ(log n) o Θ(log n + m) gracias a la garantía de balance del AVL, evitando la degradación a O(n) que sufriría un árbol binario simple con datos que ya llegan ordenados por fecha (ver sección 10.3).

La clase `GestorTransacciones` actúa como **fachada** que sincroniza ambas estructuras en cada operación de negocio (registrar, cargar, actualizar, eliminar), y el `main()` provee un menú interactivo que además **mide el tiempo de ejecución de cada operación con `<chrono>`**, generando exactamente los datos que exige la tabla de resultados del informe técnico. Un conjunto de funciones de validación de entrada (`leer*`) garantiza que el sistema nunca reciba datos corruptos desde el teclado, reforzando la robustez y consistencia de los datos exigida en la rúbrica de evaluación.

En conjunto, el código cubre las 8 funcionalidades obligatorias del PDF (carga masiva, registro manual, búsqueda por ID, consulta cronológica, consulta por rango, actualización de estado, eliminación y estadísticas generales), respetando la restricción de no usar `map`/`set`/`unordered_map` ni librerías externas, y dejando trazabilidad clara entre cada clase/función del código y el requisito correspondiente del enunciado — lo cual debería facilitar responder con seguridad cualquier pregunta técnica del profesor sobre "por qué se hizo así" en la sustentación.
