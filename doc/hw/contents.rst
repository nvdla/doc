Hardware Manual
===============

.. toctree::
   :maxdepth: 2
   :hidden:

   v1/hwarch
   format
   v1/integration_guide
   v1/ias/precision
   v1/ias/lut-programming
   v1/ias/unit_description
   v1/ias/programming_guide
   v2/environment_setup_guide
   v2/integration_guide
   v2/scalability
   v2/verif_guide

* :doc:`v1/hwarch` -- a design-level view of the NVDLA hardware architecture,
  including detail on each sub-component, and register-level documentation.

NVDLA v1
--------
This is the non-configurable "full-precision" version of NVDLA. The design code is in nvdlav1 branch.

* :doc:`format` -- description of the in-memory format for weight and activation data.

* :doc:`v1/integration_guide` -- a guide for SoC integrators, including a
  walk-through of the NVDLA build infrastructure, NVDLA's testbenches, and
  synthesis scripts.

* :doc:`v1/ias/precision` -- the technologies used to keep the precision when formats INT8/INT16/FP16 are used.

* :doc:`v1/ias/lut-programming` -- the guide to programming lut tables

* :doc:`v1/ias/unit_description` -- description of internal architecture of sub-units

* :doc:`v1/ias/programming_guide` -- programming guide of sub-units

NVDLA v2
--------
This version is scalable design of NVDLA. Currently, the config nv_small is verified. The architecture of sub-units is same as NVDLA v1.

* :doc:`v2/environment_setup_guide` -- Please follow this document to setup tools and dependency libraries

* :doc:`v2/scalability` -- Scalability parameters and ConfigROM contents in nv_large and nv_small configurations

* :doc:`v2/integration_guide` -- a guide for SoC integrators, including NVDLA
  system inferce introduction, synthesis scripts.

* :doc:`v2/verif_guide` -- NVDLA Verification Suite User Guide
