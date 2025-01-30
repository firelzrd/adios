# Adaptive Deadline I/O Scheduler (ADIOS)

## Key Features

*   **Optimized for Desktop Workloads:** ADIOS is particularly well-suited for desktop environments where users expect a smooth and responsive experience, even when running I/O-intensive tasks.

*   **Optimized for Non-Rotational Devices:** ADIOS is primarily designed and optimized for non-rotational storage devices such as Solid State Drives (SSDs), NVMe drives, USB thumb drives, SD cards and so on. While it might function on rotational drives, performance characteristics on those devices are not a primary focus.

*   **Adaptive Latency Control:** ADIOS continuously learns the latency profile of the storage device by monitoring the actual completion times of I/O requests. This learned profile is used to predict the completion time for subsequent requests, allowing it to adapt to varying I/O characteristics.

*   **Dynamic Deadline Adjustment:** Using the learned latency profile, ADIOS dynamically adjusts the deadline for each I/O request. It packs as many I/O requests as possible within a configurable global latency window, maximizing the benefits of batching while maintaining responsiveness.

*   **Double-Buffering Batch Queues:** ADIOS employs a double-buffering technique for its batch queues, allowing it to fill one queue while dispatching requests from the other. This helps to avoid dispatch stalls and keep the underlying storage devices busy.
    
*   **Latency Model Shrinking**: The latency model's statistics are periodically reduced to prevent the model from being overly influenced by outdated or irrelevant data. This keeps the model's memory footprint small while maintaining its accuracy.

*   **Derived from mq-deadline and Kyber:** ADIOS draws inspiration from `mq-deadline` and `Kyber` but represents a new design and implementation, incorporating and improving upon key concepts from these existing schedulers. It is **NOT** a direct derivative of either of them.

## How it Works

1.  **Latency Learning:** ADIOS monitors the completion time of each I/O request. It tracks the latency of different operation types and block sizes to build a parametric model of the storage device's performance.
2.  **Latency Prediction:** The learned latency profile is used to predict the time required to complete each incoming I/O request.
3.  **Adaptive Deadline Setting:** The predicted latency, along with the pre-defined target latency for the operation type, is used to set a deadline for each request.
4.  **Batching and Dispatch:** ADIOS attempts to dispatch as many I/O requests as possible within a global latency window, while also prioritizing synchronous operations and using double-buffered queues. This helps to improve overall throughput without sacrificing much responsiveness.

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

## Current Status

ADIOS is under active development and is considered experimental. Performance and tunables are still under investigation.

## Future Work
* More elaborate performance tests and comparison against other schedulers will be conducted.
* Find better default values for sysfs tunables.
* Port ADIOS to other Kernel versions.

## Special thanks
* Piotr Górski a.k.a. "sir_lucjan" from the CachyOS community, for helping me to bust some nasty bug related to bucket-based anomaly sample exclusion algorithm.

[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/Y8Y5NHO2I)

