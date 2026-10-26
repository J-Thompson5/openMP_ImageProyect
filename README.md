# Lab — Parallelising an image pipeline with OpenMP

**CI-0117 · Programación Paralela y Concurrente · ECCI, Universidad de Costa Rica**

## The scenario

A research group keeps a camera trained on a test rig and ends up with
thousands of 12-megapixel frames. For each frame they need an **edge map** and
a **histogram of edge intensities** — the histogram is what tells them whether
anything moved between two frames.

The pipeline they have works, and it is too slow. One frame takes about half a
second on one core, and they have thousands of frames. Your job is to
parallelise it with OpenMP without changing what it produces.

```
      load          stage 1        stage 2         stage 3        stage 4         save
    ┌────────┐   ┌───────────┐   ┌──────────┐   ┌──────────┐   ┌───────────┐   ┌────────┐
    │  PPM   │──▶│ grayscale │──▶│  5x5     │──▶│  Sobel   │──▶│ histogram │──▶│  PPM   │
    │  read  │   │           │   │  blur    │   │          │   │ 256 bins  │   │  write │
    └────────┘   └───────────┘   └──────────┘   └──────────┘   └───────────┘   └────────┘
      serial      per-pixel       neighbourhood  neighbourhood   shared bins      serial
```

Stages 1 to 4 are yours to parallelise. Load and save are strictly serial, and
that is what sets your Amdahl ceiling.

## Build and run

```sh
make            # builds everything into bin/
make image      # generates data/input.ppm, 4000 x 3000, deterministic
make run        # serial baseline, then your version
```

The image generator uses a fixed seed, so every machine produces the same
image and the same histogram. Two students can compare numbers directly.

To choose the thread count:

```sh
./bin/pipeline_omp data/input.ppm 4      # as an argument
OMP_NUM_THREADS=4 ./bin/pipeline_omp data/input.ppm
```

## What you have to do

Only `src/main_omp.c` is yours. Do not modify `filters.c`, `ppm.c` or
`main_serial.c` — they are the reference, and the grader uses them.

As handed to you the file compiles and runs, but every stage still calls the
serial version, so it is exactly as slow as the baseline. There are four TODOs.

| Task | Stage | What to use |
|---|---|---|
| 1 | grayscale | one `parallel for` |
| 2 | blur | `parallel for`, consider `collapse(2)` |
| 3 | Sobel | same shape as the blur |
| 4 | histogram | four versions: naive, `critical`, `atomic`, `reduction` |

Task 4 is the heart of the lab. Implement **all four** and run them, because
the comparison is the point — not any single version.

## Correctness comes first

A fast wrong answer is worth nothing. Your version must produce an edge map
identical to the serial one, byte for byte:

```sh
./bin/pipeline_serial data/input.ppm
./bin/pipeline_omp    data/input.ppm 4
cmp data/edges_serial.ppm data/edges_omp.ppm   # must print nothing
```

The program also checks the histogram for you, against the serial reference,
and prints how many of the 256 bins came out wrong.

## What to measure

Fill in this table in your report, one row per thread count, on **your** machine:

| threads | gray | blur | sobel | hist (reduction) | total | speedup |
|---|---|---|---|---|---|---|
| 1 | | | | | | 1.00x |
| 2 | | | | | | |
| 4 | | | | | | |
| 8 | | | | | | |

And this one, for stage 4 only:

| threads | naive | critical | atomic | reduction |
|---|---|---|---|---|
| 1 | | | | |
| 4 | | | | |

## Questions to answer

1. **Run the naive histogram with one thread, then with four.** With one
   thread it is correct. With four it loses counts, and loses a *different*
   number on every run. Explain both observations.

2. **Where is the barrier?** The blur reads a neighbourhood of `gray`, so
   stage 1 has to be completely finished before any thread starts stage 2.
   You should not have needed to write `#pragma omp barrier` anywhere.
   What guarantees the ordering?

3. **`critical` against `atomic`.** Both give the right answer. Measure both
   and explain the gap. What does `critical` do that `atomic` does not?

4. **`critical` as you add threads.** Time it with 1, 2 and 4 threads. It does
   not get faster. Explain why, and say what that implies about putting a
   `critical` inside a hot loop.

5. **Amdahl.** The program prints the measured `P` and the ceiling. Using the
   numbers from *your* run: what is the most threads worth buying for this
   pipeline, and which change would raise the ceiling more — more cores, or
   making the save faster?

6. **Granularity.** Regenerate the image at 400 x 300 with
   `./bin/gen_image data/small.ppm 400 300` and run both versions on it.
   Does OpenMP still win? Relate your answer to where the crossover point is.

## What to hand in

- `src/main_omp.c` with the four tasks done
- The two measurement tables, filled in, saying which machine and how many cores
- Written answers to the six questions, one short paragraph each
- The output of `cmp`, showing your edge map matches

## Layout

```
openmp-lab/
├── Makefile
├── README.md
├── include/
│   ├── ppm.h          PPM reader and writer interface
│   └── filters.h      the four stages
├── src/
│   ├── ppm.c          do not modify
│   ├── filters.c      serial reference, do not modify
│   ├── gen_image.c    deterministic test image
│   ├── main_serial.c  baseline, do not modify
│   └── main_omp.c     YOUR FILE
├── solution/          reference solution (your instructor may withhold this)
├── build/             objects, generated
├── bin/               executables, generated
└── data/              images and histograms, generated
```

## Notes

- `-fopenmp` goes on **both** the compile and the link step. Leave it off and
  the pragmas are silently ignored: the program still runs, still gives the
  right answer, and is quietly sequential.
- Time with `omp_get_wtime()`, never `clock()`. `clock()` adds up CPU time
  across all threads, so a parallel program appears to take *longer* than the
  serial one even when it finished sooner.
- Measure with the `printf` calls commented out. Writing to the terminal is
  serialised by the operating system and will dominate everything else.
- Run each configuration several times and take the median. One run of a
  parallel program measures almost nothing.
