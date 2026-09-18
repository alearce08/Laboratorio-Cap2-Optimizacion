# Ejercicio D: perfilado mediante instrumentación

En este ejercicio agregamos temporizadores con `std::chrono` para medir por separado las partes principales del programa. Cada integrante ejecuta el código instrumentado en su equipo y agrega sus resultados en su propia carpeta.

El código utilizado se encuentra en [point_cloud_collimation_instrumentado.cpp](point_cloud_collimation_instrumentado.cpp). La instrumentación registra los tiempos en formato CSV mediante `stderr`, mientras que la salida normal del programa se mantiene en `stdout`.

## 1. Regiones instrumentadas

| Nombre en el CSV | Región medida |
|---|---|
| `generate_target` | Generación del perfil objetivo. |
| `prepare_source` | Transformación, deformación y ruido del perfil fuente. |
| `grid_index_icp` | Construcción del índice fijo utilizado por ICP. |
| `nearest_neighbors` | Búsqueda de vecinos dentro de cada iteración. |
| `estimate_transform` | Estimación de la transformación rígida. |
| `profile_metrics` | Cálculo de las métricas de comparación. |
| `render_motion_frame` | Renderizado de cada cuadro cuando se utiliza `--export`. |
| `export_reconstruction` | Exportación completa de los CSV y cuadros PPM. |

Las ejecuciones normales miden las primeras seis regiones. Los temporizadores de renderizado y exportación solo se activan cuando el programa se ejecuta con `--export`.

## 2. Compilación y comandos

Compilamos con las opciones indicadas en el enunciado:

```bash
make clean
make CXXFLAGS="-std=c++17 -O2 -g -Wall -Wextra -pedantic -fno-omit-frame-pointer"
```

Para guardar por separado la salida normal del programa y las mediciones:

```bash
./point_cloud_collimation \
  > "Ejercicio D: perfilado mediante instrumentación/Alejandro/alejandro_d_normal.txt" \
  2> "Ejercicio D: perfilado mediante instrumentación/Alejandro/alejandro_d_normal.csv"
```

Se realizaron tres repeticiones del ejecutable original:

```bash
for i in 1 2 3; do
  /usr/bin/time -f "%e" \
    -o "Ejercicio D: perfilado mediante instrumentación/Alejandro/alejandro_d_original_time_${i}.txt" \
    ./point_cloud_collimation_original \
    > /dev/null 2>&1
done
```

Después se hicieron tres repeticiones del ejecutable instrumentado:

```bash
for i in 1 2 3; do
  /usr/bin/time -f "%e" \
    -o "Ejercicio D: perfilado mediante instrumentación/Alejandro/alejandro_d_instrumentado_time_${i}.txt" \
    ./point_cloud_collimation \
    > /dev/null \
    2> "Ejercicio D: perfilado mediante instrumentación/Alejandro/alejandro_d_instrumentado_${i}.csv"
done
```

El promedio por región se calculó a partir de los tres CSV:

```bash
{
  echo "region,promedio_milliseconds"
  LC_ALL=C awk -F, '
  NR > 1 && $3 ~ /^[0-9]/ {
    suma[$1] += $3
    cantidad[$1]++
  }
  END {
    for (r in suma)
      printf "%s,%.6f\n", r, suma[r] / cantidad[r]
  }' "Ejercicio D: perfilado mediante instrumentación/Alejandro"/alejandro_d_instrumentado_*.csv \
  | sort -t, -k1,1
} > "Ejercicio D: perfilado mediante instrumentación/Alejandro/alejandro_d_promedios.csv"
```

Se utilizó `LC_ALL=C` para mantener el punto decimal y evitar que la configuración regional separara incorrectamente las columnas del CSV.

Los tiempos totales promedio se calcularon con:

```bash
LC_ALL=C awk '{s += $1; n++} END {printf "%.6f segundos\n", s/n}' \
  "Ejercicio D: perfilado mediante instrumentación/Alejandro"/alejandro_d_original_time_*.txt

LC_ALL=C awk '{s += $1; n++} END {printf "%.6f segundos\n", s/n}' \
  "Ejercicio D: perfilado mediante instrumentación/Alejandro"/alejandro_d_instrumentado_time_*.txt
```

## 3. Especificaciones de los equipos

