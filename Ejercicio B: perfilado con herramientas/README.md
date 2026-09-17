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
| Procesador | Intel Core i5-10210U | Intel Core i7-12700H | AMD Ryzen 7 5700U with Radeon Graphics | Pendiente |
| Núcleos / hilos | 4 / 8 | 14 / 20 | 8 / 16 | Pendiente |
| Memoria RAM | 16 GB | 16 GB | 16 GB | Pendiente |
| Sistema operativo | Ubuntu 22.04.5 LTS | Ubuntu 26.04 LTS | Ubuntu 24.04.5 LTS| Pendiente |
| Arquitectura | x86-64 | x86-64 | Px86-64 | Pendiente |
| Kernel | 6.8.0-138-generic | 7.0.0-31-generic | 	7.0.0-31-generic | Pendiente |
| Compilador | GCC/G++ 11.4 | GCC/G++ 15.2.0 | GCC/G++ 13.3.0 | Pendiente |

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

Pendiente.

### 4.3 ¿Qué costo tiene exportar los archivos de reconstrucción?

Pendiente.

### 4.4 ¿Qué herramienta dio la evidencia más clara para decidir dónde optimizar?

Pendiente.

---

## 5. Resultados de Alejandro

Los archivos obtenidos se encuentran en la carpeta [`Alejandro/`](./Alejandro/).

### 5.1 Hotspots identificados

**perf:** Pendiente.

**Google Performance Tools:** Pendiente.

**Valgrind / Callgrind:** Pendiente.

### 5.2 ¿Coinciden los resultados de las tres herramientas?

Pendiente.

### 5.3 ¿Qué costo tiene exportar los archivos de reconstrucción?

Pendiente.

### 5.4 ¿Qué herramienta dio la evidencia más clara para decidir dónde optimizar?

Pendiente.

---

## 6. Comparación entre equipos

Una vez obtenidos los resultados de todos los integrantes, en esta sección
se compararán las mediciones considerando las diferencias de hardware.

| Métrica | Angie | Milagro | Brayan | Alejandro |
|---|---:|---:|---:|---:|
| Tiempo normal (s) | 43.63 | 33.52 | Pendiente | Pendiente |
| Tiempo `--export` (s) | 48.99 | 36.60 | Pendiente | Pendiente |
| Incremento de tiempo (%) | 12.3 | 9.17 | Pendiente | Pendiente |
| Hotspot principal (`perf`) | `GridIndex::nearest()` | `GridIndex::nearest()` | Pendiente | Pendiente |
| Hotspot principal (gperftools) | `GridIndex::nearest()` | `GridIndex::nearest()` | Pendiente | Pendiente |
| Hotspot principal (Callgrind) | `GridIndex::nearest()` | `GridIndex::nearest()` | Pendiente | Pendiente |

Esta comparación permitirá determinar qué características del comportamiento
son consistentes entre diferentes equipos y cuáles dependen del hardware
utilizado.
