<p align="justify">

# Ejercicio C: perfilado mediante revisión de ensamblador 

En este ejercicio revisamos el ensamblador del programa base y usamos `perf annotate` para relacionar las instrucciones con los hotspots encontrados en el ejercicio B.

Cada integrante presenta los resultados de su equipo. Los comandos son los mismos, pero el ensamblador y los porcentajes pueden variar según el compilador, el procesador y la ejecución.

## 1. Comandos utilizados

Desde la carpeta del programa, recompilamos siguiendo el enunciado:

```bash
make clean
make CXXFLAGS="-std=c++17 -O2 -g -Wall -Wextra -pedantic -fno-omit-frame-pointer"
```

Generamos el ensamblador:

```bash
g++ -std=c++17 -O2 -g -S -masm=intel \
    $(pkg-config --cflags gstreamer-1.0 gstreamer-app-1.0) \
    point_cloud_collimation.cpp -o point_cloud_collimation.s
```

Luego registramos las muestras y abrimos la anotación:

```bash
perf record -g ./point_cloud_collimation
perf annotate
```

Para guardar la anotación en texto:

```bash
perf annotate --stdio > point_cloud_collimation.annotate.txt
```

El último comando solamente guarda el análisis de la medición existente en `perf.data`. No vuelve a ejecutar el programa.

Cada integrante guarda estos archivos en su carpeta:

- `point_cloud_collimation.s`
- `point_cloud_collimation.annotate.txt`

El enunciado también permite usar `objdump` como alternativa para revisar el desensamblado.


---

## Especificaciones de los equipos

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



## 2. Resultados de Alejandro

### 2.1. Evidencias

Archivos:

- [Ensamblador](Alejandro/point_cloud_collimation.s)
- [Anotación de perf](Alejandro/point_cloud_collimation.annotate.txt)

El archivo de anotación contiene dos eventos, correspondientes a los tipos de núcleo del procesador. Para `GridIndex::nearest`, aparecen 91 142 muestras en `cpu_core/cycles/P` y 192 en `cpu_atom/cycles/P`. El análisis de porcentajes presentado aquí corresponde a `cpu_core`; no se mezclan ambos eventos.

### 2.2. Revisión de las cinco regiones

#### GridIndex::nearest

Esta función recorre las celdas de una tabla hash y revisa los puntos candidatos. Dentro de cada celda, los índices se leen consecutivamente, pero luego se usan para acceder a las coordenadas en el vector de puntos. Por eso el acceso a los puntos es indirecto.

En el ensamblador aparecen restas, multiplicaciones y sumas para calcular:

```cpp
const double ex = query.x - candidate.x;
const double ey = query.y - candidate.y;
const double d2 = ex * ex + ey * ey;
```

La función utiliza distancias al cuadrado, por lo que no calcula una raíz cuadrada por candidato. También aparecen divisiones enteras en el acceso a la tabla hash.

Hay saltos condicionales para descartar celdas, recorrer candidatos y decidir si se actualiza el vecino más cercano. Esta región combina cálculo repetido, accesos indirectos a memoria y decisiones dentro de los bucles.

#### compare_profiles

Construye dos índices espaciales y compara los perfiles en ambas direcciones mediante dos llamadas a `nearest_neighbor_distances`. Aparecen llamadas a `hypot`, raíces cuadradas mediante `sqrtsd` y llamadas a `percentile`. También hay accesos a la tabla hash durante la construcción de los índices.

Los vectores de puntos y distancias se recorren consecutivamente para calcular centroides y métricas. En cambio, las búsquedas de vecinos realizan accesos indirectos. Hay condiciones para controlar los recorridos, contar puntos dentro del umbral y seleccionar percentiles. Su costo combina trabajo propio con las búsquedas de vecinos que ejecuta.

#### estimate_rigid_transform

Recorre consecutivamente las correspondencias para calcular los centroides y acumular productos entre las coordenadas centradas. En el ensamblador aparecen sumas, restas y multiplicaciones. Al final se llama a `atan2` para calcular el ángulo y a `sincos` para obtener juntos el seno y el coseno. Los bucles incluyen comparaciones y saltos para avanzar hasta la última correspondencia. Por el acceso secuencial y las operaciones matemáticas, esta región parece tener mayor peso de cómputo que de acceso irregular a memoria.

