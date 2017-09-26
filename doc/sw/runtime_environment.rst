.. _runtime_environment:

===================
Runtime environment
===================

.. figure:: images/runtime_environment.png
    :scale: 70%
    :align: center

The runtime envionment includes software to run a compiled neural network on compatible :term:`NVDLA` hardware. It consists of 2 parts:

* :ref:`user_mode_driver` - This is the main interface to the application. As detailed in the :ref:`compilation_tools`, after parsing and compiling the neural network layer by layer, the compiled output is stored in a file format called :term:`NVDLA Loadable`. User mode runtime driver loads this loadable and submits inference jobs to the :ref:`kernel_mode_driver`.

* :ref:`kernel_mode_driver` - Consists of kernel mode driver and engine scheduler that does the work of scheduling the compiled network on :term:`NVDLA` and programming the :term:`NVDLA` registers to configure each functional block.

The runtime environment uses the stored representation of the network saved as :term:`NVDLA Loadable` image. From point of view of the :term:`NVDLA Loadable`, each compiled "layer" in software is loadable on a functional block in the :term:`NVDLA` implementation. Each layer includes information about its dependencies on other layers, the buffers that it uses for inputs and outputs in memory, and the specific configuration of each functional block used for its execution. Layers are linked together through a dependency graph, which the engine scheduler uses for scheduling layers. The format of an :term:`NVDLA Loadable` is standardized across compiler implementations and UMD implementations. All implementations that comply with the :term:`NVDLA` standard should be able to at least interpret any :term:`NVDLA Loadable` image, even if the implementation may not have some features that are required to run inferencing using that loadable image.

Both the :ref:`user_mode_driver` stack and the :ref:`kernel_mode_driver` stack exist as defined APIs, and are expected to be wrapped with a system portability layer. Maintaining core implementations within a portability layer is expected to require relatively few changes. This expedites any effort that may be necessary to run the :term:`NVDLA` software-stack on multiple platforms. With the appropriate portability layers in place, the same core implementations should compile as readily on both Linux and FreeRTOS. Similarly, on “headed” implementations that have a microcontroller closely coupled to :term:`NVDLA`, the existence of the portability layer makes it possible to run the same kernel mode driver on that microcontroller as would have run on the main CPU in a “headless” implementation that had no such companion microcontroller.

.. _user_mode_driver:

----------------
User Mode Driver
----------------

.. figure:: images/umd.png
    :scale: 70%
    :align: center

UMD provides standard :ref:`umd_api` (API) for processing loadable images, binding input and output tensors to memory locations, and submitting inference jobs to KMD. This layer loads the network into memory in a defined set of data structures, and passes it to the KMD in an implementation-defined fashion. On Linux, for instance, this could be an ``ioctl()``, passing data from the user-mode driver to the kernel-mode driver; on a single-process system in which the KMD runs in the same environment as the UMD, this could be a simple function call. Low-level functions are implemented in :ref:`user_mode_driver`

.. _umd_api:

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Application Programming Interface
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

NVDLA namespace
---------------

.. cpp:namespace:: nvdla

.. cpp:type:: NvError

   Enum for error codes

Runtime Interface
-----------------

This is the interface for runtime library. It implements functions to process loadable buffer passed from application after reding it from file, allocate memory for tensors and intermediate buffers, prepare synchronization points and finally submit inference job to KMD. Inference job submitted to KMD is referred as DLA task.

.. cpp:class:: nvdla::IRuntime

   Runtime interface

.. cpp:function:: nvdla::IRuntime* nvdla::createRuntime()

   Create runtime instance

   :returns: IRuntime object

**Device information**

.. cpp:function:: int nvdla::IRuntime::getMaxDevices()

   Get maximum number of device supported by HW configuration. Runtime driver supports submitting inference jobs to  multiple DLA devices. User application can select device to use. One task can't splitted across devices but one task can be submitted to only one devices.

   :returns: Maximum number of devices supported

.. cpp:function:: int nvdla::IRuntime::getNumDevices()

   Get number of available devices from the maximum number of devices supported by HW configuration.

   :returns: Number of available devices

