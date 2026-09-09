# Ejercicio A: comprensión del programa base

El programa genera dos nubes de 100 000 puntos con forma de rieles y
travesaños. Una se usa como referencia y la otra se rota, se desplaza
y recibe deformación y ruido. Después se aplica ICP para intentar
alinearlas.

## Funciones principales

| Función o estructura | Qué hace |
|---|---|
| `generate_h_rail_cloud` | Genera los puntos de los rieles y travesaños. |
| `add_random_deformation` | Deforma el perfil fuente. |
| `GridIndex` | Organiza los puntos en celdas para facilitar la búsqueda de vecinos. |
| `GridIndex::nearest` | Busca un vecino cercano al punto consultado. |
| `estimate_rigid_transform` | Calcula la rotación y traslación a partir de las correspondencias. |
| `compare_profiles` | Calcula los centroides y las métricas de comparación. |
| `render_motion_frame` | Dibuja los cuadros del movimiento de los perfiles. |

## Comandos utilizados

Todos los comandos se ejecutaron desde la carpeta principal del proyecto.

Para compilar se usaron las opciones que aparecen en el inciso C, dejando el programa preparado para el análisis posterior:

```bash
make CXXFLAGS="-std=c++17 -O2 -g -Wall -Wextra -pedantic -fno-omit-frame-pointer"
```

La primera ejecución se guardó en un archivo para conservar la salida:

```bash
mkdir -p resultados/original
./point_cloud_collimation > resultados/original/ejecucion_normal.txt 2>&1
tail -n 20 resultados/original/ejecucion_normal.txt
```

Para exportar los resultados y abrir el visor se usaron los comandos del enunciado:

```bash
./point_cloud_collimation --export
./point_cloud_collimation --viewer
```

También se usaron estos comandos para revisar los archivos generados:

```bash
ls reconstruction

head -n 6 reconstruction/target_profile.csv
head -n 6 reconstruction/source_initial_profile.csv
head -n 6 reconstruction/source_final_profile.csv

wc -l reconstruction/target_profile.csv reconstruction/source_initial_profile.csv reconstruction/source_final_profile.csv

cat reconstruction/source_motion.csv
cat reconstruction/profile_metrics.csv

xdg-open reconstruction/frame_000.ppm
xdg-open reconstruction/frame_014.ppm
xdg-open reconstruction/frame_028.ppm
```

## Revisión de los archivos

Los tres archivos de perfiles tienen las columnas `label`, `index`, `x` y `y`. Cada uno contiene 100 000 puntos más el encabezado. La fuente final conserva la cantidad de puntos, pero sus coordenadas cambian por la transformación aplicada.

En `source_motion.csv` se guardan la rotación y la traslación de los cuadros exportados. Hay 29 cuadros, del 0 al 28, aunque el algoritmo realizó 45 iteraciones. Por eso, el número de cuadro no debe confundirse con el número de iteración. El último cuadro sí coincide con la transformación final.

En `profile_metrics.csv` aparecen el estado inicial y las 45 iteraciones. El `profile_score` pasó de 0.04672448 a 0.01847086, el Chamfer RMSE bajó de 470.75 a 111.00 y la cobertura aumentó de 80.112 % a 96.936 %.

Se abrieron los cuadros 000, 014 y 028 para comparar el inicio, una etapa intermedia y el final. Al principio la fuente se ve inclinada y desplazada. Al final los rieles quedan casi superpuestos, pero todavía se observa una diferencia vertical en los extremos.

Se incluyen `source_motion.csv` y `profile_metrics.csv`, como pide la sección de entregables.

## Preguntas

### ¿Qué representan los centroides de ambos perfiles?

Son el centro promedio de los puntos de cada perfil. Se obtienen promediando las coordenadas en X y Y. El centroide del objetivo permanece fijo, mientras que el de la fuente cambia cuando se rota y se desplaza. La distancia entre ellos permite comparar la posición general de las nubes. Sin embargo, dos perfiles pueden tener el mismo centroide y estar desalineados por una rotación.

### ¿Cómo cambia la distancia entre centroides durante las iteraciones?

La distancia comenzó en 760.12 unidades y bajó hasta 541.59 en la iteración 21. Después aumentó y terminó en 600.87 unidades. Es decir,
terminó por debajo del valor inicial, pero no bajó en todas las iteraciones. Esto tiene sentido porque ICP calcula la transformación a partir de parejas de puntos cercanos, no solamente de los centroides. En los resultados se observa que el error simétrico siguió bajando incluso cuando la distancia entre centroides aumentó.

### ¿Qué diferencia hay entre match_rmse, symmetric_chamfer_rmse y profile_score?

-`match_rmse` mide el error de las parejas de puntos que el algoritmo aceptó como correspondencias. Calcula la raíz del promedio de sus distancias al cuadrado. En este código usa las distancias anteriores a aplicar la actualización de la iteración.

-`symmetric_chamfer_rmse` considera las distancias en ambos sentidos: de la fuente al objetivo y del objetivo a la fuente. Se calcula después de actualizar la nube e incluye una penalización cuando no se encuentra vecino. Por eso puede dar un valor diferente al de las correspondencias aceptadas.

-`profile_score` reúne el error simétrico y la separación entre centroides en un solo valor. Se calcula sumando el Chamfer RMSE con 0.25 veces la distancia entre centroides y dividiendo entre la diagonal del espacio de trabajo. Cuanto menor sea, mejor es el resultado según ese criterio.

### ¿Por qué una deformación no rígida evita que la transformación recuperada sea exactamente igual a la transformación ideal?

Porque no desplaza todos sus puntos de la misma manera. Una rotación y una traslación solo mueven el conjunto, sin corregir esas diferencias internas. Entonces, aunque se deshiciera el movimiento original, seguirían presentes la deformación y el ruido. El algoritmo busca el ajuste que mejor funciona con las correspondencias encontradas, que puede ser diferente de la transformación ideal.

En esta prueba se recuperó una rotación de −18.00595°, muy cercana a los −18° esperados, pero quedó una diferencia mayor en la traslación vertical. No se puede asegurar que toda esa diferencia sea por la deformación, ya que los travesaños repetidos también podrían afectar las correspondencias.