# Ejercicio B: Perfilado con herramientas

En este ejercicio se analiza el comportamiento del programa
`point_cloud_collimation` mediante tres herramientas de profiling:

- `perf`
- Google Performance Tools (`gperftools`)
- Valgrind con Callgrind

Las pruebas se realizan utilizando dos configuraciones:

1. Ejecución normal.
2. Ejecución con la opción `--export`.

Cada integrante del grupo realiza las mediciones en su propio equipo. Debido
a las diferencias de hardware y del entorno de ejecución, los resultados
pueden variar entre integrantes.

---

## 1. Especificaciones de los equipos

| Especificación | Angie | Milagro | Brayan | Alejandro |
|---|---|---|---|---|
| Procesador | Intel Core i5-10210U | Intel Core i7-12700H | AMD Ryzen 7 5700U with Radeon Graphics | Intel Core i9-13900HX |
| Núcleos / hilos | 4 / 8 | 14 / 20 | 8 / 16 | 24 / 32 |
| Memoria RAM | 16 GB | 16 GB | 16 GB | 32 GB  |
| Sistema operativo | Ubuntu 22.04.5 LTS | Ubuntu 26.04 LTS | Ubuntu 24.04.5 LTS | Ubuntu 24.04.5 LTS |
| Arquitectura | x86-64 | x86-64 | x86-64 | x86-64 |
| Kernel | 6.8.0-138-generic | 7.0.0-31-generic | 7.0.0-31-generic | 7.0.0-31-generic |
| Compilador | GCC/G++ 11.4 | GCC/G++ 15.2.0 | GCC/G++ 13.3.0 | GCC/G++ 13.3.0 |

Los resultados obtenidos deben interpretarse considerando las diferencias
entre los equipos utilizados por cada integrante.

---

## 2. Resultados de Angie

Los archivos completos obtenidos durante el profiling se encuentran en
la carpeta [`Angie/`](./Angie/).

### 2.1 Hotspots identificados

#### perf

En la ejecución normal, `perf report` identificó a
`GridIndex::nearest()` como el principal hotspot del programa, con
aproximadamente un **97.76 % Self**.

En la ejecución con `--export`, la misma función continúa concentrando
gran parte del trabajo. El árbol de llamadas muestra que una parte
importante de las llamadas ocurre mediante:

```text
collimate_icp()
└── compare_profiles()
    └── nearest_neighbor_distances()
        └── GridIndex::nearest()
```

También se observan llamadas directas a `GridIndex::nearest()` desde
el flujo de `collimate_icp()`.

#### Google Performance Tools

En la ejecución normal se obtuvieron 4177 muestras. La función
`GridIndex::nearest()` representó:

- 62.2 % de las muestras directas (`flat`).
- 97.9 % de las muestras acumuladas (`cum`).

También aparecen funciones relacionadas con las estructuras utilizadas
por `GridIndex::nearest()`, entre ellas:

- `std::_Hashtable::_M_find_before_node`
- `std::__detail::_Mod_range_hashing::operator`
- `std::vector::operator[]`
- `std::__detail::_Hashtable_base::_M_equals`

Con `--export`, `GridIndex::nearest()` continúa siendo el hotspot
principal, con 54.7 % `flat` y 85.8 % acumulado.

#### Valgrind / Callgrind

Callgrind contabilizó en la ejecución normal:

- 203,308,860,828 instrucciones (`Ir`) totales.
- 169,657,883,014 instrucciones directamente asociadas al código de
  `GridIndex::nearest()`.
- Esto corresponde aproximadamente al 83.45 % del total.

También se observa trabajo asociado a `std::vector` y a la implementación
de la tabla hash utilizada por `GridIndex::nearest()`.

Con `--export`, Callgrind contabilizó 239,900,455,408 instrucciones
totales y `GridIndex::nearest()` representó el 70.71 %.

### 2.2 ¿Coinciden los resultados de las tres herramientas?

