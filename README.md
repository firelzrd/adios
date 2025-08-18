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

ADIOS processes I/O in four priority tiers based on the nature of the request.

1.  **Tier 0 (Highest Priority): Emergency & System Integrity Requests**
    *   Targets requests with the `BLK_MQ_INSERT_AT_HEAD` flag. These are for critical operations like device error recovery that must bypass all other scheduling logic. They are placed in a dedicated FIFO queue (`prio_queue[0]`) for immediate dispatch.

2.  **Tier 1 (High Priority): Data Persistence & Ordering Guarantees**
    *   Targets requests with integrity-sensitive flags like `REQ_FUA` or `REQ_PREFLUSH`. These are handled in a separate FIFO queue (`prio_queue[1]`) to ensure their submission order is guaranteed.

3.  **Tier 2 (Medium Priority): Application Responsiveness**
    *   Targets normal synchronous requests. The deadline for these requests is set to their start time (`rq->start_time_ns`). This effectively enforces FIFO-like behavior within the deadline-sorted tree, guaranteeing that dependent synchronous operations execute in order.

4.  **Tier 3 (Normal Priority): Background Throughput**
    *   Targets asynchronous requests. These are the only requests where ADIOS's adaptive latency prediction model is used. A dynamic deadline is calculated based on the predicted I/O latency, allowing for aggressive reordering to optimize I/O efficiency.

**Dispatch Logic:**
The scheduler always dispatches requests in strict priority order: Tier 0 -> Tier 1 -> Tier 2/3. Tiers 2 and 3 are managed in a single deadline-sorted data structure (a red-black tree), which naturally prioritizes them based on their calculated deadlines (Tier 2 is prioritized over Tier 3). Requests are moved from this red-black tree into batch queues before being finally dispatched to the hardware.

When a request completes, its actual completion latency is measured and fed back into the latency model to improve future predictions.

## Sysfs Tunables

ADIOS provides several tunables via sysfs, allowing users to fine-tune its behavior. These attributes can be found under `/sys/block/<device_name>/queue/iosched/`

*   **`adios_version`**: (Read-only) Displays the current version of the ADIOS scheduler.

*   **`batch_actual_max`**: (Read-only) Shows the maximum batch size achieved by each operation type (Read, Write, Discard).

*   **`global_latency_window`**: (Read/Write) Sets the global latency window in nanoseconds. ADIOS attempts to dispatch as many I/O requests in a batch as possible within this window. Lowering this value will reduce latency and may limit throughput. Conversely, increasing it may improve throughput, at the cost of latency.

*   **`bq_refill_below_ratio`**: (Read/Write) Determines the ratio, in percentage of `global_latency_window`, below which the batch queues should be refilled. Lower values increase the likelihood of refilling batch queues, thus lowering latency, at the expense of throughput. The default value is 25. Range should be between 0 and 100.
     
*   **`batch_limit_read`, `batch_limit_write`, `batch_limit_discard`**: (Read/Write) Configures the maximum batch size for each operation type (Read, Write, Discard). 

*   **`lat_target_read`, `lat_target_write`, `lat_target_discard`**: (Read/Write) Sets the target latency (in nanoseconds) for each operation type. ADIOS targets to complete corresponding requests within their target latency + predicted latency.

*   **`lat_model_read`, `lat_model_write`, `lat_model_discard`**: (Read/Write) Shows or sets the learned latency model parameters (base latency and slope) for the corresponding operation types. The base latency is the constant latency value that is incurred for I/O completion, and the slope represents the latency rate according to I/O request size. You can write initial values to the model by writing a string in the format `"base_value slope_value"`.

*   **`place_deadline_pred_lat`**: (Read/Write) Controls whether the predicted latency (`pred_lat`) is included in the deadline calculation for asynchronous requests. Enabling this (1) can prioritize small I/Os over large transfers, potentially improving system responsiveness. For maximum compatibility, it is disabled by default (0).

*   **`lat_model_latency_limit`**: (Read/Write) Sets an upper limit, in nanoseconds, on the latency samples that the model will learn from. Latency samples exceeding this value will be ignored.

*   **`read_priority`**: (Read/Write) This sets the priority for Read operations relative to other operations with a similar deadline. A higher positive value increases the likelihood of serving Read requests before Write and Discard requests when they share approximately the same deadline. The default value is 7. The range should be between -20 and 19.

*   **`shrink_at_kreqs`, `shrink_at_gbytes`, `shrink_resist`**: (Read/Write) Dynamic thresholds that control the latency model's shrinkage behavior (i.e., how it forgets old samples).

*   **`reset_bq_stats`**: (Write-only) Writing `1` to this attribute resets the batch queue statistics, such as the actual high batch sizes.

*   **`reset_lat_model`**: (Write-only) Writing `1` to this attribute resets the learned latency model to its initial, zero value. Alternatively, you can load initial values for all operation types by writing six space-separated values in the format `"R_base R_slope W_base W_slope D_base D_slope"`.

## Build and Install

1.  Clone this repository.
2.  Navigate to the `block` directory.
3.  Build the module: `make`
4.  Load the kernel module: `sudo insmod adios.ko`
5.  To activate the ADIOS I/O scheduler for a block device, please check your Linux documentation. Usually, it can be done via `echo adios | sudo tee /sys/block/<device_name>/queue/scheduler`

## Special thanks
*   Piotr Górski a.k.a. "sir_lucjan" from the CachyOS community, for helping me to bust some nasty bug related to the bucket-based anomaly sample exclusion algorithm.

[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/Y8Y5NHO2I)
