# Reporte de Laboratorio: OpenMP Image Pipeline

**Máquina:** Apple M1 Pro (8 cores)  
**Compilador:** gcc-16 (Homebrew GCC 16.2.0) `-std=c11 -Wall -Wextra -O2 -fopenmp`  
**Imagen:** `data/input.ppm` (4000 x 3000, 12 Mpx)

---

## Verificación de Correctitud

```sh
$ ./bin/pipeline_serial data/input.ppm
$ ./bin/pipeline_omp data/input.ppm 4
$ cmp data/edges_serial.ppm data/edges_omp.ppm
```

---

## Tablas de Medición

### Tabla 1: Escalamiento del pipeline por número de hilos

| threads | gray | blur | sobel | hist (reduction) | total | speedup |
|---|---|---|---|---|---|---|
| 1 | 0.013 s | 0.143 s | 0.035 s | 0.008 s | 0.216 s | 1.00x |
| 2 | 0.007 s | 0.073 s | 0.018 s | 0.004 s | 0.118 s | 1.83x |
| 4 | 0.004 s | 0.037 s | 0.010 s | 0.002 s | 0.069 s | 3.13x |
| 8 | 0.004 s | 0.028 s | 0.009 s | 0.002 s | 0.060 s | 3.60x |

### Tabla 2: Etapa 4 (histograma)

| threads | naive | critical | atomic | reduction |
|---|---|---|---|---|
| 1 | 0.008 s | 0.103 s | 0.012 s | 0.008 s |
| 4 | 0.029 s | 0.384 s | 0.305 s | 0.002 s |

---

## Questions to answer

### 1. Run the naive histogram with one thread, then with four.
With one thread it is correct. With four it loses counts, and loses a *different* number on every run. Explain both observations.

Con 1 hilo no hay concurrencia; cada incremento `hist[edges[i]]++` se ejecuta de forma estrictamente secuencial y ordenada en memoria. Con 4 hilos se produce una condición de carrera sobre las mismas posiciones de memoria compartidas: la operación de incremento consta de tres pasos a nivel de procesador (lectura, suma, escritura), por lo que cuando dos o más hilos leen el mismo valor antiguo antes de que el otro escriba, se pierden conteos. El número de cuentas perdidas varía en cada corrida debido al no-determinismo del planificador del sistema operativo al alternar y pausar los hilos en los núcleos.

### 2. Where is the barrier?
The blur reads a neighbourhood of `gray`, so stage 1 has to be completely finished before any thread starts stage 2. You should not have needed to write `#pragma omp barrier` anywhere. What guarantees the ordering?

Al final de cada directiva de compartición de trabajo como `#pragma omp parallel for` (así como al cierre de cada bloque `#pragma omp parallel`) existe una barrera de sincronización implícita. Ningún hilo puede abandonar la función de la etapa actual hasta que todos los hilos hayan concluido sus iteraciones asignadas, garantizando que `grayscale` termine por completo antes de que inicie `blur`, y este antes de `sobel`, sin requerir un `#pragma omp barrier` explícito.

### 3. `critical` against `atomic`.
Both give the right answer. Measure both and explain the gap. What does `critical` do that `atomic` does not?

Ambas variantes producen 0 bins erróneos, pero `atomic` (0.305 s con 4 hilos) es más rápida que `critical` (0.384 s). `#pragma omp critical` establece una sección crítica general basada en un cerrojo (mutex) por software administrado por el runtime de OpenMP, el cual bloquea globalmente a cualquier hilo que intente entrar a la sección sin importar qué posición del arreglo esté modificando. `#pragma omp atomic`, en cambio, delega la atomicidad directamente a una instrucción de hardware del procesador (como operaciones atómicas en memoria o bus-locking a nivel de caché) afectando exclusivamente la dirección de memoria específica del bin a actualizar y eliminando la sobrecarga del cerrojo de software.

### 4. `critical` as you add threads.
Time it with 1, 2 and 4 threads. It does not get faster. Explain why, and say what that implies about putting a `critical` inside a hot loop.

`critical` pasa de 0.103 s con 1 hilo a 0.384 s con 4 hilos. No acelera porque fuerza una ejecución estrictamente serializada del bucle: un solo hilo puede incrementar a la vez. Agregar más hilos únicamente incrementa la contención por el único cerrojo global y la sobrecarga de context switching y sincronización del sistema operativo. Esto evidencia que ubicar un `#pragma omp critical` dentro de un bucle de alta frecuencia de ejecución (*hot loop*) elimina cualquier ganancia del paralelismo y degrada severamente el rendimiento.

### 5. Amdahl.
The program prints the measured `P` and the ceiling. Using the numbers from *your* run: what is the most threads worth buying for this pipeline, and which change would raise the ceiling more — more cores, or making the save faster?

A partir de la medición base:
- $T_{\text{ser}} = 0.017\text{ s}$ (I/O: carga y guardado)
- $T_{\text{par}} = 0.182\text{ s}$
- $P = 0.182 / 0.199 \approx 0.915$
- Fracción estrictamente serial: $1 - P \approx 0.085$
- Techo teórico de Amdahl: $\frac{1}{1 - P} = \frac{1}{0.085} \approx 11.7\times$

Al llegar a 4 u 8 hilos, el tiempo paralelizable se reduce a ~0.039 s, punto en el que el I/O serial (0.017 s) representa cerca del 30% del tiempo total. Más de 8 hilos ofrece ganancias marginales muy bajas. Para elevar significativamente el techo de aceleración, conviene optimizar el guardado/carga (E/S), ya que reduce directamente la fracción serial $(1 - P)$.

### 6. Granularity.
Regenerate the image at 400 x 300 with `./bin/gen_image data/small.ppm 400 300` and run both versions on it. Does OpenMP still win? Relate your answer to where the crossover point is.

Prácticamente no. En una imagen de 400 x 300 (120,000 píxeles), el tiempo serial es de ~0.003 s y el tiempo con OpenMP (4 hilos) es de ~0.002 s (ganancia de apenas 1 ms, dentro del margen de ruido de medición). El problema cae por debajo del punto de corte (*crossover point*): la carga de trabajo por hilo es tan pequeña que la sobrecarga del runtime de OpenMP (despertar de hilos, particionado de bucles y barreras de sincronización) iguala o supera el tiempo útil de cómputo. OpenMP solo ofrece una ventaja real cuando la carga computacional supera ese costo fijo de coordinación.