#### add_random_deformation

Recorre los puntos y calcula dos ondas mediante llamadas a `sin`. Después recorre las ocho deformaciones locales y utiliza `exp` para calcular su influencia sobre cada punto. También aparecen multiplicaciones, divisiones y una raíz cuadrada al final para obtener el desplazamiento RMS.

Los puntos y las deformaciones locales se recorren de forma contigua. Hay saltos para controlar los bucles y condiciones para limitar las coordenadas al espacio permitido. Las llamadas matemáticas repetidas hacen que esta región parezca principalmente exigente en cómputo.

#### render_motion_frame

Inicializa el búfer RGB con `memset` y llama a `draw_cloud` para dibujar los perfiles y los cuadros anteriores.

La inicialización de la imagen es contigua. Los puntos también se leen consecutivamente, pero las escrituras de píxeles dependen de sus coordenadas y no necesariamente siguen posiciones consecutivas. Se realizan cálculos de coordenadas y redondeos. También hay condiciones para controlar los límites de los píxeles y los recorridos de cuadros y puntos.

Esta región combina cálculo y escritura en memoria. Como la medición se hizo sin exportación ni visor, se revisó estáticamente en el ensamblador; no se le atribuyen muestras de esa ejecución.

#### Ubicación de las regiones

Estas referencias corresponden al archivo `.s` de Alejandro. Las líneas pueden cambiar al compilar en otro equipo.

| Región | Ubicación en el archivo `.s` |
|---|---|
| `GridIndex::nearest` | Inicio en la línea 4514; bucle de candidatos entre 4944 y 5005. |
| `compare_profiles` | Región principal desde la línea 36509; `sqrtsd` en 38465. |
| `estimate_rigid_transform` | Código incorporado en `main`; acumulaciones entre 48053 y 48213, llamadas a `atan2` y `sincos` en 48224 y 48230. |
| `add_random_deformation` | Código incorporado en `main`; llamadas a `sin` en 50619 y 50641, y a `exp` en 50785. |
| `render_motion_frame` | Inicio en 1510; `memset` en 1775 y llamadas a `draw_cloud` en 1811 y 1820. |

Con `-O2`, algunas funciones quedan incorporadas dentro de otras. También aparecen versiones especializadas identificadas con `.constprop.0`.

Las observaciones sobre cómputo y memoria son hipótesis basadas en el código y los accesos observados. No equivalen a demostrar que una región esté limitada exclusivamente por uno de estos factores.

### 2.3. ¿Qué instrucciones concentran más muestras?

Después de revisar la anotación completa de `GridIndex::nearest` para `cpu_core/cycles/P`, estas son las seis instrucciones con mayores porcentajes locales:

| Dirección | Instrucción | Porcentaje local | Operación |
|---|---|---:|---|
| `0xccbf` | `comisd %xmm0,%xmm1` | 11,88 % | Compara la distancia del candidato con la mejor encontrada. |
| `0xccae` | `mulsd %xmm0,%xmm0` | 11,66 % | Calcula el cuadrado de la diferencia en X. |
| `0xccc3` | `jbe 0xccd7` | 10,84 % | Omite la actualización si el candidato no mejora la distancia. |
| `0xccde` | `jne 0xcc90` | 10,60 % | Continúa con el siguiente candidato. |
| `0xcc9e` | `shlq $0x4,%rax` | 10,42 % | Multiplica el índice por 16 para localizar el punto. |
| `0xcca5` | `subsd (%rax),%xmm0` | 9,75 % | Lee la coordenada X del candidato y calcula la diferencia. |

Las muestras se concentran en el bucle que accede a los candidatos, calcula las distancias y decide si actualiza el vecino más cercano. También aparecen muestras en el recorrido de la tabla hash. Por ejemplo, `movq 0x8(%rax),%r8` registra 3,57 % y `divq -0x48(%rbp)` registra 3,34 %.

El encabezado indica `percent: local period`: los porcentajes están calculados dentro de la función y ponderados por el período de muestreo. No representan porcentajes del tiempo total del programa ni mediciones aisladas de la latencia de cada instrucción. Un porcentaje alto en un salto tampoco demuestra por sí solo que haya muchos fallos de predicción.

