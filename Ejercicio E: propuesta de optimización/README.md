# Ejercicio E: propuesta de optimización


## Especificaciones de los equipos

| Especificación | Angie | Milagro | Brayan | Alejandro |
|---|---|---|---|---|
| Procesador | Intel Core i5-10210U | Intel Core i7-12700H | AMD Ryzen 7 5700U with Radeon Graphics | Intel Core i9-13900HX |
| Núcleos / hilos | 4 / 8 | 14 / 20 | 8 / 16 | 24 / 32 |
| Memoria RAM | 16 GB | 16 GB | 16 GB | 32 GB |
| Sistema operativo | Ubuntu 22.04.5 LTS | Ubuntu 26.04 LTS | Ubuntu 24.04.5 LTS | Ubuntu 24.04.5 LTS |
| Arquitectura | x86-64 | x86-64 | x86-64 | x86-64 |
| Kernel | 6.8.0-138-generic | 7.0.0-31-generic | 7.0.0-31-generic | 7.0.0-31-generic |
| Compilador | GCC/G++ 11.4 | GCC/G++ 15.2.0 | GCC/G++ 13.3.0 | GCC/G++ 13.3.0 |

Los tiempos absolutos deben interpretarse considerando las diferencias de
hardware y software entre los equipos. Para evaluar la optimización se compara
principalmente cada versión modificada contra su versión original en el mismo
equipo.

---

## Optimización: tamaño de celda de `GridIndex`

### Cambio realizado

Se modificó el programa para definir el tamaño de celda de `GridIndex` mediante
una constante:

```cpp
constexpr double GRID_CELL_SIZE = 90.0;
```

Esto permite evaluar diferentes tamaños de celda manteniendo sin cambios el
resto del algoritmo.

### Hipótesis

El tamaño de celda afecta la búsqueda de vecinos más cercanos. Una celda
pequeña contiene menos puntos, pero puede requerir consultar más celdas. Una
celda grande reduce la cantidad de celdas consultadas, pero puede aumentar la
cantidad de candidatos evaluados.

Se evaluó si modificar el valor original de `90.0` podía reducir el tiempo de
búsqueda.

### Metodología

El programa se compiló con:

```bash
make clean
make CXXFLAGS="-std=c++17 -O2 -g -Wall -Wextra -pedantic -fno-omit-frame-pointer"
```

Para el valor original (`90.0`) se realizaron cinco ejecuciones:

```bash
for i in {1..5}; do
    echo "=== RUN $i ===" | tee <nombre>_E_grid90_run${i}.txt
    ./point_cloud_collimation 2>&1 | tee -a <nombre>_E_grid90_run${i}.txt
done
```

También se realizaron pruebas exploratorias con tamaños de celda de `45.0`,
`60.0`, `120.0` y `180.0`.

Se analizaron principalmente las regiones instrumentadas
`nearest_neighbors` y `profile_metrics`. Además, se verificaron las iteraciones
y el `profile_score` final.

---

## Resultados de Angie

| Tamaño de celda | `nearest_neighbors` (ms) | `profile_metrics` (ms) | Iteraciones | `profile_score` |
|---:|---:|---:|---:|---:|
| 45.0 | 409.933 | 837.756 | 45 | 0.01847086 |
| 60.0 | 400.667 | 741.867 | — | — |
| **90.0 (original)** | **340.733** | **681.182** | **45** | **0.01847086** |
| 120.0 | 412.867 | 835.200 | 45 | 0.01847086 |
| 180.0 | 584.222 | 1214.578 | 45 | 0.01847086 |

El valor de `90.0` corresponde al promedio de cinco ejecuciones. Los demás
valores corresponden a pruebas exploratorias individuales.

### Conclusión

Los tamaños evaluados no mejoraron el rendimiento respecto al valor original
de `90.0`.

Los resultados son consistentes con un compromiso entre consultar más celdas
cuando estas son pequeñas y evaluar más candidatos cuando son grandes. Para
los valores probados, `90.0` presentó el menor tiempo.

Por lo tanto, la hipótesis de mejorar el rendimiento modificando el tamaño de
celda **no se confirmó para los valores evaluados**.

---

## Comparación grupal

| Integrante | Tamaño original | Tamaños evaluados | Mejor valor observado | `nearest_neighbors` original (ms) | `nearest_neighbors` mejor (ms) | Resultado preservado |
|---|---:|---|---:|---:|---:|---|
| Angie | 90.0 | 45, 60, 120, 180 | 90.0 | 340.733 | 340.733 | Sí* |
| Milagro | 90.0 | — | — | — | — | — |
| Brayan | 90.0 | — | — | — | — | — |
| Alejandro | 90.0 | — | — | — | — | — |

\* Verificado en los tamaños para los cuales se registraron explícitamente las
iteraciones y el `profile_score`.

---

## Evidencia con herramientas

La instrumentación manual con `std::chrono` se utilizó para medir directamente
las regiones afectadas por la optimización.

La comparación con `perf` se agregará para contrastar el programa original y
la versión modificada utilizando una segunda técnica de perfilado.