**Loading NVDLA loadable image**

.. cpp:function:: NvError nvdla::IRuntime::load(const NvU8 *buf)

   Parse loadable from buffer and update ILoadable with information required to create task

   :param buf: Loadable image buffer
   :returns: :cpp:type:`NvError`

**Input tensors**

.. cpp:function:: NvError nvdla::IRuntime::getNumInputTensors(int *input_tensors)

   Get number of network's input tensors from loadable

   :param input_tensors: Pointer to update number of input tensors value
   :returns: :cpp:type:`NvError`

.. cpp:function:: NvError nvdla::IRuntime::getInputTensorDesc(int id, nvdla::ILoadable::TensorDescListEntry *tensors)

   Get network's input tensor descriptor

   :param id: Tensor ID
   :param tensors: Tensor descriptor
   :returns: :cpp:type:`NvError`

.. cpp:function:: NvError nvdla::IRuntime::setInputTensorDesc(int id, const nvdla::ILoadable::TensorDescListEntry *tensors)

   Set network's input tensor descriptor

   :param id: Tensor ID
   :param tensors: Tensor descriptor
   :returns: :cpp:type:`NvError`

**Output tensors**

.. cpp:function:: NvError nvdla::IRuntime::getNumOutputTensors(int *output_tensors)

   Get number of network's output tensors from loadable

   :param output_tensors: Pointer to update number of output tensors value
   :returns: :cpp:type:`NvError`

.. cpp:function:: NvError nvdla::IRuntime::getOutputTensorDesc(int id, nvdla::ILoadable::TensorDescListEntry *tensors)

   Get network's output tensor descriptor

   :param id: Tensor ID
   :param tensors: Tensor descriptor
   :returns: :cpp:type:`NvError`

.. cpp:function:: NvError nvdla::IRuntime::setOutputTensorDesc(int id, const nvdla::ILoadable::TensorDescListEntry *)

   Set network's output tensor descriptor

   :param id: Tensor ID
   :param tensors: Tensor descriptor
   :returns: :cpp:type:`NvError`

**Binding tensors**

.. cpp:function:: NvError nvdla::IRuntime::bindInputTensor(int id, NvDlaMemHandle hMem)

   Bind network's input tensor to memory handle

   :param id: Tensor ID
   :param hMem: DLA memory handle returned by :c:func:`NvDlaGetMem`
   :returns: :cpp:type:`NvError`

.. cpp:function:: NvError nvdla::IRuntime::bindOutputTensor(int id, NvDlaMemHandle hMem)

   Bind network's output tensor to memory handle

   :param id: Tensor ID
   :param hMem: DLA memory handle returned by :c:func:`NvDlaGetMem`
   :returns: :cpp:type:`NvError`

**Running inference**

.. cpp:function:: NvError nvdla::IRuntime::submit()

   Submit task for inference, it is blocking call

   :returns: :cpp:type:`NvError`

.. cpp:function:: NvError nvdla::IRuntime::submitNonBlocking(std::vector<ISync *> *outputSyncs)

   Submit non-blocking task for inference

   :param outputSyncs: List of output ISync objects
   :returns: :cpp:type:`NvError`

Sync Interface
--------------

Sync interface is used to synchronize between inference tasks. Software implementation can add more synchronization primitives of choice to the ISync wrapper.

.. cpp:class:: nvdla::ISync

   Sync interface

.. cpp:function:: NvError wait(NvU32 timeout)

   Blocks the caller until ISync object has signaled, or the timeout expires

   :param timeout: timeout The timeout value, in milliseconds
   :returns: :cpp:type:`NvError`

.. cpp:function:: NvError signal()

   Requests the ISync object to be signaled

   :returns: :cpp:type:`NvError`

.. cpp:function:: NvError nvdla::ISync::setWaitValue(NvU32 val)

   Set the comparison value for determining synchronization

   :param val: Comparison value to set
   :returns: :c:type:`NvError`

.. cpp:function:: NvError nvdla::ISync::getWaitValue(NvU32 *val) const

   Get the comparison value for determining synchronization

   :param val: Pointer to read wait value
   :returns: :c:type:`NvError`