La anotación muestra sintaxis AT&T y el `.s` utiliza sintaxis Intel. Además, el comando del enunciado que genera el `.s` no incluye `-fno-omit-frame-pointer`, aunque la compilación del ejecutable sí. Por eso relacionamos las operaciones con el código fuente sin asumir que ambos archivos tengan exactamente los mismos registros y direcciones.

### 2.4. ¿Coinciden con el hotspot del ejercicio B?

Sí. En el perfil normal del ejercicio B, `GridIndex::nearest` registró un 97,12 % de costo propio en el reporte de ciclos de `cpu_core`. También fue la función dominante en Callgrind y Google Performance Tools.

La anotación permite ubicar ese trabajo dentro del bucle de candidatos. Los porcentajes de B comparan funciones dentro del perfil del evento, mientras que los porcentajes locales de esta tabla describen la distribución dentro de `GridIndex::nearest`.

### 2.5. ¿Qué cambio intentaríamos primero?

Primero probaríamos cambiar la organización de los puntos en la estructura de vecinos, guardando juntas las coordenadas de los puntos de cada celda. Actualmente se lee un índice y después se accede al punto en otra posición del vector. Agrupar los puntos por celda permitiría recorrer sus coordenadas consecutivamente y podría mejorar la localidad de memoria. La propuesta se enfoca en la región donde coinciden las tres herramientas y donde aparecen las instrucciones con más muestras. La mejora todavía debe comprobarse: habría que medir tanto la construcción de la estructura como las búsquedas y verificar que se conserve el resultado de la alineación.


## 3. Resultados de Angie

### 3.1. Evidencias

Archivos:

- [Ensamblador](Angie/point_cloud_collimation.s)
- [Anotación de perf](Angie/point_cloud_collimation.annotate.txt)

La ejecución analizada con `perf annotate` contiene 196 863 muestras para el evento `cycles:P`. Los porcentajes mostrados por la herramienta corresponden a `percent: local period`, por lo que indican cómo se distribuyen las muestras dentro de la región anotada y no el porcentaje total de ejecución del programa.

### 3.2. Revisión de las cinco regiones

#### GridIndex::nearest

Esta función realiza la búsqueda del punto más cercano. Primero determina las celdas que deben revisarse mediante la tabla hash y después recorre los candidatos almacenados en ellas.

El acceso a los candidatos es indirecto: se obtiene un índice desde la celda y este índice se utiliza posteriormente para acceder al vector de puntos. Esto puede observarse en instrucciones como `movslq` para obtener el índice y `shl` para calcular la posición del punto.

El cálculo de la distancia utiliza operaciones escalares de punto flotante. En particular, aparecen instrucciones `subsd`, `mulsd` y `addsd` correspondientes al cálculo de:

```cpp
const double ex = query.x - candidate.x;
const double ey = query.y - candidate.y;
const double d2 = ex * ex + ey * ey;
```

Después, `comisd` compara la distancia obtenida con la mejor distancia encontrada. También aparece una división entera `div` durante el acceso a la tabla hash y saltos condicionales para controlar la búsqueda y el recorrido de candidatos.

Por lo tanto, esta región presenta una combinación de cálculo de distancias, accesos indirectos a memoria, operaciones de la tabla hash y control de flujo dentro de los bucles.

#### compare_profiles

Esta región compara los dos perfiles mediante búsquedas de vecinos en ambas direcciones. Para ello construye índices espaciales y utiliza `nearest_neighbor_distances`, lo que hace que parte de su trabajo dependa directamente de `GridIndex::nearest`.

En el ensamblador se identificaron llamadas a `hypot` e instrucciones `sqrtsd`, asociadas al cálculo de distancias y métricas. Los vectores utilizados para almacenar puntos y resultados presentan recorridos secuenciales en varias partes de la función, mientras que las búsquedas de vecinos introducen accesos indirectos.

Por esta razón, `compare_profiles` combina operaciones matemáticas, recorridos de vectores y el costo de las búsquedas espaciales.

#### estimate_rigid_transform

Con la optimización `-O2`, esta función no permaneció como una región independiente en el ensamblador, ya que parte de su código fue incorporado en `main`.