Sí. Las tres herramientas identifican `GridIndex::nearest()` como la
principal región de interés para el rendimiento.

Los porcentajes obtenidos no son iguales porque las herramientas no
miden exactamente lo mismo.

`perf` utiliza contadores de rendimiento y muestreo para determinar dónde
se concentra la ejecución. Callgrind utiliza instrumentación y, en estas
pruebas, contabilizó el evento `Ir`. Google Performance Tools utiliza
muestreo del CPU y diferencia entre muestras directas (`flat`) y
acumuladas (`cum`).

Por lo tanto, los valores porcentuales no deben compararse directamente.
Sin embargo, las tres herramientas coinciden en señalar la misma función
como hotspot.

### 2.3 ¿Qué costo tiene exportar los archivos de reconstrucción?

Con `perf stat` se obtuvieron los siguientes tiempos:

| Configuración | Tiempo |
|---|---:|
| Normal | 43.63 s |
| `--export` | 48.99 s |

En estas ejecuciones, la configuración con exportación tardó
aproximadamente 5.36 s más, lo que corresponde a un incremento cercano
al 12.3 %.

Estos valores corresponden a ejecuciones independientes, por lo que no
deben interpretarse como un costo exacto y determinista de la exportación.

Callgrind también muestra un aumento del trabajo total:

| Configuración | Instrucciones (`Ir`) |
|---|---:|
| Normal | 203,308,860,828 |
| `--export` | 239,900,455,408 |

Esto representa un incremento aproximado del 18 % en las instrucciones
contabilizadas.

Además, al utilizar `--export` aparecen funciones relacionadas con el
formateo y la salida de datos, como `__printf_fp_l`, `hack_digit`,
`writev` y `std::__ostream_insert`.

Por lo tanto, la exportación introduce un costo adicional observable
tanto en tiempo de ejecución como en cantidad de trabajo realizado.

### 2.4 ¿Qué herramienta dio la evidencia más clara para decidir dónde optimizar?

Para estas mediciones, `perf` proporcionó la evidencia más directa para
identificar inicialmente el hotspot, ya que `perf report` mostró a
`GridIndex::nearest()` concentrando aproximadamente el 97.76 % Self
en la ejecución normal.

Callgrind complementó este resultado al mostrar que una gran cantidad
de las instrucciones ejecutadas se concentra en esta función y en las
estructuras de datos que utiliza.

Google Performance Tools permitió confirmar nuevamente el mismo hotspot
mediante una técnica de profiling diferente.

Por tanto, `perf` resultó especialmente claro para localizar inicialmente
la región donde concentrar el análisis, mientras que Callgrind y
gperftools aportaron evidencia adicional para confirmar el resultado.

---

## 3. Resultados de Milagro

Los archivos obtenidos se encuentran en la carpeta [`Milagro/`](./Milagro/).

Nota: en este equipo, `perf record`, Valgrind/Callgrind y Google Performance
Tools se ejecutaron únicamente en la configuración normal (sin `--export`).
Solo `perf stat` se corrió en ambas configuraciones.

### 3.1 Hotspots identificados

#### perf

En la ejecución normal, `perf report` identificó a `GridIndex::nearest()`
como el hotspot dominante, con **97.14 % Self** y **97.67 % Children**.
Dentro de la propia función, buena parte del tiempo se concentra en la
búsqueda dentro del `unordered_map` que indexa las celdas del grid
(`std::_Hashtable::find`/`_M_locate`).

Nota sobre limitaciones del equipo: este procesador es híbrido (núcleos
de rendimiento `cpu_core` y de eficiencia `cpu_atom`). El proceso corrió
completo en un núcleo `cpu_atom`, por lo que los contadores de `cpu_core`
aparecieron como `<not counted>`. Tampoco fue posible obtener
cache-references/cache-misses con `perf stat` por defecto en este equipo.

#### Google Performance Tools