| Especificación | Angie | Milagro | Brayan | Alejandro |
|---|---|---|---|---|
| Procesador | Intel Core i5-10210U | Intel Core i7-12700H | AMD Ryzen 7 5700U with Radeon Graphics | Intel Core i9-13900HX |
| Núcleos / hilos | 4 / 8 | 14 / 20 | 8 / 16 | 24 / 32 |
| Memoria RAM | 16 GB | 16 GB | 16 GB | 32 GB |
| Sistema operativo | Ubuntu 22.04.5 LTS | Ubuntu 26.04 LTS | Ubuntu 24.04.5 LTS | Ubuntu 24.04.5 LTS |
| Arquitectura | x86-64 | x86-64 | x86-64 | x86-64 |
| Kernel | 6.8.0-138-generic | 7.0.0-31-generic | 7.0.0-31-generic | 7.0.0-31-generic |
| Compilador | GCC/G++ 11.4 | GCC/G++ 15.2.0 | GCC/G++ 13.3.0 | GCC/G++ 13.3.0 |

Los resultados deben interpretarse considerando las diferencias de hardware y software entre los equipos.

## 4. Resultados de Alejandro

Las mediciones completas se encuentran en la carpeta [Alejandro/](Alejandro/). El resumen de los tiempos está en [alejandro_d_promedios.csv](Alejandro/alejandro_d_promedios.csv).

### 4.1 Tiempos por región

| Región | Llamadas por ejecución | Promedio por llamada | Tiempo acumulado estimado |
|---|---:|---:|---:|
| `generate_target` | 1 | 7.014 ms | 7.014 ms |
| `prepare_source` | 1 | 17.527 ms | 17.527 ms |
| `grid_index_icp` | 1 | 2.736 ms | 2.736 ms |
| `nearest_neighbors` | 45 | 161.651 ms | 7274.310 ms |
| `estimate_transform` | 45 | 0.381 ms | 17.143 ms |
| `profile_metrics` | 46 | 329.849 ms | 15173.035 ms |

`profile_metrics` tiene 46 mediciones porque se calcula una vez antes de iniciar ICP y otra vez en cada una de las 45 iteraciones. `nearest_neighbors` y `estimate_transform` se ejecutan una vez por iteración.

Las regiones medidas acumulan aproximadamente 22.492 segundos. La mayor parte del tiempo se concentra en `profile_metrics` y `nearest_neighbors`. Las demás regiones representan una fracción pequeña del tiempo total.

### 4.2 Comparación con las herramientas externas

La región con mayor tiempo manual fue `profile_metrics`, con aproximadamente 329.849 ms por llamada y 15.173 segundos acumulados. Este resultado coincide con las herramientas externas, aunque el nombre de la región medida sea diferente.

`compare_profiles`, medida dentro de `profile_metrics`, construye índices espaciales y realiza búsquedas de vecinos en las dos direcciones. Por eso incluye muchas llamadas a `GridIndex::nearest`.

En el ejercicio B, `GridIndex::nearest` registró:

- 97.12 % de costo propio en `perf`.
- 90.9 % de las muestras directas en Google Performance Tools.
- 81.10 % de las instrucciones de Callgrind asociadas directamente con su código.

La instrumentación manual agrupa el tiempo por etapa del programa, mientras que las herramientas externas atribuyen las muestras o instrucciones a la función interna que realiza el trabajo. Ambos métodos coinciden en que la búsqueda de vecinos es la principal fuente de costo.

### 4.3 Overhead de la instrumentación

| Versión | Tiempo promedio |
|---|---:|
| Original | 23.040 s |
| Instrumentada | 22.720 s |

La diferencia observada fue de -0.320 segundos, equivalente a -1.39 %. Esto no significa que los temporizadores aceleraran el programa. La diferencia es pequeña y corresponde a la variación normal entre ejecuciones.

Con estas pruebas no se detectó un overhead apreciable. Se considera cercano a 0 % frente al tiempo total del programa.

### 4.4 ¿Qué resulta más fácil de entender con instrumentación manual?

La instrumentación permite separar directamente el tiempo de generación, preparación de la nube fuente, construcción del índice, búsqueda de vecinos, estimación de la transformación y cálculo de métricas. También permite observar cuánto tarda cada iteración y calcular promedios por región sin tener que reconstruir el flujo del programa mediante un árbol de llamadas.

Fue especialmente útil para comprobar que `profile_metrics` tarda aproximadamente el doble que la búsqueda directa de vecinos de una iteración. Esto tiene sentido porque la comparación de perfiles busca vecinos en las dos direcciones y además calcula centroides, percentiles y otras métricas.

### 4.5 ¿Qué información no entrega la instrumentación manual?

Los temporizadores no indican qué instrucciones concentran el trabajo, cuántos fallos de caché o de predicción de saltos ocurren, ni cómo se distribuye el costo dentro de una función. Tampoco muestran automáticamente funciones de bibliotecas, llamadas del sistema o regiones que no fueron instrumentadas. Para responder esas preguntas siguen siendo necesarias herramientas como `perf`, Callgrind, Google Performance Tools y `perf annotate`.

## 5. Resultados de Angie