La región procesa las correspondencias entre puntos para obtener los parámetros de la transformación. El cálculo utiliza sumas, restas y multiplicaciones sobre las coordenadas, además de operaciones trigonométricas. En el ensamblador optimizado se localizaron llamadas relacionadas con `cos` y `sin`.

Los datos de las correspondencias se recorren principalmente de forma secuencial. Debido a este patrón de acceso y a las operaciones matemáticas realizadas, esta región presenta un comportamiento principalmente orientado al cálculo, aunque esta observación no constituye por sí sola una medición de que esté limitada por cómputo.

#### add_random_deformation

Con `-O2`, esta función también fue incorporada en otras regiones del programa y no aparece como un símbolo independiente.

En el ensamblador se localizaron llamadas a `sin` y `exp` asociadas con los cálculos de las deformaciones. También se observaron operaciones de raíz cuadrada en el código relacionado con los cálculos matemáticos del programa.

Los puntos y parámetros de deformación se procesan mediante bucles, con operaciones matemáticas repetidas para modificar las coordenadas. Los accesos son más regulares que los observados en la búsqueda mediante tabla hash de `GridIndex::nearest`.

Por ello, esta región presenta principalmente trabajo aritmético y llamadas a funciones matemáticas dentro de los recorridos.

#### render_motion_frame

Esta región prepara la representación gráfica del movimiento. En el ensamblador aparece como una versión optimizada identificada mediante `.constprop.0`.

La función trabaja sobre el búfer de la imagen y utiliza las coordenadas de los puntos para determinar las posiciones donde deben dibujarse. La inicialización del búfer presenta un patrón regular, mientras que las escrituras de los píxeles dependen de las coordenadas calculadas y no necesariamente ocurren de manera consecutiva.

También se realizan operaciones para transformar y redondear coordenadas antes de dibujar. Por tanto, esta región combina procesamiento de coordenadas con accesos de escritura al búfer de imagen.

#### Ubicación de las regiones

Las siguientes ubicaciones corresponden al ensamblador generado en el equipo de Angie:

| Región | Ubicación observada |
|---|---|
| `GridIndex::nearest` | Inicio en la dirección `0xb970`; el bucle de candidatos comienza alrededor de `0xbb40`. |
| `compare_profiles` | Región principal desde la línea 19220 del archivo `.s`, identificada como `.constprop.0`. |
| `estimate_rigid_transform` | Con `-O2` su código aparece incorporado en `main`; no permanece como función independiente. |
| `add_random_deformation` | Con `-O2` su código aparece incorporado en otras regiones; no permanece como función independiente. |
| `render_motion_frame` | Región principal desde la línea 1397 del archivo `.s`, identificada como `.constprop.0`. |

Para facilitar la identificación de `estimate_rigid_transform` y `add_random_deformation` se utilizó adicionalmente una compilación auxiliar sin optimización. El análisis de rendimiento, sin embargo, se realizó sobre la compilación requerida con `-O2`.

Las clasificaciones relacionadas con cómputo y memoria se utilizan únicamente para describir los patrones observados en el ensamblador; no demuestran por sí solas que una función esté limitada exclusivamente por alguno de estos factores.

### 3.3. ¿Qué instrucciones concentran más muestras?

En `GridIndex::nearest`, las instrucciones con mayores porcentajes locales observados mediante `perf annotate` fueron:

| Dirección | Instrucción | Porcentaje local | Operación |
|---|---|---:|---|
| `0xbb55` | `subsd (%rax),%xmm0` | 16,00 % | Calcula la diferencia entre una coordenada de la consulta y del candidato. |
| `0xbb66` | `addsd %xmm1,%xmm0` | 9,43 % | Suma los dos términos utilizados para obtener la distancia al cuadrado. |
| `0xbb70` | `comisd %xmm0,%xmm1` | 8,60 % | Compara la distancia calculada con la mejor encontrada. |
| `0xbb5e` | `mulsd %xmm0,%xmm0` | 7,54 % | Calcula el cuadrado de una de las diferencias. |
| `0xbacf` | `div %r9` | 6,75 % | Realiza una división utilizada durante el acceso a la tabla hash. |
| `0xbb4e` | `shl $0x4,%rax` | 5,71 % | Calcula el desplazamiento para acceder al punto candidato. |
| `0xbb91` | `jne bb40` | 5,69 % | Controla la continuación del recorrido de candidatos. |
| `0xbb40` | `movslq (%rdx),%rax` | 3,64 % | Obtiene el índice del candidato. |
| `0xbb62` | `mulsd %xmm1,%xmm1` | 3,31 % | Calcula el cuadrado de la segunda diferencia. |
| `0xbb59` | `subsd 0x8(%rax),%xmm1` | 2,83 % | Calcula la diferencia de la segunda coordenada. |

