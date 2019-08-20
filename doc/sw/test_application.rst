
.. _test_application:

=================
Test applications
=================

Nvidia provides reference test applications for runtime and compiler verified on Linux.

.. _runtime_test_app:

--------------------------
Runtime sample application
--------------------------

Runtime test application provides options to run neural network loadable generated using NVDLA compiler. Along with network execution it also provides options for pre and post processing. This application can be launched in server mode too which allows sending test commands and files over network.

It demonstrates usage of IRuntime interface to execute a network on NVDLA.

Usage:
    For sanity tests which has input embedded in it.
    ./nvdla_runtime --loadable <loadable_file>

    For network tests which need image input
    ./nvdla_runtime --loadable <loadable_file> --image <image_file>

    Run in server mode
    ./nvdla_runtime -s

.. _compiler_test_app:

---------------------------
Compiler sample application
---------------------------

Compiler test application also called as compiler tool gives option to convert caffe model to NVDLA format. It stores compiled network in loadable which can be loaded by runtime test application for execution.

Usage:
   ./nvdla_compiler [-options] --prototxt <prototxt_file> --caffemodel <caffemodel_file> -o <outputpath>
Options:
   --profile <basic|default|performance|fast-math>     computation profile (default: fast-math)
   --cprecision <fp16|int8>                            compute precision (default: fp16)
   --configtarget <nv_full|nv_large|nv_small>          target platform (default: nv_full)
   --calibtable <int8 calib file>                      calibration table for INT8 networks (default: 0.00787)
   --quantizationMode <per-kernel|per-filter>          quantization mode for INT8 (default: per-kernel)
   --batch                                             batch size (default: 1)
   --informat <ncxhwx|nchw|nhwc>                       input data format (default: nhwc)
