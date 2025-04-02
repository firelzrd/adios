# Adaptive Deadline I/O Scheduler (ADIOS)

## Overview

ADIOS (Adaptive Deadline I/O Scheduler) is a block layer I/O scheduler for the Linux kernel, designed for modern multi-queue block devices (`blk-mq`). It aims to provide low latency for I/O operations by combining deadline scheduling principles with a learning-based adaptive latency control mechanism.

ADIOS is inspired by and builds upon concepts from the `mq-deadline` and `Kyber` I/O schedulers. Its core feature is the ability to predict I/O completion latency based on past performance and request characteristics (operation type, size) and use this prediction to dynamically adjust request deadlines and batching behavior.

## Demo

https://youtu.be/L9WDcEeHgy4

## Key Features

*   **Adaptive Latency Control:** ADIOS continuously learns the latency profile of the storage device by monitoring the actual completion times of I/O requests. This learned profile is used to predict the completion time for subsequent requests, allowing it to adapt to varying I/O characteristics.

*   **Dynamic Deadline Adjustment:** Using the learned latency profile, ADIOS dynamically adjusts the deadline for each I/O request. It packs as many I/O requests as possible within a configurable global latency window, maximizing the benefits of batching while maintaining responsiveness.

*   **Request Batching:** ADIOS groups requests into batches before dispatching to the hardware, aiming to improve efficiency while staying within a configurable global latency window.
    
*   **Dynamic Model Updates:** Continuously refines latency predictions based on the actual completion times of recent requests.

*   **Model Shrinkage:** Periodically reduces the weight of older samples in the latency model to adapt to changing device performance characteristics.

*   **Sysfs Tunability:** Provides various sysfs knobs to configure latency targets, batch limits, model behavior, and other parameters.

## How it Works
![alt adios-depicted](https://github.com/user-attachments/assets/1de83e36-31e2-4a72-bca8-efa39cf6747f)

1.  **Request Arrival:** When a new I/O request arrives, ADIOS determines its type (Read, Write, Discard, Other) and size.
2.  **Latency Prediction:** It queries the corresponding `latency_model` to predict how long this specific request is expected to take based on historical data (`latency_model_predict`).
3.  **Deadline Calculation:** A deadline is calculated for the request using: `request_start_time + target_latency[optype] + predicted_latency`.
4.  **Insertion:** The request is inserted into one of two deadline-sorted red-black trees (one primarily for reads, one for writes/discards) based on its calculated deadline.
5.  **Batch Queue Filling:** Periodically, or when the current batch queues are running low (`bq_refill_below_ratio`), ADIOS pulls requests with the earliest deadlines from the red-black trees (`fill_batch_queues`). It attempts to fill batches up to a `global_latency_window` based on the *predicted* latency of the requests being batched, respecting per-operation type `batch_limit`s.
6.  **Dispatch:** Requests are dispatched to the hardware primarily from these batch queues (`dispatch_from_bq`). A separate priority queue (`prio_queue`) handles requests inserted at the head (e.g., for request merging).
7.  **Completion & Learning:** When a request completes, its actual completion latency (`completion_time - io_start_time`) is measured. This measured latency, along with the request size and its predicted latency, is fed back into the `latency_model` (`latency_model_input`, `latency_model_update`). This allows the scheduler to learn and adapt its predictions over time. The model uses buckets to handle outliers and performs periodic updates.
8.  **Bias Adjustment:** A bias factor (`dl_bias`) is adjusted based on which queue (read-mostly vs. write-mostly) requests are dispatched from, influenced by the `read_priority` setting. This helps enforce the desired priority between reads and writes when deadlines are close.

## Sysfs Tunables

ADIOS provides several tunables via sysfs, allowing users to fine-tune its behavior. These attributes can be found under `/sys/block/<device_name>/queue/iosched/`

*   **`adios_version`**: (Read-only) Displays the current version of the ADIOS scheduler.

*   **`batch_size_actual_high`**: (Read-only) Shows the maximum batch size achieved by each operation type (Read, Write, Discard).

*   **`global_latency_window`**: (Read/Write) Sets the global latency window in nanoseconds. ADIOS attempts to dispatch as many I/O requests in a batch as possible within this window. Lowering this value will reduce latency and may limit throughput. Conversely, increasing it may improve throughput, at the cost of latency.

*   **`bq_refill_below_ratio`**: (Read/Write) Determines the ratio, in percentage of `global_latency_window`, below which the batch queues should be refilled. Lower values increase the likelihood of refilling batch queues, thus lower latency, at the expense of throughput. The default value is 15. Range should be between 0 and 100.
     
*   **`batch_limit_read`, `batch_limit_write`, `batch_limit_discard`**: (Read/Write) Configures the maximum batch size for each operation type (Read, Write, Discard). 

*   **`lat_target_read`, `lat_target_write`, `lat_target_discard`**: (Read/Write)  Sets the target latency (in nanoseconds) for each operation type.  ADIOS targets to complete corresponding requests within their target latency + predicted latency.

*   **`lat_model_read`, `lat_model_write`, `lat_model_discard`**: (Read-only) Shows the learned latency model parameters (base latency and slope) for the corresponding operation types. The base latency is the constant latency value that is incurred to the I/O completion, and slope represents the latency rate according to I/O request size.

*   **`read_priority`**: (Read/Write) This sets the priority for Read operations relative to other operations with the same deadline.  A higher positive value increases the likelihood of serving Read requests before Write and Discard requests when they share approximately the same deadline. The default value is 5. The range should be between -20 and 19.

*   **`reset_bq_stats`**: (Write-only) Writing `1` to this attribute resets the batch queue statistics, such as the actual high batch sizes, etc.

*   **`reset_latency_model`**: (Write-only) Writing `1` to this attribute resets the learned latency model to the initial, zero value.

## Build and Install

1.  Clone this repository.
2.  Navigate to the `block` directory.
3.  Build the module: `make`
4.  Load the kernel module: `sudo insmod adios.ko`
5.  To activate the ADIOS I/O scheduler for a block device, please check your Linux documentation. Usually, it can be done via `echo adios | sudo tee /sys/block/<device_name>/queue/scheduler`

## Future Work
* More elaborate performance tests and comparison against other schedulers will be conducted.
* Find better default values for sysfs tunables.
* Port ADIOS to other Kernel versions.

## Special thanks
* Piotr Górski a.k.a. "sir_lucjan" from the CachyOS community, for helping me to bust some nasty bug related to bucket-based anomaly sample exclusion algorithm.

[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/Y8Y5NHO2I)
