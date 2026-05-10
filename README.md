
# Apache Arrow IPC Serialization Analysis

A systems-level reverse engineering and experimental analysis of Apache Arrow IPC internals, zero-copy semantics, buffer-oriented transport, and analytical serialization behavior using instrumented Apache Arrow source code.

---

## Table of Contents

1. [Overview](#overview)
2. [Problem Statement](#problem-statement)
3. [Project Objectives](#project-objectives)
4. [System Under Study](#system-under-study)
5. [Execution Path Analysis](#execution-path-analysis)
6. [Source Code Instrumentation](#source-code-instrumentation)
7. [Systems Design Decisions](#systems-design-decisions)
8. [Experimental Setup](#experimental-setup)
9. [Experiments](#experiments)
10. [Failure Analysis](#failure-analysis)
11. [Reproducing the Experiments](#reproducing-the-experiments)
12. [Project Structure](#project-structure)
13. [Learning Outcomes](#learning-outcomes)
14. [References](#references)

---

## Overview

This project focuses on reverse engineering Apache Arrow IPC internals through:

- source code analysis
- execution tracing
- instrumentation of internal serialization logic
- experimental benchmarking
- systems-level observation
- failure analysis

The objective is **not** to demonstrate API usage, but to understand:

- how Arrow IPC internally serializes data
- why specific architectural decisions were made
- what tradeoffs those decisions introduce
- how the system behaves under stress conditions

The project analyzes actual Apache Arrow source code and traces the execution path of Arrow IPC serialization from the Python API down to low‑level buffer construction inside the Arrow C++ implementation.

---

## Problem Statement

Modern analytical systems spend significant time:

- parsing serialized data
- reconstructing rows
- copying memory buffers
- converting formats between systems

Traditional row‑oriented and text‑based formats such as JSON and CSV introduce substantial overhead during:

- analytical ingestion
- machine learning preprocessing
- cross‑language data exchange
- inter‑process communication

Apache Arrow IPC attempts to solve this problem using:

- columnar memory layout
- contiguous typed buffers
- schema‑aware serialization
- zero‑copy semantics
- buffer‑oriented transport

This project investigates how these mechanisms are implemented internally and how they affect performance behavior.

---

## Project Objectives

The primary objectives of this project were:

1. Trace the complete Arrow IPC serialization execution path
2. Analyze Arrow IPC internal buffer construction
3. Study zero‑copy memory reuse semantics
4. Benchmark Arrow IPC against alternative serialization systems
5. Observe internal serializer behavior through instrumentation
6. Analyze batching tradeoffs and metadata overhead
7. Investigate failure scenarios and stress behavior
8. Connect Arrow IPC internals to systems concepts taught in class

---

## System Under Study

**Project Topic:** Apache Arrow IPC Serialization

**Primary implementation analyzed:**

```
arrow/cpp/src/arrow/ipc/writer.cc
```

**Core function analyzed:**

```cpp
RecordBatchSerializer::Assemble()
```

---

## Execution Path Analysis

The following execution path was traced during Arrow IPC serialization:

```
Python API
    ↓
pyarrow.ipc.new_stream()
    ↓
SerializeRecordBatch()
    ↓
WriteRecordBatch()
    ↓
RecordBatchSerializer::Assemble()
    ↓
VisitArray()
    ↓
Buffer metadata generation
    ↓
IPC payload construction
    ↓
Serialized IPC stream output
```

The project traced this execution path directly inside `arrow/cpp/src/arrow/ipc/writer.cc`.

**Important internal structures observed:**

- `field_nodes_`
- `body_buffers`
- `buffer_meta_`
- `variadic_counts_`
- `payload.body_length`
- `raw_body_length`

This execution tracing enabled direct observation of how Arrow transforms columnar arrays into IPC payloads.

---

## Source Code Instrumentation

The Arrow IPC serializer was modified directly inside `arrow/cpp/src/arrow/ipc/writer.cc`.

Instrumentation was added inside:

```cpp
Status Assemble(const RecordBatch& batch)
```

**Instrumentation captured:**

- serialization timing
- row counts
- column counts
- field node counts
- body buffer counts
- payload sizes
- IPC body lengths

**Example instrumentation output:**

```
CUSTOM IPC BUILD ACTIVE
RecordBatch Serialization Started
Rows          : 100000
Columns       : 4
Field Nodes   : 4
Buffers       : 8
Body Buffers  : 8
Raw Body Size : 3200000 bytes
IPC Body Size : 3200048 bytes
Assemble Time : 2150 us
Serialization Complete
```

This instrumentation enabled direct observation of internal IPC serialization behavior during runtime.

---

## Systems Design Decisions

### 1. Columnar Memory Layout

Arrow IPC stores data using contiguous typed columns instead of row‑oriented structures.

**Benefit:** improves cache locality, enables vectorized execution, minimizes parsing overhead, accelerates analytical scans.

**Tradeoff:** inefficient for transactional row‑wise access, requires more complex buffer management, increases implementation complexity.

### 2. Zero‑Copy Slicing

Arrow arrays support offset‑based logical slicing without physical copying.

**Benefit:** minimizes memory allocation, reduces copying overhead, enables fast analytical subsetting.

**Tradeoff:** shared buffers can retain unused memory, complicates offset rebasing logic, nested structures become harder to serialize.

Observed directly in: `GetZeroBasedValueOffsets()`, `GetTruncatedBuffer()`, `SliceBuffer()`.

### 3. Batch‑Oriented IPC Transport

Arrow IPC transports data using RecordBatches.

**Benefit:** amortizes metadata overhead, improves throughput, enables efficient streaming.

**Tradeoff:** small batches become metadata‑heavy, large batches increase memory pressure, batching strategy affects latency behavior.

### 4. Schema‑Aware Serialization

Arrow IPC embeds explicit schema metadata into serialized payloads.

**Benefit:** enables cross‑language interoperability, preserves data types, avoids expensive inference.

**Tradeoff:** introduces metadata framing overhead, increases payload complexity.

---

## Experimental Setup

### Environment

- Ubuntu WSL
- Python 3.12
- PyArrow editable build
- Apache Arrow local source build
- Jupyter Notebook

### Technologies Used

| Component | Purpose |
|-----------|---------|
| Python | Experimentation |
| PyArrow | Arrow bindings |
| Apache Arrow C++ | IPC implementation |
| NumPy | Dataset generation |
| Pandas | Data processing |
| Matplotlib | Visualization |
| Jupyter Notebook | Experiment execution |

---

## Experiments

The project includes seven main experiments. Full details and code are available in the `notebooks/` directory. Key findings are summarized below.

### 1. Serialization Format Benchmark

Compared Arrow IPC against JSON, CSV, Pickle, Parquet, SQLite, HDF5, and MessagePack.

**Observed Behavior:** Arrow IPC consistently achieved the fastest deserialization latency. JSON incurred heavy parsing overhead. CSV required expensive row reconstruction and type inference. Parquet optimized storage size more aggressively than latency. Pickle performed reasonably well but is Python‑specific, insecure for untrusted data, and not cross‑language.

**Systems Insight:** Arrow IPC preserves typed columnar memory buffers during serialization, reducing preprocessing overhead because analytical systems can directly consume structured memory instead of reconstructing rows from text.

**Tradeoff:** Arrow IPC prioritises analytical transport speed over human readability and storage compactness.

### 2. Zero‑Copy Slicing Experiment

Compared slice views vs physically copied arrays.

**Observed Behavior:** Slice views reused underlying buffers (same memory address). Copied arrays allocated new memory. Slice creation was nearly instantaneous (0.001s vs 1.89s for copy). Serialised payload size remained identical.

**Systems Insight:** Arrow achieves efficient slicing through immutable offset‑based logical views rather than physically copying data buffers.

**Tradeoff:** Zero‑copy semantics reduce allocations and improve latency but increase serializer complexity because offsets and nested structures must still be normalised correctly.

### 3. ML Pipeline Ingestion Benchmark

Measured loading and preprocessing latency for Arrow IPC, CSV, JSON, Pickle, and Parquet.

**Observed Behavior (1 million rows):**

| Format     | Load (s) | Preprocess (s) | Total (s) |
|------------|----------|----------------|-----------|
| Arrow IPC  | 0.0896   | 0.0730         | 0.1626    |
| Pickle     | 0.2554   | 0.0504         | 0.3058    |
| CSV        | 0.5864   | 0.1044         | 0.6908    |
| Parquet    | 0.7526   | 0.0595         | 0.8120    |
| JSON       | 5.9731   | 0.0925         | 6.0656    |

**Systems Insight:** Serialisation overhead directly affects analytical and ML startup performance. Faster ingestion reduces the delay before preprocessing and execution can begin.

**Tradeoff:** Formats optimised for storage compactness are not necessarily optimised for analytical execution latency.

### 4. Arrow IPC Batch Effects Experiment

Analyzed how RecordBatch size affects serialization efficiency.

**Observed Behavior:**

| Batch size | Num batches | Serialisation time (s) | Avg batch cost (ms) |
|------------|-------------|------------------------|----------------------|
| 1,000      | 1000        | 0.3450                 | 0.345                |
| 10,000     | 100         | 0.2426                 | 2.43                 |
| 50,000     | 20          | 0.0811                 | 4.06                 |
| 100,000    | 10          | 0.0569                 | 5.69                 |
| 250,000    | 4           | 0.0481                 | 12.03                |
| 500,000    | 2           | 0.0397                 | 19.87                |
| 1,000,000  | 1           | 0.0287                 | 28.67                |

**Systems Insight:** Arrow IPC is fundamentally batch‑oriented. Metadata structures (field nodes, buffer metadata) are emitted per batch, making batch size a major performance parameter.

**Tradeoff:** Small batches improve granularity and responsiveness; larger batches improve throughput but increase memory pressure.

### 5. Arrow IPC Compression Effects Experiment

Studied compression impact using LZ4 and ZSTD.

**Observed Behavior (1 million rows, 6 columns):**

| Codec       | Ser time (s) | Payload size (MB) | Space savings |
|-------------|--------------|-------------------|----------------|
| Uncompressed| 0.059        | 59.99             | 0%             |
| LZ4         | 0.225        | 34.19             | 43.0%          |
| ZSTD        | 0.597        | 25.66             | 57.2%          |

**Systems Insight:** Arrow IPC treats compression as an optional optimisation layer applied after buffer construction, not as a core serialisation requirement.

**Tradeoff:** Compression improves transport efficiency and storage usage but increases CPU cost and deployment complexity.

### 6. Arrow IPC Nested Array Effects Experiment

Analyzed how nested structures (ListArray, StructArray, DeepNested) affect serialization.

**Observed Behavior:**

| Structure   | Buffers | Serialisation time (s) | Payload size (MB) |
|-------------|---------|------------------------|-------------------|
| Primitive   | 2       | 0.0092                 | 3.05              |
| List        | 4       | 0.0052                 | 5.34              |
| Struct      | 5       | 0.0031                 | 3.05              |
| DeepNested  | Many    | 0.0142                 | 8.39              |

**Systems Insight:** Arrow serialises nested arrays using recursive traversal through child arrays. Nested layouts require additional offset handling, validity bitmaps, and recursive buffer management.

**Tradeoff:** Nested columnar structures improve analytical flexibility but increase serialisation complexity and metadata overhead.

### 7. Arrow IPC Memory Layout Visualization

Visualized buffer organization for primitive, list, and struct arrays.

**Observed Behavior:** Primitive arrays use compact contiguous value buffers. Variable‑length arrays require separate offset buffers. Nested arrays increase total buffer count significantly. Validity bitmaps are stored separately from value buffers. IPC serialisation adds an 8‑byte continuation token and message header.

**Systems Insight:** Arrow IPC organises data into independent aligned buffers rather than row‑oriented records, improving vectorised analytical processing and cache efficiency.

**Tradeoff:** Columnar buffer layouts improve analytical execution but make serialisation logic more complex than traditional row‑oriented systems.

---

## Failure Analysis

### 1. Tiny Batch Overhead

**Observed:** Small RecordBatches produced excessive metadata overhead, reduced throughput, and serialisation fragmentation.

**Systems Insight:** Batch‑oriented systems depend heavily on amortising metadata costs across sufficiently large payloads.

**Tradeoff:** Tiny batches improve latency granularity but reduce transport efficiency.

### 2. Large Batch Memory Pressure

**Observed:** Large RecordBatches increased memory allocation costs, contiguous buffer pressure, and serialisation latency.

**Systems Insight:** Throughput‑oriented batching strategies improve amortisation efficiency but increase memory pressure during serialisation.

**Tradeoff:** Larger batches improve throughput but reduce memory efficiency and increase allocation overhead.

---

## Reproducing the Experiments

### Clone Repository

```bash
git clone https://github.com/Megh39/arrow-ipc-analysis
cd arrow-ipc-analysis
```

### Create Environment

```bash
python3 -m venv venv
source venv/bin/activate   # On Windows: venv\Scripts\activate
```

### Install Dependencies

```bash
pip install -r requirements.txt
```

### Launch Jupyter Notebook

```bash
jupyter notebook
```

All experiments are available in the `notebooks/` directory.

### Building Arrow from Source (Optional)

To modify and instrument the Arrow C++ source code:

```bash
# Install build dependencies (Ubuntu example)
sudo apt install build-essential cmake python3-dev

# Clone Arrow repository
git clone https://github.com/apache/arrow.git
cd arrow
git submodule update --init --recursive

# Build Arrow C++ with Python bindings
mkdir cpp/build
cd cpp/build
cmake .. -DCMAKE_BUILD_TYPE=debug -DARROW_PYTHON=ON -DARROW_PARQUET=ON -DARROW_DATASET=ON
make -j4 install

# Install PyArrow in editable mode
cd ../..
pip install -e python/ --no-deps
```

---

## Project Structure

```
arrow-ipc-analysis-linux/
│
├── notebooks/
│   ├── 01_serialization_benchmark.ipynb
│   ├── 02_zero_copy_slicing.ipynb
│   ├── 03_ml_pipeline_benchmark.ipynb
│   ├── 04_arrow_ipc_batch_effects.ipynb
│   ├── 05_arrow_ipc_compression_effects.ipynb
│   ├── 06_arrow_ipc_nested_array_effects.ipynb
│   └── 07_arrow_ipc_memory_layout_visualization.ipynb
│
├── results/
│   ├── serialization/
│   ├── slicing/
│   ├── ml_pipeline/
│   ├── batch_effects/
│   ├── compression_effects/
│   ├── nested_array_effects/
│   └── memory_layout_visualization/
│
├── modified_arrow/          (optional, for source modifications)
├── report/
├── scripts/
├── requirements.txt
└── README.md
```

---

## Learning Outcomes

This project explored:

- IPC serialization internals
- columnar memory systems
- buffer‑oriented transport
- analytical execution behavior
- zero‑copy data structures
- systems instrumentation
- serialization overhead
- batching tradeoffs
- memory management behavior
- performance benchmarking

---

## References

- [Apache Arrow Documentation](https://arrow.apache.org/docs/)
- [Apache Arrow IPC Specification](https://arrow.apache.org/docs/format/Columnar.html)
- [PyArrow Documentation](https://arrow.apache.org/docs/python/)
- [Apache Arrow Source Code](https://github.com/apache/arrow)
- [FlatBuffers Documentation](https://google.github.io/flatbuffers/)

---

## Author

Project developed as part of a systems analysis and Big Data Engineering study focused on Apache Arrow IPC internals, serialization behavior, and analytical systems performance.