Se obtuvieron 3489 muestras (`interrupts/evictions/bytes =
3489/890/72232`). `GridIndex::nearest()` representó:

- 71.91 % de las muestras directas (`flat`).
- 97.94 % de las muestras acumuladas (`cum`).

El resto del tiempo se reparte en funciones asociadas a la tabla hash
usada internamente por `GridIndex::nearest()`: `std::_Hashtable::
_M_find_before_node` (20.29 % cum), `std::__detail::_Mod_range_hashing::
operator()` (3.98 %) y `std::equal_to::operator()` (3.87 %).

#### Valgrind / Callgrind

Callgrind contabilizó 205,733,272,629 instrucciones (`Ir`) totales en la
ejecución normal.

| Origen | Ir | % |
|---|---:|---:|
| `GridIndex::nearest` (cuerpo propio) | 166,674,287,838 | 81.01 % |
| `stl_vector.h` | 20,804,515,668 | 10.11 % |
| `hashtable.h` | 7,290,929,053 | 3.54 % |
| `stl_algobase.h` | 3,680,330,298 | 1.79 % |
| `hashtable_policy.h` | 3,484,986,660 | 1.69 % |
| `stl_function.h` | 617,513,896 | 0.30 % |
| `stl_iterator.h` | 214,381,544 | 0.10 % |
| **Total atribuible a `nearest`** | **~202,767 M** | **~98.56 %** |

### 3.2 ¿Coinciden los resultados de las tres herramientas?

Sí. Las tres herramientas señalan a `GridIndex::nearest()` como la región
dominante de la ejecución, con porcentajes acumulados muy similares entre
sí (97.67 % en `perf`, 97.94 % en gperftools, 98.56 % en Callgrind).

Las pequeñas diferencias se explican por cómo mide cada herramienta:
`perf` usa muestreo estadístico basado en ciclos de CPU; gperftools usa
muestreo por interrupciones de temporizador (menos muestras: solo 3489);
y Callgrind usa instrumentación exhaustiva contando instrucciones
ejecutadas (`Ir`), sin muestreo, por lo que su resultado es determinista
pero corre bajo una carga artificial mucho más lenta. Aun así, las tres
apuntan a la misma función y, dentro de ella, al mismo mecanismo interno:
la búsqueda en el `unordered_map` de celdas del grid.

### 3.3 ¿Qué costo tiene exportar los archivos de reconstrucción?

Con `perf stat` se obtuvieron los siguientes tiempos:

| Configuración | Tiempo elapsed | Instrucciones (`cpu_atom`) |
|---|---:|---:|
| Normal | 33.52 s | 206,149,647,332 |
| `--export` | 36.60 s | 238,279,633,152 |

La configuración con exportación tardó aproximadamente 3.08 s más
(+9.17 %) y ejecutó cerca de 32,130 millones de instrucciones adicionales
(+15.58 %). Esa diferencia corresponde al trabajo de escribir los CSV y
los 46 archivos `.ppm` de reconstrucción.

Estos valores corresponden a ejecuciones independientes de un proceso de
~33-37 segundos, por lo que no deben interpretarse como una medición
perfectamente determinista, aunque la magnitud del incremento es
consistente con el trabajo adicional de E/S que realiza `--export`.

### 3.4 ¿Qué herramienta dio la evidencia más clara para decidir dónde optimizar?

`perf` fue la herramienta más rápida y directa para identificar el
hotspot: sin recompilar nada, `perf report` mostró de inmediato que
`GridIndex::nearest()` concentraba el 97 % del tiempo.

Callgrind aportó el nivel de detalle más profundo, desglosando el tiempo
dentro de `nearest()` por archivo de cabecera (`stl_vector.h`,
`hashtable.h`, etc.), lo que permite ver exactamente qué operación
interna (la búsqueda en el `unordered_map`) es la más costosa —
información que `perf` no mostró con el mismo nivel de granularidad.