.. cpp:function:: NvError nvdla::ISync::getValue(NvU32 *val) const

   Get the current value of the semaphore

   :param val: Pointer to read current value
   :returns: :c:type:`NvError`

Loadable Interface
------------------

Loadable contains compiled network and model data converted to DLA format. This interface implements functions to read data from loaded image.

.. cpp:class:: nvdla::ILoadable

   Loadable interface

.. cpp:class:: nvdla::ILoadable::TensorDescListEntry

.. _umd_layer:

^^^^^^^^^^^^^^^^^
Portability layer
^^^^^^^^^^^^^^^^^

.. c:type:: NvDlaEngineSelect

   Implementation defined enum to select device instance if there are multiple DLA devices

.. c:type:: NvDlaHandle

   Implementation defined handle used to communicate with portability layer from Runtime driver. :c:func:`NvDlaOpen` allocates this handle and returns to Runtime driver.

.. c:type:: NvDlaMemHandle

   Implementation defined memory handle used for memory operations implemented by portability layer. :c:func:`NvDlaGetMem` allocates and returns this handle to runtime driver for future operation on memory buffer allocated

.. c:type:: NvDlaTask

   DLA task structure. Runtime driver populates it using information from loadable and is used by portability layer to submit inference task to KMD in an implementation define manner.

.. c:type:: NvDlaFence

   Implementation defined sync object descriptor. It is populated by runtime driver in ISync implementation. This object is used by portability layer to send sync object information to KMD.

.. c:type:: NvDlaTaskStatus

   Task status structure used to report the task status to runtime.

.. c:type:: NvDlaHeap

   Implementation defined enum for memory heaps supported by the system.

.. c:type:: NvError

   Enum for error codes

.. c:function:: NvError NvDlaOpen(NvDlaEngineSelect DlaSelect, NvDlaHandle *phDla)

   This API should initialize portability layer which includes opening DLA device instance, allocating required structures, initializing session. It is implementation defined how integrator wants to implement portability layer. It should allocate NvDlaHandle and update phDla with it. This handle will be used for any future requests to portability layer for this session such as memory allocation, memory mapping or submit inference job.

   :param DlaSelect: Engine to initialize
   :param phDla: DLA handle updated if initializaton is successful
   :returns: :c:type:`NvError`

.. c:function:: void NvDlaClose(NvDlaHandle hDla)

   Close DLA device instance

   :param hDla: DLA handle returned by :c:func:`NvDlaOpen`

.. c:function:: NvError NvDlaSubmit(NvDlaHandle hDla, NvDlaTask *tasks, NvU32 num_tasks)

   Submit inference task to KMD

   :param hDla: DLA handle returned by :c:func:`NvDlaOpen`
   :param tasks: Lists of tasks to submit for inferencing
   :param num_tasks: Number of tasks to submit
   :returns: :c:type:`NvError`

.. c:function:: NvError NvDlaGetMem(NvDlaHandle hDla, NvDlaMemHandle *handle, void **pData, NvU32 size, NvDlaHeap heap, NvU32 flags)

   Allocate, pin and map DLA engine accessible memory. For example, in case of systems where DLA is behind IOMMU then this call should ensure that IOMMU mappings are created for this memory. In case of Linux, internal implementation can use readily available frameworks such as ION for this.

   :param hDla: DLA handle returned by :c:func:`NvDlaOpen`
   :param [out] handle: Memory handle updated by this function
   :param size: Size of buffer to allocate
   :param pData: If the allocation and mapping is successful, provides a virtual address through which the memory buffer can be accessed.
   :param heap: Implementation defined memory heap selection
   :param flags: Implementation defined
   :returns: :c:type:`NvError`

.. c:function:: NvError NvDlaFreeMem(NvDlaHandle hDla, NvDlaMemHandle handle, void *pData, NvU32 size)

   Free DMA memory allocated using :c:func:`NvDlaGetMem`

   :param hDla: DLA handle returned by :c:func:`NvDlaOpen`
   :param handle: Memory handle returned by :c:func:`NvDlaGetMem`
   :param pData: Virtual address returned by :c:func:`NvDlaGetMem`
   :param size: Size of the buffer allocated
   :returns: :c:type:`NvError`