Las mediciones completas se encuentran en la carpeta [Angie/](Angie/). El resumen de los tiempos se encuentra en [angie_instrumentation_summary.csv](Angie/angie_instrumentation_summary.csv).

Se realizaron cinco repeticiones para obtener los promedios de cada región.

### 5.1 Tiempos por región

| Región | Muestras | Promedio |
|---|---:|---:|
| `target_generation` | 5 | 16.025 ms |
| `source_preparation` | 5 | 31.660 ms |
| `grid_construction` | 5 | 3.906 ms |
| `nearest_neighbors` | 225 | 315.466 ms |
| `rigid_transform` | 225 | 0.600 ms |
| `profile_metrics` | 225 | 635.408 ms |
| `file_export` | 5 | 5863.887 ms |
| `gstreamer` | 5 | 9455.041 ms |

Las regiones `nearest_neighbors`, `rigid_transform` y `profile_metrics` se midieron en cada una de las 45 iteraciones del algoritmo. Por esta razón se obtuvieron 225 muestras para cada región al realizar cinco ejecuciones.

Entre las regiones del algoritmo iterativo, `profile_metrics` presentó el mayor tiempo promedio, con 635.408 ms por iteración, seguido de `nearest_neighbors`, con 315.466 ms.

Los tiempos de `file_export` y `gstreamer` corresponden a fases completas de ejecución y no deben compararse directamente con los tiempos por iteración. Además, la medición de GStreamer incluye las esperas utilizadas por el visor.

### 5.2 Comparación con las herramientas externas

La región con mayor tiempo dentro del procesamiento iterativo fue `profile_metrics`, seguida de `nearest_neighbors`. Esto es consistente con los resultados obtenidos mediante las herramientas del ejercicio B.

En `perf`, `GridIndex::nearest` concentró aproximadamente 97.76 % del costo propio y 98.15 % considerando sus hijos. Google Performance Tools también identificó esta función como la principal región de trabajo, con 62.2 % de las muestras directas y 97.9 % de costo acumulado. En Callgrind, `GridIndex::nearest` representó aproximadamente 83.45 % de las instrucciones ejecutadas.

Aunque la instrumentación manual identifica `profile_metrics` como la región de mayor duración, esta función realiza internamente búsquedas de vecinos mediante `GridIndex::nearest`. Entonces ambos métodos muestran que la búsqueda de vecinos es la principal fuente de costo computacional.

### 5.3 Overhead de la instrumentación

| Versión | Tiempo promedio |
|---|---:|
| Original | 44.842 s |
| Instrumentada | 44.446 s |

La diferencia observada fue de -0.396 segundos, equivalente aproximadamente a -0.88 %. Esto no significa que la instrumentación acelere el programa, sino que la diferencia obtenida se encuentra dentro de la variabilidad observada entre las ejecuciones.
Con estas mediciones no se detectó un overhead apreciable causado por la instrumentación.

### 5.4 ¿Qué resulta más fácil de entender con instrumentación manual?

La instrumentación manual permite conocer directamente cuánto tarda cada etapa del algoritmo y observar su comportamiento en cada iteración. Permitió distinguir el tiempo utilizado por `nearest_neighbors`, `rigid_transform` y `profile_metrics`, además de medir de forma independiente las fases de exportación y visualización.

También facilita relacionar el tiempo medido con una etapa concreta del algoritmo sin depender únicamente del árbol de llamadas generado por las herramientas de perfilado.

### 5.5 ¿Qué información no entrega la instrumentación manual?

La instrumentación manual no permite determinar qué instrucciones específicas concentran el costo ni proporciona contadores de hardware como ciclos, IPC, fallos de caché o fallos de predicción de saltos.

Tampoco muestra automáticamente cómo se distribuye el costo dentro de una función o entre funciones internas que no fueron instrumentadas. Para obtener este nivel de detalle siguen siendo necesarias herramientas como `perf`, Google Performance Tools, Callgrind y `perf annotate`.

## 6. Resultados de Milagro

Las mediciones completas se encuentran en la carpeta [Milagro/](Milagro/). El resumen de los tiempos está en [milagro_d_promedios.csv](Milagro/milagro_d_promedios.csv).

Se realizaron tres repeticiones tanto del ejecutable original como del instrumentado.

### 6.1 Tiempos por región

| Región | Llamadas por ejecución | Promedio por llamada | Tiempo acumulado estimado |
|---|---:|---:|---:|
| `generate_target` | 1 | 9.803 ms | 9.803 ms |
| `prepare_source` | 1 | 27.655 ms | 27.655 ms |
| `grid_index_icp` | 1 | 3.090 ms | 3.090 ms |
| `nearest_neighbors` | 45 | 250.452 ms | 11270.333 ms |
| `estimate_transform` | 45 | 0.332 ms | 14.952 ms |
| `profile_metrics` | 46 | 509.280 ms | 23426.866 ms |