gperftools, con solo 3489 muestras, confirmó el mismo resultado con una
tercera técnica independiente, aunque con menor resolución que las otras
dos por tener muchas menos muestras.

En conjunto: `perf` para localizar rápido dónde mirar, y Callgrind para
entender con precisión qué parte de esa función es costosa y por qué.

---

## 4. Resultados de Brayan

Los archivos obtenidos se encuentran en la carpeta [`Brayan/`](./Brayan/).

### 4.1 Hotspots identificados

#### Perf

En la ejecución normal, 'perf report' identificó a 'GridIndex::nearest()' como el
hotspot dominante con **97.74 % Children** (tiempo de muestreo acumulado más lo que la 
función llama) y **96.72 % Self (tiempo solo de la función, sin llamadas internas) **.

En la ejecución con '--export', el árbol de llamadas muestra:

```text
main
├── compare_profiles() → nearest_neighbor_distances() → GridIndex::nearest()   (57.34%)
├── GridIndex::nearest()  (llamada directa)                                    (29.03%)
└── export_reconstruction()                                                    (11.24%)
    └── write_cloud_csv() → formateo de doubles (ostream/printf_fp)            (9.91%)
```

Esto quiere decir que entre ambas rutas de llamada, 'GridIndex::nearest()' sigue concentrando
hasta un **86 %** de los ciclos mientras que la exportación añade un **11 %** adicional.

#### Google Performance Tools

En la ejecución normal se obtuvieron 3375 muestras. 'GridIndex::nearest()' representó:

- 94.9 % de las muestras dentro de la funcion misma (`flat`).
- 97.0 % de las muestras `flat` más subárbol de llamadas  (`cum`).

Con '--export' fueron 3730 muestras. 'GridIndex::nearest()' bajó levemente 86.9 % `flat` / 89.1 % `cum`
`export reconstruction` aparece con 8.7 % `cum` el cual tiene su reparto en otras funciones como
`write_cloud_csv`, `std::num_put::_M_insert_float` y `std::ostream::_M_insert`.

Un detalle a mencionar, es que en este equipo `google-pprof` no logró resolver los símbolos del binario 
original, por lo que se compiló con la bandera `-no-pie` para obtener legibilidad. Entonces, estas
muestras provienen de un binario distinto a las otras, pero el comportamiento del programa y el hotspot
no se ven realmente afectados.

#### Valgrind / Callgrind

Para este perfilado solo se usó la ejecución de `point_cloud_colimation` normal, la cual reveló hasta
207,961,170,394 instrucciones totales. Esta es la tabla que engloba el código asociado a 'GridIndex::nearest()',
que incluye el trabajo dentro de la función. 

| Origen | Ir | % |
|---|---:|---:|
| `GridIndex::nearest` (cuerpo propio) | 168,662,797,252 | 81.10 % |
| `stl_vector.h` | 20,804,515,668 | 10.00 % |
| `hashtable.h` | 6,421,932,315 | 3.09 % |
| `hashtable_policy.h` | 4,139,601,854 | 1.99 % |
| `stl_algobase.h` | 3,680,330,298 | 1.77 % |
| `stl_function.h` | 617,513,896 | 0.30 % |
| `stl_iterator.h` | 214,381,544 | 0.10 % |
| **Total atribuible a `nearest`** | **~204,541 M** | **~98.35 %** |

### 4.2 ¿Coinciden los resultados de las tres herramientas?

Sí, se puede observar que `GridIndex::nearest()` es la región dominante en las 3 herramientas. Las diferencias
solo son producto de la ejecución diferente en el cálculo de cada herramienta.

### 4.3 ¿Qué costo tiene exportar los archivos de reconstrucción?

Genera más muestras y el tiempo de ejecución sube levemente debido a todas las nuevas instrucciones respecto a la
ejecución normal que deben aplicarse para la obtención de todos los archivos generados.

