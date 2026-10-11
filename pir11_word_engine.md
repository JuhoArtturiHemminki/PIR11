# PIR11 Word Engine Technical Documentation

**Author:** Juho Artturi Hemminki  
**License Inquiries:** projectflagcarrier@gmail.com
**Date:** October 11, 2026  
**Component:** `Pir11PageEngine` Low-Level Architecture  

---

The **PIR11 Word Engine** serves as the lowest-level execution unit in this architecture, operating directly beneath the page-level management layer (`Pir11PageEngine`). 

In the context of this codebase, a **word** is defined as a single 64-bit unsigned integer (`uint64_t`). The Word Engine is fundamentally responsible for how the processor ingests, processes, and filters these individual 64-bit words before the intermediate results are accumulated at the page level.

Within PIR system, the structural hierarchy and division of labor operate as follows:

### 1. Data Type: Defining the "Word"
The database page (`PAGE_SIZE_BYTES = 4096` bytes) is segmented into exactly 512 individual 64-bit words (`WORDS_PER_PAGE`). The Word Engine operates strictly at this granularity: it processes data by moving a single 64-bit register value into the execution pipeline at a time.

### 2. Task: Low-Level Bit Pattern Extraction
The critical and singular objective of the Word Engine is to execute low-level bitmasking and population count (`popcount`) operations using the absolute minimum number of CPU cycles. It transforms 64 bits of raw, unstructured data into a single scalar integer (ranging from 0 to 64), which represents the total number of "hits" or active bits set within that specific word.

### 3. Integration: Connecting the Word Engine to the Page Engine
The unrolled loop implemented in your kernel merges **four distinct Word Engine operations** into a single, high-throughput Page Engine iteration:

```cpp
// 4 separate Word Engine executions running in parallel via ILP:
ones_accumulator += __builtin_popcountll(page_data[i]);     // Word Engine 1
ones_accumulator += __builtin_popcountll(page_data[i + 1]); // Word Engine 2
ones_accumulator += __builtin_popcountll(page_data[i + 2]); // Word Engine 3
ones_accumulator += __builtin_popcountll(page_data[i + 3]); // Word Engine 4
```

Depending on the compiler flags and target architecture, this execution maps to hardware in two distinct ways:

* **Scalar Mode:** The CPU's execution engine dispatches four independent words and schedules them across hardware `POPCNT` units. Because modern superscalar processors can execute multiple independent popcount instructions per clock cycle (utilizing high Instruction-Level Parallelism, or ILP), these Word Engine iterations execute concurrently and overlap in the pipeline.
* **SIMD / Vector Mode (e.g., AVX2 / AVX-512):** The individual 64-bit Word Engines fuse together into a single, wide vector lane. The processor issues a single vectorized memory load (such as a 256-bit or 512-bit chunk) and computes the population count for all underlying words simultaneously using a dedicated vector instruction (e.g., `_mm256_popcnt_epi64` or `_mm512_popcnt_epi64`).

---

### Summary of Architectural Roles
* **Word Engine:** The atomic, data-parallel execution unit processing 64 bits at a time.
* **Page Engine (`Pir11PageEngine`):** The control and aggregation layer that collects the 512 micro-results from the Word Engines, accumulates them into `ones_accumulator`, evaluates the entropy barrier (`is_collapsed`), and enforces the necessary compiler memory barriers.

---

**Author:** Juho Artturi Hemminki  
**License Inquiries:** projectflagcarrier@gmail.com