.. c:function:: NvError NvDlaMemMap(NvDlaMemHandle hMem, NvU32 Offset, NvU32 Size, NvU32 Flags, void **pVirtAddr)

   Attempts to map a memory buffer into the process's virtual address space.

   :param hMem: A memory handle returned from :c:func:`NvDlaGetMem`
   :param Offset: Byte offset within the memory buffer to start the map at.
   :param Size: Size in bytes of mapping requested.  Must be greater than 0.
   :param Flags: Implementation defined
   :param pVirtAddr: If the mapping is successful, provides a virtual address through which the memory buffer can be accessed.
   :returns: :c:type:`NvError`

.. c:function:: void NvDlaMemUnmap(NvDlaMemHandle hMem, void *pVirtAddr, NvU32 length)

   Unmaps a memory buffer from the process's virtual address space.

   :param hMem: A memory handle returned from :c:func:`NvDlaGetMem`
   :param pVirtAddr: The virtual address returned by a previous call to :c:func:`NvDlaMemMap` with hMem.
   :param length: The size in bytes of the mapped region. Must be the same as the size value originally passed to :c:func:`NvDlaMemMap`.

.. c:function:: void NvDlaMemRead(NvDlaMemHandle hMem, NvU32 Offset, void *pDst, NvU32 Size)

   Reads a block of data from a buffer.

   :param hMem: A memory handle returned from :c:func:`NvDlaGetMem`
   :param Offset: Byte offset relative to the base of hMem.
   :param pDst: The buffer where the data should be placed.
   :param Size: The number of bytes of data to be read.

.. c:function:: void NvDlaMemWrite(NvDlaMemHandle hMem, NvU32 Offset, const void *pSrc, NvU32 Size)

   Writes a block of data to a buffer

   :param hMem: A memory handle returned from :c:func:`NvDlaGetMem`
   :param Offset: Byte offset relative to the base of hMem.
   :param pSrc: The buffer to obtain the data from.
   :param Size: The number of bytes of data to be written.

.. c:function:: NvU64 NvDlaMemGetSize(NvDlaMemHandle hMem)

   Get the size of the buffer associated with a memory handle

   :param hMem: A memory handle returned from :c:func:`NvDlaGetMem`
   :returns: Size in bytes of memory allocated for this handle or 0 in case of error.

.. c:function:: void NvDlaDebugPrintf(const char *format, ...)

   Outputs a message to the debugging console, if present.

   :param format: A pointer to the format string

.. c:function:: void *NvDlaAlloc(size_t size)

   Dynamically allocates memory. Alignment, if desired, must be done by the caller.

   :param size: The size of the memory to allocate

.. c:function:: void NvDlaFree(void *ptr)

   Frees a dynamic memory allocation. Freeing a null value is okay

   :param ptr: A pointer to the memory to free, which should be from :c:func:`NvDlaAlloc`.

.. c:function:: void NvDlaSleepMS(NvU32 msec)

   Unschedule calling thread for at least the given number of milliseconds. Other threads may run during the sleep time.

   :param msec: The number of milliseconds to sleep.

.. c:function:: NvU32 NvDlaGetTimeMS(void)

   Return the system time in milliseconds. The returned values are guaranteed to be monotonically increasing, but may wrap back to zero (after about 50 days of runtime). In some systems, this is the number of milliseconds since power-on, or may actually be an accurate date.

   :returns: System time in milliseconds

.. _kernel_mode_driver:

------------------
Kernel Mode Driver
------------------

.. figure:: images/kmd.png
    :scale: 70%
    :align: center

The KMD main entry point receives an inference job in memory, selects from multiple available jobs for execution (if on a multi-process system), and submits it to the core engine scheduler. This core engine scheduler is responsible for handling interrupts from :term:`NVDLA`, scheduling layers on each individual functional block, and updating any dependencies based upon the completion of the layer. The scheduler uses information from the dependency graph to determine when subsequent layers are ready to be scheduled; this allows the compiler to decide scheduling of layers in an optimized way, and avoids performance differences from different implementations of the KMD.