### 4.4 ¿Qué herramienta dio la evidencia más clara para decidir dónde optimizar?

Considero que fue `perf` principalmente. Es la herramienta más directa y a la que personalmente estoy acostumbrado. Tanto con
`perf` como con `GPT` se puede ver de forma muy apreciable cuál es el hotspot para tomar la decisión de qué optimizar. Valgrind
por su parte es un poco más complejo pero no le quita utilidad y más bien ofrece una mayor resolución y profundidad a la búsqueda
de funciones poco eficientes.

---

## 5. Resultados de Alejandro

Los archivos obtenidos se encuentran en la carpeta [`Alejandro/`](./Alejandro/).

Nota: `perf stat` y `perf record` se ejecutaron con y sin `--export`. Valgrind/Callgrind y Google Performance Tools se ejecutaron únicamente sin exportación, siguiendo la aclaración del profesor.

### 5.1 Hotspots identificados

#### perf

En la ejecución normal, `perf report` identificó a `GridIndex::nearest()` como el hotspot principal, con **97.12 % Self**. Con `--export`, la misma función continúa concentrando la mayor parte del trabajo, con **86.08 % Self**.

Estos resultados corresponden al evento `cpu_core/cycles/P`, que reunió aproximadamente 92 mil muestras en la ejecución normal y 104 mil con exportación.

Este procesador es híbrido y el reporte separa los eventos de `cpu_core` y `cpu_atom`. En estas ejecuciones, la mayor parte de las muestras quedó en `cpu_core`. No se fijó el programa a un CPU específico. Algunas cadenas de llamadas mostraron direcciones sin resolver. Por eso se utilizó la columna `Self` para identificar el hotspot, sin interpretar el árbol completo como una reconstrucción confiable de las llamadas.

#### Google Performance Tools

En la ejecución normal se obtuvieron **2322 muestras**. `GridIndex::nearest()` representó:

- **90.9 %** de las muestras directas (`flat`).
- **97.8 %** de las muestras acumuladas (`cum`).

También aparecen funciones relacionadas con las estructuras utilizadas por `GridIndex::nearest()`, entre ellas:

- `std::vector::operator[]`
- `GridIndex::key`
- `GridIndex::cell_of`
- `std::_Hashtable::find`

Para utilizar esta herramienta se enlazó `libprofiler` y se compiló con `-fno-omit-frame-pointer` y `-no-pie`. La opción `-no-pie` permitió mostrar los nombres de las funciones, ya que en un intento anterior aparecían direcciones. Estas muestras provienen de una compilación distinta de la usada con `perf` y Callgrind.

#### Valgrind / Callgrind

Callgrind contabilizó **207,961,183,472 instrucciones (`Ir`)** totales en la ejecución normal.

El trabajo atribuido a `GridIndex::nearest()` se distribuyó de la siguiente manera:

| Origen | Ir | % |
|---|---:|---:|
| `point_cloud_collimation.cpp` | 168,662,797,252 | 81.10 % |
| `stl_vector.h` | 20,804,515,668 | 10.00 % |
| `hashtable.h` | 6,421,932,315 | 3.09 % |
| `hashtable_policy.h` | 4,139,601,854 | 1.99 % |
| `stl_algobase.h` | 3,680,330,298 | 1.77 % |
| `stl_function.h` | 617,513,896 | 0.30 % |
| `stl_iterator.h` | 214,381,544 | 0.10 % |
| **Total de las filas de `nearest`** | **204,541,072,827** | **98.35 %** |

Estos valores representan instrucciones ejecutadas, no tiempo.
El desglose muestra que parte del trabajo de la función aparece atribuido a los encabezados de vectores y tablas hash.

### 5.2 ¿Coinciden los resultados de las tres herramientas?