La mayor concentración individual aparece en `subsd (%rax),%xmm0`, con 16,00 %. Sin embargo, las muestras no se concentran en una única operación: están distribuidas entre el acceso al candidato, el cálculo de la distancia, la comparación del resultado, el recorrido del bucle y el manejo de la tabla hash.

Esto permite observar que el costo de `GridIndex::nearest` proviene del conjunto de operaciones ejecutadas repetidamente durante cada búsqueda, y no solamente de una instrucción aislada.

Los porcentajes son locales a la región mostrada por `perf annotate`. No deben interpretarse como porcentajes del tiempo total del programa ni como la latencia individual de cada instrucción.

### 3.4. ¿Coinciden con el hotspot del ejercicio B?

Sí. En el ejercicio B, `perf report` mostró a `GridIndex::nearest` como la función dominante, con 97,76 % en `Self` y 98,15 % en `Children`. Callgrind también concentró aproximadamente 83,45 % de las instrucciones registradas directamente en esta función.

El análisis de ensamblador permite precisar dónde se concentra ese trabajo. Las instrucciones con mayor cantidad de muestras pertenecen al recorrido de candidatos y al cálculo de sus distancias, junto con operaciones asociadas al acceso a la tabla hash.

Por lo tanto, el resultado de `perf annotate` es consistente con el hotspot encontrado previamente: `GridIndex::nearest` continúa siendo la región principal sobre la cual tendría sentido investigar optimizaciones.

### 3.5. ¿Qué cambio intentaríamos primero?

El primer cambio que probaríamos sería reducir el trabajo realizado durante la búsqueda de vecinos. Los resultados muestran que una parte importante de las muestras aparece justamente mientras se obtiene cada candidato, se accede a sus coordenadas y se calcula su distancia.

Una posibilidad sería reorganizar la estructura utilizada para almacenar los puntos de las celdas, buscando que el acceso a los candidatos sea más regular y que se evalúen únicamente los puntos necesarios. Esto atacaría directamente la región identificada como hotspot sin modificar inicialmente otras partes del programa que tienen una contribución menor.

Después del cambio sería necesario repetir las mediciones y comparar tanto el tiempo de ejecución como el resultado de la alineación para determinar si realmente existe una mejora.


#### GridIndex::nearest

Esta función realiza la búsqueda del vecino más cercano utilizando una tabla
hash para localizar las celdas y posteriormente recorrer los puntos candidatos.
En el ensamblador aparecen instrucciones `subsd`, `mulsd` y `addsd` para
calcular la distancia al cuadrado entre el punto consultado y cada candidato.
También aparece `comisd` para comparar la distancia calculada con la mejor
distancia encontrada.
El acceso a los puntos es indirecto. Primero se obtiene el índice del candidato
con una instrucción como `movsxd`, luego se calcula su desplazamiento mediante
`shl` y finalmente se accede a sus coordenadas. En la búsqueda sobre la tabla
hash también aparecen accesos mediante buckets y punteros, además de
instrucciones `div` asociadas al cálculo del bucket.
Se observan saltos como `jbe`, `je` y `jne` dentro de los recorridos y las
comparaciones. Por estas características, esta región combina cálculo repetido
de distancias con accesos irregulares a memoria.

#### compare_profiles

Esta función construye índices espaciales y utiliza
`nearest_neighbor_distances` para comparar los perfiles. Estas búsquedas
terminan utilizando `GridIndex::nearest`, por lo que parte de su ejecución
incluye los accesos indirectos asociados a la búsqueda de vecinos.
En el ensamblador también aparecen operaciones como `hypot` y `sqrtsd`
relacionadas con el cálculo de métricas y distancias.
Los vectores utilizados por la función presentan recorridos secuenciales, pero
las búsquedas de vecinos introducen accesos indirectos. Por esta razón, esta
región presenta un comportamiento mixto entre cálculo y acceso a memoria.

