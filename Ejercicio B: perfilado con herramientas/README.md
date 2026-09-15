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
| Procesador | Intel Core i5-10210U | Pendiente | Pendiente | Pendiente |
| Núcleos / hilos | 4 / 8 | Pendiente | Pendiente | Pendiente |
| Memoria RAM | 16 GB | Pendiente | Pendiente | Pendiente |
| Sistema operativo | Ubuntu 22.04.5 LTS | Pendiente | Pendiente | Pendiente |
| Arquitectura | x86-64 | Pendiente | Pendiente | Pendiente |
| Kernel | 6.8.0-138-generic | Pendiente | Pendiente | Pendiente |
| Compilador | GCC/G++ 11.4 | Pendiente | Pendiente | Pendiente |

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

### 3.1 Hotspots identificados

**perf:** Pendiente.

**Google Performance Tools:** Pendiente.

**Valgrind / Callgrind:** Pendiente.

### 3.2 ¿Coinciden los resultados de las tres herramientas?

Pendiente.

### 3.3 ¿Qué costo tiene exportar los archivos de reconstrucción?

Pendiente.

### 3.4 ¿Qué herramienta dio la evidencia más clara para decidir dónde optimizar?

Pendiente.

---

## 4. Resultados de Brayan

Los archivos obtenidos se encuentran en la carpeta [`Brayan/`](./Brayan/).

### 4.1 Hotspots identificados

**perf:** Pendiente.

**Google Performance Tools:** Pendiente.

**Valgrind / Callgrind:** Pendiente.

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
| Tiempo normal (s) | 43.63 | Pendiente | Pendiente | Pendiente |
| Tiempo `--export` (s) | 48.99 | Pendiente | Pendiente | Pendiente |
| Incremento de tiempo (%) | 12.3 | Pendiente | Pendiente | Pendiente |
| Hotspot principal (`perf`) | `GridIndex::nearest()` | Pendiente | Pendiente | Pendiente |
| Hotspot principal (gperftools) | `GridIndex::nearest()` | Pendiente | Pendiente | Pendiente |
| Hotspot principal (Callgrind) | `GridIndex::nearest()` | Pendiente | Pendiente | Pendiente |

Esta comparación permitirá determinar qué características del comportamiento
son consistentes entre diferentes equipos y cuáles dependen del hardware
utilizado.