.. figure:: images/kmd_interface.png
    :scale: 70%
    :align: center

^^^^^^^^^
Interface
^^^^^^^^^

.. c:type:: dla_task_descriptor

   Task descriptor structure. This structure includes all the information required to execute a network such as number of layers, dependency graph address etc.

.. c:function:: int dla_execute_task(struct dla_task_descriptor *task);

   Task is submitted to engine scheduler for execution. Engine scheduler initiates all functional blocks if a layer is present for that functional block. Driver should process all the events after this and wait till all layers are completed.

   :param task: Task descriptor
   :returns: 0 if success otherwise error code

.. c:function:: int dla_engine_init(void *priv_data)

   Initialize engine scheduler. This function should be called when driver is probed at boot time.

   :param priv_data: Any data which is required back when scheduler engine calls function implemented in portability layer
   :returns: 0 if success otherwise error code

.. c:function:: int dla_process_events(void);

   Process events recorded in interrupt handler. This function must be called from thread/process context and not from interrupt context. It reads the events recorded in interrupt handler, updates dependency using events information and programs next layers in network. Driver should call this function immediately after handling interrupt and it's execution must be atomic, protected by some lock.

   :returns: 0 if success otherwise error code

.. c:function:: int dla_isr_handler(void);

   Interrupt handler records events by reading interrupt status registers. Driver should call this function from OS interrupt handler and it's execution must be atomic, protected by some lock.

   :returns: 0 if success otherwise error code

^^^^^^^^^^^^^^^^^
Portability layer
^^^^^^^^^^^^^^^^^

Driver should implement below functions which are called from engine scheduler.

Register read/write
-------------------

.. c:function:: void dla_reg_write(uint32_t addr, uint32_t reg)

   Register write. This function implementation depends on how is DLA accessible from CPU. It should take care of adding base address to `addr`.

   :param addr: Register offset starting from 0 as base address
   :param reg: Value to write

.. c:function:: uint32_t dla_reg_read(uint32_t addr)

   Register read. This function implementation depends on how is DLA accessible from CPU. It should take care of adding base address to `addr`

   :param addr: Register offset starting from 0 as base address
   :returns: Register value

Read address
------------

.. c:function:: int32_t dla_read_dma_address(struct dla_task_desc *task_desc, int16_t index, void *dst)

   Read DMA address from address list at index specified. This function is used by functional block programming operations to read address for DMA engines in functional blocks.

   :param task_desc: Task descriptor for in execution task
   :param index: Index in address list
   :param dst: Destination pointer to update address
   :returns: 0 in case success, error code in case of failure

.. c:function:: int32_t dla_read_cpu_address(struct dla_task_desc *task_desc, int16_t index, void *dst)

   Read CPU accessible address from address list at index specified. This function is used by engine scheduler to read data from memory buffer. Address returned by this function must be accessible by processor running engine scheduler.

   :param task_desc: Task descriptor for in execution task
   :param index: Index in address list
   :param dst: Destination pointer to update address
   :returns: 0 in case success, error code in case of failure

Data read/write
---------------

.. c:function:: int32_t dla_data_read(uint64_t src, void* dst, uint32_t size, uint32_t offset)

   Read data from src buffer to dst. Here src is memory buffer shared by UMD and dst is local structure in KMD.

   :param src: Source address to read data from
   :param dst: Destination to write data data
   :param size: Size of data to read
   :param offset: Offset from source address to read data
   :returns: 0 in case success, error code in case of failure

.. c:function:: int32_t dla_data_write(void* src, uint64_t dst, uint32_t size, uint32_t offset)

   Write data from src to dst. Here src is local structure in KMD and dst is memory buffer shared by UMD.

   :param src: Source to read data
   :param dst: Destination address to write data
   :param size: Size of data to write
   :param offset: Offset from destination address to write data
   :returns: 0 in case success, error code in case of failure