Sí. Las tres herramientas identifican `GridIndex::nearest()` como la función que concentra la mayor parte del trabajo. Esto señala la búsqueda del vecino más cercano como el primer lugar donde conviene revisar una posible optimización. Los porcentajes cambian porque las herramientas miden de forma distinta. `perf` muestrea ciclos de CPU, Google Performance Tools muestrea el uso del CPU y Callgrind cuenta instrucciones mediante instrumentación.

Además, el código insertado por el compilador puede aparecer repartido entre distintas funciones o archivos. Por eso la coincidencia está en el hotspot identificado, aunque los porcentajes no sean iguales.

### 5.3 ¿Qué costo tiene exportar los archivos de reconstrucción?

Con `perf stat` se obtuvieron los siguientes resultados:

| Configuración | Tiempo elapsed | Instrucciones (`cpu_core`) |
|---|---:|---:|
| Normal | 22.9617 s | 208,483,812,663 |
| `--export` | 25.3931 s | 241,538,927,969 |

La ejecución con exportación tardó aproximadamente **2.4315 s más**, lo que corresponde a un incremento del **10.59 %**.

Los eventos de caché se midieron en ejecuciones separadas mediante `perf stat -e cache-references,cache-misses`:

| Contador (`cpu_core`) | Normal | `--export` |
|---|---:|---:|
| Referencias a caché | 1,793,882,697 | 1,625,925,466 |
| Fallos de caché | 29,623,229 | 40,047,712 |

Los contadores escalados de `cpu_core` y `cpu_atom` se mantuvieron separados; no se sumaron entre sí.

En el perfil con exportación aparecen funciones relacionadas con la escritura y el formateo de datos, como `write_cloud_csv`, `std::ostream::_M_insert` y `std::num_put::_M_insert_float`. Ese trabajo adicional ayuda a explicar el aumento del tiempo y que el porcentaje relativo de `GridIndex::nearest()` disminuya con `--export`. Esto no significa que la búsqueda de vecinos se haya acelerado.

Los resultados corresponden a ejecuciones individuales, por lo que la diferencia de tiempo no debe interpretarse como un costo fijo.
Tampoco se puede atribuir toda la variación de los contadores de caché únicamente a la exportación.

### 5.4 ¿Qué herramienta dio la evidencia más clara para decidir dónde optimizar?

`perf` dio la evidencia más directa para identificar el hotspot: `GridIndex::nearest()` concentró el **97.12 % Self** en la ejecución normal. Con ese resultado ya se tiene una función concreta donde
empezar a revisar. Callgrind complementó el análisis al mostrar cómo se distribuyen las instrucciones entre el código de esa función y los encabezados de las estructuras que utiliza. Google Performance Tools confirmó el mismo resultado mediante muestreo del CPU. Las tres herramientas apuntan a la búsqueda de vecinos, aunque todavía hace falta revisar el código y el ensamblador
para decidir qué cambio aplicar.

---

## 6. Comparación entre equipos

Una vez obtenidos los resultados de todos los integrantes, en esta sección se compararán las mediciones considerando las diferencias de hardware.

| Métrica | Angie | Milagro | Brayan | Alejandro |
|---|---:|---:|---:|---:|
| Tiempo normal (s) | 43.63 | 33.52 | Pendiente | 22.9617 |
| Tiempo `--export` (s) | 48.99 | 36.60 | Pendiente | 25.3931 |
| Incremento de tiempo (%) | 12.3 | 9.17 | Pendiente | 10.59 |
| Hotspot principal (`perf`) | `GridIndex::nearest()` | `GridIndex::nearest()` | Pendiente | `GridIndex::nearest()` |
| Hotspot principal (gperftools) | `GridIndex::nearest()` | `GridIndex::nearest()` | Pendiente | `GridIndex::nearest()` |
| Hotspot principal (Callgrind) | `GridIndex::nearest()` | `GridIndex::nearest()` | Pendiente | `GridIndex::nearest()` |
Esta comparación permitirá determinar qué características del comportamiento
son consistentes entre diferentes equipos y cuáles dependen del hardware
utilizado.
