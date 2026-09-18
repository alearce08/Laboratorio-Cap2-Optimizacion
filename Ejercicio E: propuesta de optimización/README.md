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

## Optimización: tamaño de celda de `GridIndex`

### Cambio realizado

Se modificó el tamaño de celda utilizado por `GridIndex`. El programa original
utiliza:

```cpp
constexpr double GRID_CELL_SIZE = 90.0;
```

Se probaron de forma exploratoria los valores `45.0`, `60.0`, `120.0` y
`180.0`. Para la comparación final se seleccionó `60.0`, por ser el valor
alternativo que presentó el menor tiempo entre los tamaños modificados
evaluados.

---

### Hipótesis

El tamaño de celda afecta el costo de la búsqueda de vecinos más cercanos.

Una celda pequeña contiene menos puntos candidatos, pero puede requerir
consultar más celdas. Una celda grande puede reducir la cantidad de celdas
consultadas, pero aumenta la cantidad de candidatos por celda.

La hipótesis fue que modificar el tamaño original de `90.0` podía reducir el
tiempo de búsqueda.

---

## Metodología

Se comparó el programa original (`90.0`) contra la versión modificada (`60.0`)
utilizando dos técnicas:

1. Instrumentación manual con `std::chrono`.
2. `perf stat`.

El programa se compiló con:

```bash
make clean
make CXXFLAGS="-std=c++17 -O2 -g -Wall -Wextra -pedantic -fno-omit-frame-pointer"
```

Para la instrumentación manual se realizaron cinco ejecuciones por versión:

```bash
for i in {1..5}; do
    echo "=== RUN $i ===" | tee <nombre>_E_grid90_run${i}.txt
    ./point_cloud_collimation 2>&1 | tee -a <nombre>_E_grid90_run${i}.txt
done
```

Para `perf` se utilizó:

```bash
sudo perf stat ./point_cloud_collimation 2>&1 | tee <nombre>_E_perf_grid90.txt
```

Los comandos se repiten cambiando `grid90` por `grid60` para la versión
modificada.

---

## Resultados de Angie

### Instrumentación manual

Los valores corresponden al promedio de cinco ejecuciones.

| Región | 90.0 original (ms) | 60.0 modificado (ms) | Variación |
|---|---:|---:|---:|
| `nearest_neighbors` | 340.733 | 390.102 | +14.49 % |
| `profile_metrics` | 681.182 | 716.476 | +5.18 % |
| `grid_construction` | 3.800 | 4.400 | +15.79 % |

Las cinco ejecuciones de ambas versiones finalizaron con 45 iteraciones y
`profile_score = 0.01847086`.

### `perf stat`

| Métrica | 90.0 original | 60.0 modificado | Variación |
|---|---:|---:|---:|
| Tiempo total | 44.455 s | 56.451 s | +26.99 % |
| Ciclos | 167,689,016,613 | 202,549,739,371 | +20.79 % |
| Instrucciones | 203,917,947,016 | 177,269,800,206 | -13.07 % |
| IPC | 1.22 | 0.88 | -27.87 % |
| Branch misses | 693,761,054 | 1,111,822,866 | +60.26 % |
| Branch-miss rate | 2.60 % | 4.34 % | +1.74 pp |

Aunque la versión con `60.0` ejecutó menos instrucciones, necesitó más ciclos,
presentó menor IPC y una mayor proporción de fallos de predicción de saltos.

---

## Resultados de Milagro

| Tamaño de celda | `nearest_neighbors` (ms) | `profile_metrics` (ms) | Iteraciones | `profile_score` |
|---:|---:|---:|---:|---:|
| 45.0 | 302.379 | 616.797 | 45 | 0.01847086 |
| 60.0 | 284.824 | 527.213 | 45 | 0.01847086 |
| **90.0 (original)** | **250.712** | **507.303** | **45** | **0.01847086** |
| 120.0 | 298.300 | 611.272 | 45 | 0.01847086 |
| 180.0 | 434.721 | 902.340 | 45 | 0.01847086 |

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

## Conclusión

La hipótesis no se confirmó. Reducir `GRID_CELL_SIZE` de `90.0` a `60.0`
mantuvo el resultado de la colimación, pero aumentó el tiempo de ejecución.

La instrumentación manual mostró un aumento de 14.49 % en
`nearest_neighbors`, mientras que `perf` mostró un aumento de 26.99 % en el
tiempo total.

Para este caso, el tamaño original de `90.0` presentó mejor rendimiento que
los tamaños alternativos evaluados.

---

## Comparación grupal

| Integrante | Original | Modificado | `nearest_neighbors` original (ms) | `nearest_neighbors` modificado (ms) | Tiempo `perf` original (s) | Tiempo `perf` modificado (s) | Hipótesis confirmada |
|---|---:|---:|---:|---:|---:|---:|---|
| Angie | 90.0 | 60.0 | 340.733 | 390.102 | 44.455 | 56.451 | No |
| Milagro | 90.0 | — | — | — | — | — | — |
| Brayan | 90.0 | — | — | — | — | — | — |
| Alejandro | 90.0 | — | — | — | — | — | — |