`profile_metrics` tiene 46 mediciones porque se calcula una vez antes de iniciar ICP y otra vez en cada una de las 45 iteraciones. `nearest_neighbors` y `estimate_transform` se ejecutan una vez por iteración.

Las regiones medidas acumulan aproximadamente 34.75 segundos, prácticamente el mismo valor que el tiempo total promedio medido para el ejecutable instrumentado (35.11 s), lo que indica que casi todo el tiempo de ejecución queda cubierto por las regiones instrumentadas. La mayor parte se concentra en `profile_metrics`, seguida de `nearest_neighbors`. Las demás regiones representan una fracción pequeña del tiempo total.

### 6.2 Comparación con las herramientas externas

La región con mayor tiempo manual fue `profile_metrics`, con aproximadamente 509.28 ms por llamada y 23.43 segundos acumulados, seguida de `nearest_neighbors`, con 250.45 ms por llamada y 11.27 segundos acumulados.

`profile_metrics` corresponde al cálculo de `compare_profiles`, que construye índices espaciales y busca vecinos en ambas direcciones (por eso incluye internamente muchas llamadas a `GridIndex::nearest`, además de calcular centroides y percentiles). `nearest_neighbors` mide directamente la búsqueda de correspondencias que usa ICP en cada iteración, la cual también recae en `GridIndex::nearest`.

En el ejercicio B, `GridIndex::nearest` concentró:

- 97.67 % de costo acumulado en `perf`.
- 97.94 % de costo acumulado en Google Performance Tools.
- 98.56 % de las instrucciones en Callgrind.

La instrumentación manual agrupa el tiempo por etapa del programa (`profile_metrics`, `nearest_neighbors`, etc.), mientras que las herramientas externas atribuyen el costo directamente a la función `GridIndex::nearest` que ambas etapas invocan. Ambos métodos coinciden en que la búsqueda de vecinos es la principal fuente de costo, sea medida como una sola función o repartida entre las dos etapas que la utilizan.

### 6.3 Overhead de la instrumentación

| Versión | Tiempo promedio |
|---|---:|
| Original | 34.753 s |
| Instrumentada | 35.110 s |

La diferencia observada fue de +0.357 segundos, equivalente a aproximadamente +1.03 %. Esta diferencia es pequeña y se encuentra dentro de la variación normal esperada entre ejecuciones; no representa un overhead apreciable introducido por los temporizadores de `std::chrono`.

### 6.4 ¿Qué resulta más fácil de entender con instrumentación manual?

La instrumentación permite separar directamente cuánto tiempo toma cada etapa del programa (generación del perfil, preparación de la nube fuente, construcción del índice, búsqueda de vecinos, estimación de la transformación y cálculo de métricas) sin tener que interpretar un árbol de llamadas. También permite ver cómo cambia el tiempo de cada región a través de las iteraciones, algo que no se obtiene directamente de una sola corrida de `perf report` o Callgrind.

Fue útil para confirmar que `profile_metrics` toma alrededor del doble de tiempo que `nearest_neighbors` en cada iteración, lo cual es consistente con que `profile_metrics` recorre las correspondencias en ambas direcciones y calcula métricas adicionales (centroides, percentiles), mientras que `nearest_neighbors` solo hace la búsqueda en una dirección para ICP.

### 6.5 ¿Qué información no entrega la instrumentación manual?

Los temporizadores manuales no indican qué instrucciones concentran el trabajo dentro de una región, ni entregan contadores de hardware como ciclos, instrucciones por ciclo, fallos de caché o fallos de predicción de saltos. Tampoco muestran automáticamente el costo de funciones de biblioteca o de código que no fue envuelto explícitamente con temporizadores. Para ese nivel de detalle siguen siendo necesarias herramientas como `perf`, Google Performance Tools, Callgrind y `perf annotate`.

## 7. Resultados de Brayan

Pendiente de agregar sus archivos, tiempos promedio, overhead y comparación con las herramientas del ejercicio B.

## 8. Comparación grupal

| Integrante | Región con mayor tiempo | Tiempo promedio por llamada | Overhead observado | Coincide con B |
|---|---|---:|---:|---|
| Alejandro | `profile_metrics` | 329.849 ms | No apreciable (-1.39 % observado) | Sí; incluye búsquedas con `GridIndex::nearest`. |
| Angie | `profile_metrics` | 635.408 ms | No apreciable (-0.88 % observado) | Sí; incluye búsquedas con `GridIndex::nearest`. |
| Milagro | Pendiente | Pendiente | Pendiente | Pendiente |
| Brayan | Pendiente | Pendiente | Pendiente | Pendiente |