#### estimate_rigid_transform

Esta región recorre las correspondencias almacenadas en un `vector<Match>` para
calcular centroides y acumular operaciones sobre las coordenadas. El recorrido
es principalmente secuencial y en el ensamblador aparecen sumas, restas y
multiplicaciones.
También se identificaron llamadas a `atan2`, `cos` y `sin` para calcular la
transformación. Los bucles contienen comparaciones y saltos para avanzar entre
las correspondencias. Debido al acceso secuencial a los datos y a las operaciones matemáticas, esta región parece tener mayor peso de cómputo que de acceso irregular a memoria.

#### add_random_deformation

Esta región recorre los puntos y aplica las deformaciones utilizando operaciones
matemáticas como `sin`, `exp`, `hypot` y `sqrt`. También aparecen
multiplicaciones y otras operaciones aritméticas dentro de los bucles.
Los puntos y las deformaciones se recorren principalmente de forma secuencial.
Se observan saltos asociados al control de los bucles y a las condiciones
utilizadas durante la modificación de los puntos.
La presencia repetida de operaciones matemáticas como `sin` y `exp`, junto con
un patrón de acceso relativamente regular, hace que esta región parezca estar
más asociada al cómputo.

#### render_motion_frame

Esta función prepara el frame utilizado para representar el movimiento. En el
ensamblador aparecen operaciones de inicialización del buffer y llamadas a
`draw_cloud`, `draw_rect` y `lround`.
La inicialización del buffer presenta un acceso regular. Los puntos pueden
recorrerse secuencialmente, aunque las posiciones donde se escriben los píxeles
dependen de las coordenadas proyectadas y, por tanto, las escrituras no
necesariamente son consecutivas.
Esta región combina cálculos de coordenadas y escritura en memoria. No presenta
la misma concentración de operaciones matemáticas costosas observada en
`add_random_deformation` ni los accesos mediante tabla hash de
`GridIndex::nearest`.

#### Ubicación de las regiones

Estas referencias corresponden al archivo `point_cloud_collimation.objdump`
generado en el equipo de Angie.

| Región | Ubicación observada |
|---|---|
| `GridIndex::nearest` | Símbolo principal en `0xb970`; búsqueda hash aproximadamente entre `0xbab7` y `0xbb25`; bucle de candidatos desde `0xbb40`. |
| `compare_profiles` | Región principal identificada como `compare_profiles(...) [clone .constprop.0]`. |
| `estimate_rigid_transform` | Con `-O2` no permanece como símbolo independiente. Mediante `addr2line` se localizó código incorporado en `main`. |
| `add_random_deformation` | Con `-O2` no permanece como símbolo independiente. Mediante `addr2line` se localizaron operaciones incorporadas en `main`, incluyendo `sin` y `exp`. |
| `render_motion_frame` | Región identificada como `render_motion_frame(...) [clone .constprop.0]`. |

Para facilitar la identificación de `estimate_rigid_transform` y
`add_random_deformation` se realizó adicionalmente una compilación auxiliar con
`-O0`. Posteriormente se restauró la compilación requerida con `-O2`. Con
`addr2line` se comprobó que código correspondiente a estas regiones había sido
incorporado dentro de `main` por las optimizaciones del compilador.

Modificar la estructura utilizada para la búsqueda de
vecinos. `GridIndex::nearest` utiliza una tabla hash para localizar
las celdas y posteriormente obtiene índices que se utilizan para acceder a los
puntos candidatos.

El ensamblador y `perf annotate` muestran muestras tanto en las operaciones
asociadas con la tabla hash como en el acceso a los candidatos y el cálculo
repetido de sus distancias. Una organización que permita recorrer los puntos de
cada celda de forma más regular, o que reduzca la cantidad de candidatos
evaluados, podría disminuir estos costos.



## 4. Resultados de Milagro

### 4.1. Equipo y evidencias

