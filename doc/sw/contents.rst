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

