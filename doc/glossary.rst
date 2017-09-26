=====================
Glossary And Acronyms
=====================


.. glossary::

    NVDLA
       NVIDIA Deep Learning Accelerator. Hardware logic implementing different function engines.

    DLA
       Deep Learning Accelerator

    UMD
       User Mode Driver, software running in user space. It provides interfaces to user application

    KMD
       Kernel Mode Driver, software running in kernel space. It implement low level functions such as hardware 
       programming and provides interfaces to UMD

    NVDLA Loadable
       Format used to store compiled neural network.

    hw-layer
       One complete hardware block processing. It starts with a set of register configuration with an
       enable field. When it is done, it triggers ONE interrupt.

    SDP
       Single data processor, a functional sub unit in NVDLA engine

    PDP
       Planar data processor, a functional sub unit in NVDLA engine

    CDP
       Channel data processor, a functional sub unit in NVDLA engine

    Rubik
       Data operation processor. A functional sub unit in NVDLA engine to 
       support data layout transformation

    BDMA
       Bridge DMA. BDMA transform data from CVSRAM to MC and vice versa

    CDMA
       Convolution DMA, responsible for load image/weight/feature data to convolution core

    RDMA
       Read DMA, instanced in SDP/CDP/PDP/Rubik engines which responsible for load 
       input data from external memory (either MC or CVSRAM)

    WDMA
       Write DMA, instanced in SDP/CDP/PDP/Rubik engines and responsible for write 
       output data to external memory (either MC or CVSRAM)

    CBB
       Control backbone. System control data path that connected to NVDLA

    DBB
       Data backbone. The memory system that to which NVDLA connects

    AMBA
       ARM Advanced Microcontroller Bus Architecture. A set of ARM defined bus standards.

    AXI
       Advanced eXtensible Interface. The third generation of AMBA interface defined in the AMBA 3 specification.

    CNN
       Convolutional Neural Network. A class of artificial neural networks.

    CSB
       Configuration Space Bus.  (See :ref:`external_interfaces`.)

    DBB
       Data Back Bone -- the path that NVDLA uses to perform DMA to system memory, such as the DRAM interface.

    DBBIF
       The NVDLA interface to the DBB.  (See :ref:`external_interfaces`.)

    IRQ
       Interrupt request.  (See :ref:`external_interfaces`.)

    NVDLA
       NVIDIA Deep Learning Accelerator.

    SRAMIF
       The NVDLA interface to an optional high-performance SRAM subsystem.