**Pendiente:** indicar procesador, sistema operativo, compilador, versión de perf, evento analizado y enlaces a los archivos de la carpeta `Milagro`.

### 4.2. Revisión de las cinco regiones

Completar según el ensamblador generado en este equipo:

| Región | Instrucciones o llamadas costosas | Acceso contiguo o indirecto | Saltos condicionales | Posible limitación por cómputo o memoria |
|---|---|---|---|---|
| `GridIndex::nearest` | Pendiente | Pendiente | Pendiente | Pendiente |
| `compare_profiles` | Pendiente | Pendiente | Pendiente | Pendiente |
| `estimate_rigid_transform` | Pendiente | Pendiente | Pendiente | Pendiente |
| `add_random_deformation` | Pendiente | Pendiente | Pendiente | Pendiente |
| `render_motion_frame` | Pendiente | Pendiente | Pendiente | Pendiente |

Agregar referencias a las regiones revisadas mediante líneas, símbolos o fragmentos del ensamblador.

### 4.3. ¿Qué instrucciones concentran más muestras?

**Pendiente:** incluir función, evento, instrucciones, porcentajes y explicación. Indicar si los porcentajes son locales o globales.

### 4.4. ¿Coinciden con el hotspot del ejercicio B?

**Pendiente:** comparar con los resultados de Milagro del ejercicio B.

### 4.5. ¿Qué cambio intentaríamos primero?

**Pendiente:** proponer un cambio y justificarlo con las mediciones y el ensamblador.

## 5. Resultados de Brayan

### 5.1. Equipo y evidencias

**Pendiente:** indicar procesador, sistema operativo, compilador, versión de perf, evento analizado y enlaces a los archivos de la carpeta `Brayan`.

### 5.2. Revisión de las cinco regiones

Completar según el ensamblador generado en este equipo:

| Región | Instrucciones o llamadas costosas | Acceso contiguo o indirecto | Saltos condicionales | Posible limitación por cómputo o memoria |
|---|---|---|---|---|
| `GridIndex::nearest` | Pendiente | Pendiente | Pendiente | Pendiente |
| `compare_profiles` | Pendiente | Pendiente | Pendiente | Pendiente |
| `estimate_rigid_transform` | Pendiente | Pendiente | Pendiente | Pendiente |
| `add_random_deformation` | Pendiente | Pendiente | Pendiente | Pendiente |
| `render_motion_frame` | Pendiente | Pendiente | Pendiente | Pendiente |

Agregar referencias a las regiones revisadas mediante líneas, símbolos o fragmentos del ensamblador.

### 5.3. ¿Qué instrucciones concentran más muestras?

**Pendiente:** incluir función, evento, instrucciones, porcentajes y explicación. Indicar si los porcentajes son locales o globales.

### 5.4. ¿Coinciden con el hotspot del ejercicio B?

**Pendiente:** comparar con los resultados de Brayan del ejercicio B.

### 5.5. ¿Qué cambio intentaríamos primero?

**Pendiente:** proponer un cambio y justificarlo con las mediciones y el ensamblador.

## 6. Comparación de resultados

| Integrante | Función analizada con mayor detalle | Instrucción con mayor porcentaje local en esa función | Porcentaje | Coincidencia con B | Primera propuesta |
|---|---|---|---:|---|---|
| Alejandro | `GridIndex::nearest` | `comisd %xmm0,%xmm1` | 11,88 % | Sí | Agrupar las coordenadas de los puntos por celda. |
| Angie | `GridIndex::nearest`  | `subsd (%rax),%xmm0` | 16,00 % | Sí | Modificar la estructura utilizada para la búsqueda de
vecinos |
| Milagro | Pendiente | Pendiente | Pendiente | Pendiente | Pendiente |
| Brayan | Pendiente | Pendiente | Pendiente | Pendiente | Pendiente |

Los porcentajes locales sirven para identificar dónde se concentran las muestras dentro de cada función. Por sí solos no permiten decidir qué equipo ejecutó el programa más rápido.

La conclusión grupal queda pendiente hasta incorporar los resultados de todos.

## 7. Referencia

- [Manual de perf annotate: interpretación de porcentajes y opciones](https://man7.org/linux/man-pages/man1/perf-annotate.1.html).
</p>
