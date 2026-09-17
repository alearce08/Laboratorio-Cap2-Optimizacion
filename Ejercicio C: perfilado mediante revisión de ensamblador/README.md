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

En este inciso no se modificó el programa.

## 3. Resultados de Angie

### 3.1. Equipo y evidencias

**Pendiente:** indicar procesador, sistema operativo, compilador, versión de perf, evento analizado y enlaces a los archivos de la carpeta `Angie`.

### 3.2. Revisión de las cinco regiones

Completar según el ensamblador generado en este equipo:

| Región | Instrucciones o llamadas costosas | Acceso contiguo o indirecto | Saltos condicionales | Posible limitación por cómputo o memoria |
|---|---|---|---|---|
| `GridIndex::nearest` | Pendiente | Pendiente | Pendiente | Pendiente |
| `compare_profiles` | Pendiente | Pendiente | Pendiente | Pendiente |
| `estimate_rigid_transform` | Pendiente | Pendiente | Pendiente | Pendiente |
| `add_random_deformation` | Pendiente | Pendiente | Pendiente | Pendiente |
| `render_motion_frame` | Pendiente | Pendiente | Pendiente | Pendiente |

Agregar referencias a las regiones revisadas mediante líneas, símbolos o fragmentos del ensamblador.

### 3.3. ¿Qué instrucciones concentran más muestras?

**Pendiente:** incluir función, evento, instrucciones, porcentajes y explicación. Indicar si los porcentajes son locales o globales.

### 3.4. ¿Coinciden con el hotspot del ejercicio B?

**Pendiente:** comparar con los resultados de Angie del ejercicio B.

### 3.5. ¿Qué cambio intentaríamos primero?

**Pendiente:** proponer un cambio y justificarlo con las mediciones y el ensamblador.

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
| Angie | Pendiente | Pendiente | Pendiente | Pendiente | Pendiente |
| Milagro | Pendiente | Pendiente | Pendiente | Pendiente | Pendiente |
| Brayan | Pendiente | Pendiente | Pendiente | Pendiente | Pendiente |

Los porcentajes locales sirven para identificar dónde se concentran las muestras dentro de cada función. Por sí solos no permiten decidir qué equipo ejecutó el programa más rápido.

La conclusión grupal queda pendiente hasta incorporar los resultados de todos.

## 7. Referencia

- [Manual de perf annotate: interpretación de porcentajes y opciones](https://man7.org/linux/man-pages/man1/perf-annotate.1.html).