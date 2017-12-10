
.. _test_application:

=================
Test applications
=================

Nvidia provides reference test applications for runtime and compiler verified on Linux.

.. _runtime_test_app:

------------------------
Runtime test application
------------------------

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

-------------------------
Compiler test application
-------------------------

Compiler test application also called as compiler tool gives option to convert caffe model to NVDLA format. It stores compiled network in loadable which can be loaded by runtime test application for execution.

Usage:
   ./nvdla_compiler [-options] --prototxt <prototxt_file> --caffemodel <caffemodel_file> -o <outputpath>

