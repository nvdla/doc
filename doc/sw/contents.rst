Software Manual
===============

NVDLA has a full software ecosystem including support from compiling network to inference. Part of this ecosystem includes the on-device software stack, a part of the :term:`NVDLA` open source release; additionally, NVIDIA will provide a full training infrastructure to build new models that incorporate Deep Learning, and to convert existing models to a form that is usable by :term:`NVDLA` software.  In general, the software associated with :term:`NVDLA` is grouped into two parts: the :ref:`compilation_tools` (model conversion), and the :ref:`runtime_environment` (run-time software to load and execute compiled neural network on :term:`NVDLA`). The general flow of this is as shown in the figure below;

.. _fig_software_package:
.. figure:: images/software_package.png
  :alt: Software Package
  :scale: 70%
  :align: center

.. toctree::
   :maxdepth: 2

   compilation_tool
   runtime_environment
   test_application

Browsing Source Code
====================

- top (https://github.com/nvdla/sw)
    - umd: :ref:`user_mode_driver`
        - core: NVDLA specific implementation of user mode components
            - **runtime:** :ref:`runtime_environment`
            - **compiler:** :ref:`compilation_tools`
            - **include:** Header file for common implementation
            - **common:** Implementation shared between runtime and compiler such as loadable and logging
        - **external:** External modules used in UMD such as flatbuffers
        - **include:** :ref:`umd_api`
        - **make:** Make files
        - port: :ref:`umd_layer`
            - **linux:** Portability layer for Linux
        - tests: Test applications
            - runtime :ref:`runtime_test_app`
            - compiler :ref:`compiler_test_app`
    - kmd: :ref:`kernel_mode_driver`
        - **Documentation:** Device tree bindings for NVDLA device
        - **firmware:** Core DLA hardware programming including HW layer scheduler
        - **include:** :ref:`kmd_interface`
        - **port:** :ref:`kmd_layer`
            - **linux:** Portability layer for Linux
    - **prebuilt: Prebuilt binaries**
    - regression: Test regression framework
        - flatbufs: Pre-generated loadables for sanity tests
        - golden: Golden results
        - scripts: Scripts used for test execution
        - testplan: Test plans
    - **scripts: General scripts**
