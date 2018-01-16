================
Unit Description
================
(Notice: This version of *Unit Description* is only for nvdlav1 release.)

Bridge DMA
----------

.. overview-1:

Overview
~~~~~~~~

The input images and processed results are stored in external DRAM, but
external DRAM bandwidth and latency do not meet NVDLA’s requirements.
NVDLA needs to move data between external DRAM and internal SRAM.

Bridge DMA is proposed to full-fill this purpose. There are two
independent paths, one is copying data from external DRAM to internal
SRAM , and the other one is copying data from internal SRAM to external
DRAM. Bi-direction cannot work simultaneously. BDMA can also move data
from external DRAM to external DRAM, or from internal SRAM to internal
SRAM.

Bridge DMA has two DMA interfaces, one connects to external DRAM, and
the other connects to internal SRAM. Both two interfaces support read
and write requests. The data widths of both interfaces are 512 bits, and
the maximum burst length is 4.

In order to move all data in a cube, BDMA support line repeat which can
fetch multiple lines with address jumps between lines, reflect a
surface. And also, BDMA will support one more layer of repeat, that
fetch of multiple lines can be repeated, which reflect multiple
surfaces, reflect a cube.

.. _fig_image3_bdma_block:

.. figure:: ias_image3_bdma_block.png
  :align: center

  Bridge DMA

Convolution Pipeline
--------------------

.. overview-2:

Overview
~~~~~~~~

Convolution pipeline is one of pipelines of NVDLA core logic. It is used
to accelerate convolution algorithm. It supports comprehensive
programmable parameters for variable convolution size. Some features
like Winograd, multi-batch are applied in convolution pipeline to
improve the performance and reduce the efficiency loss caused by
limitation.

Convolution pipeline has five stages, which are convolution DMA,
convolution buffer, convolution sequence controller, convolution MAC and
convolution accumulator. They are also called CDMA, CBUF, CSC, CMAC and
CACC. Each stage has its own CSB slave port to receive configuration
from CPU. The synchronization mechanism is used by all stages.

Convolution pipeline supports three types of operations. They are:

-  Direct convolution for feature data, or DC mode

-  Convolution of image input, or image input mode

-  Winograd convolution, or Winograd mode

In convolution pipeline, there are 1024 MACs for int16/fp16. Convolution
pipeline reuses these resources as 2048 MACs for int8 precision. There
are same adders in convolution pipeline as well. Besides, there are
512KB SRAM in convolution buffer

Below is the diagram of convolution pipeline.

.. _fig_image4_convolution_pipeline:

.. figure:: ias_image4_convolution_pipeline.png
  :align: center

  convolution pipeline

Direct Convolution
~~~~~~~~~~~~~~~~~~

For convolution pipeline, it always has two parts of input. One is input
activation data, the other is weight data. Suppose NVDLA engine has such
input parameters:

-  Size of feature data cube: *W*\ x\ *H*\ x\ *C*

-  Size of one weight kernel: *R*\ x\ *S*\ x\ *C*

-  Total kernel number: *K*

-  Zero padding size: *LP* at left boundary, *RP* at right boundary,
   *TP* at top boundary, *BP* at bottom boundary.

-  Convolution stride: *SX* in X dimension, *SY* in Y dimension

-  Dilation: *DX* in X dimension, *DY* in Y dimension

-  Size of output data cube: *W’*\ x\ *H’*\ x\ *C’*

.. _fig_image5_convolution_operation:

.. figure:: ias_image5_convolution_operation.svg
  :align: center

  Convolution operation

Figure below shows the convolution stride and zero padding.

.. _fig_image6_convolution_stride_and_pad:

.. figure:: ias_image6_convolution_stride_and_pad.svg
  :align: center

  Convolution stride and zero padding

Then the equations of these parameters are:

.. math:: S^{'} = \left( S - 1 \right) \times DX + 1

.. math:: R^{'} = \left( R - 1 \right) \times DY + 1

.. math:: W^{'} = \frac{LP + W + RP - S'}{\text{SX}} + 1

.. math:: H^{'} = \frac{TP + H + BP - R'}{\text{SY}} + 1

.. math:: C^{'} = K

.. equation of convolution parameters

The relationship of each element *y* in output data cube, element *x* in
input feature data cube and element *wt* in weight kernels is:

.. math:: y_{w,\ h,\ k} = \ \sum_{r = 0}^{R - 1}{\sum_{s = 0}^{S - 1}{\sum_{c = 0}^{C - 1}{x_{(w*SX - LP + r),(h*SY - TP + s),\ c}*\text{wt}_{r,s,\ c,k}}}}

.. equation of convolution

The coordinate *w,h,c,k* in above equations are all start from zero.

To accomplish the convolution operation in the equation, convolution
pipeline uses a method called **direct convolution**. The key idea of
direct convolution is to split multiplications in one convolution kernel
into groups that each group contains 64 multiplications. The basic rules
are:

1. Distribute all MACs into 16 sub units. One sub unit is called MAC
   cell which has 64 MACs for int16/fp16 or 128 MACs for int8.

2. The assembly of MAC cells is called MAC cell array.

3. Divide all input data cube into 1x1x64 elements small cubes for
   int16, fp16 and int8.

4. Divide all weight data cubes into 1x1x64 elements small cubes for
   int16, fp16 and int8.

5. Multiply one small input data cube by one small weight data cube, and
   add products together. These multiplications and additions are
   operated in one MAC cell.

6. Combine these computing into 4 operation levels, which are atomic
   operation, stripe operation, block operation and channel operation.

Let’s take with int16 as example.

Atomic Operation
^^^^^^^^^^^^^^^^

Atomic operation is the basic step of direct convolution. In one atomic
operation, each MAC cell caches one 1x1x64 weight cubes from an
individual weight kernel. 16 MAC cells cache weights from 16 kernels
(int16/fp16) or 32 kernels (int8). One 1x1x64 atomic cube of feature
data shares by all MAC cells. Then MAC cells do computing mentioned in
rule 5. The output of each MAC cell is called **partial sum**. This
operation takes 1 cycle to finish the calculation, and gets 16 partial
sums. Partial sums are sent to convolution accumulator module for
further calculation.

The equation of partial sum is:

.. math:: \text{PS}_{w,\ h,k,r,s,\ c} = \ \sum_{i = c}^{min(c + 63,\ C - 1)}{x_{(w*SX - LP + r),(h*SY - TP + s),\ i}*\text{wt}_{r,\ s,\ i,k}}

..  equation of atomic operation

In the equation, *PS* refers to partial sum. Variable *c* is always
divisible by 64.

The diagram of atomic operation is as below.

.. _fig_image7_atomic_operation:

.. figure:: ias_image7_atomic_operation.svg
  :align: center

  Atomic operation

Stripe Operation
^^^^^^^^^^^^^^^^

A stripe operation combines a group of atomic operations from several
convolutions. During one stripe operation, the weight data in MAC cell
array keep unchanged. And input data cubes slides in input data cube.

Notice the partial sums in one stripe operation cannot be added
together.

The length of stripe operation has limitations. The lower limit 16 is
due to internal bandwidth to fetch weights for next stripe operation.
The upper limit 32 is due to buffer size in ACCU. But the length may be
less than lower limit in extreme cases.

The figure below shows an example of stripe operation which contains 16
atomic operations. The padding size is 0 in this case. Notice it’s not a
progressive scanning of input data cube!

.. _fig_image8_stripe_operation:

.. figure:: ias_image8_stripe_operation.svg
  :align: center

  Stripe operation

Block operation
^^^^^^^^^^^^^^^

Block operation is a higher level to stripe operations. During block
operation, each kernel in kernel group uses RxSx64 weight elements. And
small cube of input feature data shifts properly, to guarantee that the
results can add together across stripe operations.

.. _fig_image9_block_operation:

.. figure:: ias_image9_block_operation.svg
  :align: center

  Block operation

All stripe operations in one block operation have the same atomic
operation number. The partial sums from the same block operation are
added together per stripe operation in convolution accumulator. These
results are called accumulative sum

The equation of accumulative sum is:

.. math:: \text{AS}_{w,\ h,k,c} = \ \sum_{r = 0}^{R - 1}{\sum_{s = 0}^{S - 1}{\sum_{i = c}^{min(c + 63,\ C - 1)}{x_{(w*SX - LP + r),(h*SY - TP + s),\ i}*\text{wt}_{r,\ s,\ i,k}}}}

..  equation of block operation

In the equation, *AS* refers to accumulative sum. Variable *c* is always
divisible by 64.

Channel Operation
^^^^^^^^^^^^^^^^^

Channel operation is an even higher-level operation. It includes
(C+63)/64 block operations. The block operations in one channel
operation are similar, except the coordinate of channel direction is
different, showing as below

.. _fig_image10_channel_operation:

.. figure:: ias_image10_channel_operation.svg
  :align: center

  Channel operation

All partial sums of one channel operation can be added together by
stripe operation. After a channel operation, the result in convolution
accumulator is exactly the convolution result.

The equation is:

.. math:: y_{w,\ h,k} = \ \sum_{i = 0}^{\left\lfloor C/64 \right\rfloor - 1}{\sum_{r = 0}^{R - 1}{\sum_{s = 0}^{S - 1}{\sum_{j = c}^{min(c + 63,\ C - 1)}{x_{(w*SX - LP + r),(h*SY - TP + s),\ (i*64 + j)}*\text{wt}_{r,\ s,\ (i*64 + j),k}}}}}

..  equation of channel operation

This equation is identically equal to the original convolution equation.

Group Operation
^^^^^^^^^^^^^^^

Group operation is a higher-level operation than channel operation. It
includes about int((dataout_height \* dataout_width) / stripe_size)
channel operations. After a group operation, the output data compose a W
x H x K’ output surface. Here K’ refers to kernel size in a kernel
group.

Output Sequence
^^^^^^^^^^^^^^^

The sequence mentioned in each operation is mainly for input feature
data and weight data, but not the output sequence. The output data
sequence is quite simple. It follows the order of C’(K’)->W->H->C(K).
Here C’ or K’ refers to kernel group size, which is 16 for int16/fp16
and 32 for int8.

The output order of direct convolution is consistent with feature memory
mapping order.

.. _fig_image11_output_sequence:

.. figure:: ias_image11_output_sequence.svg
  :align: center

  Output sequence of a partition

Operation for Int8 and fp16
^^^^^^^^^^^^^^^^^^^^^^^^^^^

The operations mentioned above are almost about int16. For fp16, the
operations are the same. While for int8 precision, the parameters of
operations are various.

In convolution pipeline, each MAC for int16/fp16 is combined from two
MACs for int8, as well as adders. The element throughput of int8 is
doubled. Table below records parameters of one atomic operation.

.. table:: Precision parameters of atomic operation
 :name: tab_precision_atomic_operation

 +-------------+-------------+-------------+-------------+-------------+
 | convolution | input data  | weights per | kernels     | Output      |
 | precision   | elements    | kernel      |             | elements    |
 +=============+=============+=============+=============+=============+
 | int16       | 64          | 1024        | 16          | 16          |
 +-------------+-------------+-------------+-------------+-------------+
 | fp16        | 64          | 1024        | 16          | 16          |
 +-------------+-------------+-------------+-------------+-------------+
 | int8        | 64          | 2048        | 32          | 32          |
 +-------------+-------------+-------------+-------------+-------------+

Winograd Convolution
~~~~~~~~~~~~~~~~~~~~

Winograd convolution refers to an optional algorithm to optimize the
performance of direct convolution. Convolution pipeline only support
Winograd for 3x3xC kernel.

The key idea for Winograd convolution is to reduce the number of
multiplication, while increase adders to deal with additional
transformation. The equation of Winograd convolution used in convolution
pipeline is:

.. math:: S = \ A^{T}\left\lbrack \left( \text{Gg}G^{T} \right) \odot \left( C^{T}\text{dC} \right) \right\rbrack A

..  equation of Winograd convolution

Here symbol ⊙ indicates element-wise multiplication. Symbol *g* is a 3x3
kernel and *d* is a 4x4 tile of input data cube. Symbol *S* is the
convolution result of *g* and *d.* It’s a 2x2 matrix.

.. math::

   g = \begin{bmatrix}
   \text{wt}_{0,0} & \text{wt}_{0,1} & \text{wt}_{0,2} \\
   \text{wt}_{1,0} & \text{wt}_{1,1} & \text{wt}_{1,2} \\
   \text{wt}_{2,0} & \text{wt}_{2,1} & \text{wt}_{2,2} \\
   \end{bmatrix}

.. math::

   d = \begin{bmatrix}
   x_{0,0} & x_{0,1} & x_{0,2} & x_{0,3} \\
   x_{1,0} & x_{1,1} & x_{1,2} & x_{1,3} \\
   x_{2,0} & x_{2,1} & x_{2,2} & x_{2,3} \\
   x_{3,0} & x_{3,1} & x_{3,2} & x_{3,3} \\
   \end{bmatrix}

..  matrices of oprand

*A*, *G* and *C* are matrices to transform the weight and input feature
data.

.. math::

   C = \begin{bmatrix}
   1 & 0 & 0 & 0 \\
   0 & 1 & - 1 & 1 \\
    - 1 & 1 & 1 & 0 \\
   0 & 0 & 0 & - 1 \\
   \end{bmatrix}

.. math::

   G = \begin{bmatrix}
   1 & 0 & 0 \\
   0.5 & 0.5 & 0.5 \\
   0.5 & - 0.5 & 0.5 \\
   0 & 0 & 1 \\
   \end{bmatrix}

.. math::

   A^{T} = \begin{bmatrix}
   1 & 1 & 1 & 0 \\
   0 & 1 & - 1 & - 1 \\
   \end{bmatrix}

..  matrices of transform

Suppose *U = GgG\ :sup:`T`* and *V = C\ :sup:`T` dC*, then the equation
is:

.. math:: S = \ A^{T}\left\lbrack U \odot V \right\rbrack A

..  equation of Winograd convolution

According to the equation, the multiplication with *A*, *G* and *C* can
be implemented by adders. Only 16 multiplications are required to
calculate 4 results for a 3x3 kernel. While in direct convolution mode,
it needs 36 multiplications. So based on multiplications, the
performance of Winograd is 2.25 times of direct convolution.

Step *U = GgGT* is to convert 3x3 kernel to 4x4 kernel. Software should
convert weight kernel before NVDLA engine is running. Convolution
pipeline handles the conversion of input feature data and the result of
multiplications.

Unlike direct convolution, convolution pipeline divide kernels and input
feature data into 4x4x4 elements small data cubes. Before MAC cell,
extra adders are used to convert these cubes with matrix *C\ :sup:`T`*
and *C*. This step is called PRA.

In one atomic operation, 64 products in the same MAC cell are not added
together. The addition has three phases:

-  Phase 1, each 4 products in channel direction is added together. The
   output of phase 1 is 16 partial sums

-  Phase 2, each 4 partial sums is multiplied with matrix *A\ :sup:`T`*.
   The output of phase 2 is 8 partial sums

-  Phase 3, each 4 partial sums is multiplied with matrix *A*. The
   output is 4 partial sums.

Then 4 partial sums are stored in accumulator for further calculation.
Both phase 2 and phase 3 are called POA.

Winograd mode also has five operations. The comparing of parameters is
listed in table below.

.. table:: Parameters of operation modes
 :name: tab_cc_operation modes

 +-------------+-------------+-------------+-------------+-------------+
 | mode        | direct      | direct      | Winograd    | Winograd    |
 |             | convolution | convolution |             |             |
 +=============+=============+=============+=============+=============+
 | formats     | int16/fp16  | int8        | int16/fp16  | int8        |
 +-------------+-------------+-------------+-------------+-------------+
 | small data  | 1x1x64      | 1x1x64      | 4x4x4       | 4x4x4       |
 | cube per    |             |             |             |             |
 | MAC cell    |             |             |             |             |
 +-------------+-------------+-------------+-------------+-------------+
 | kernels per | 16          | 32          | 16          | 32          |
 | atomic      |             |             |             |             |
 | operation   |             |             |             |             |
 +-------------+-------------+-------------+-------------+-------------+
 | atomics     | 16~32       | 16~32       | 16~32       | 16~32       |
 | operation   |             |             |             |             |
 | per stripe  |             |             |             |             |
 | operation   |             |             |             |             |
 +-------------+-------------+-------------+-------------+-------------+
 | strips      | R*S         | R*S         | 1           | 1           |
 | operation   |             |             |             |             |
 | per block   |             |             |             |             |
 | operation   |             |             |             |             |
 +-------------+-------------+-------------+-------------+-------------+
 | blocks      | C/64        | C/64        | C/4         | C/4         |
 | operation   |             |             |             |             |
 | per channel |             |             |             |             |
 | operation   |             |             |             |             |
 +-------------+-------------+-------------+-------------+-------------+

The output sequence of Winograd convolution is similar to direct
convolution.

Some difference of Winograd:

-  For Winograd operation, the output width and height shall be
   divisible by 4. It’s a mandatory requirement. It’s for special scan
   order.

-  The scan order of stripe operation in Winograd convolution is
   different from direct convolution. Please see figure below.

-  The block operation always has only one stripe operation.

-  Winograd layer always outputs 4 lines in parallel. SDP will guarantee
   the correction of memory mapping of output data cube.

.. _fig_image12_scan_order_wino:

.. figure:: ias_image12_scan_order_wino.svg
  :align: center

  Scan order of stripe operation in Winograd (W-H projection)

Deconvolution
~~~~~~~~~~~~~

Deconvolution is a type of special convolution. It is some kind of
inverse operation of normal convolution. Unlike normal convolution case,
deconvolution layer always enlarges the data cube after calculation.

In NVDLA architecture deconvolution is a SW feature. From HW
perspective, a SW deconvolution layer consists of a serial convolution
layer and a contract layer supported by RUBIK unit.

:numref:`fig_image13_1d_deconvolution` is an example of one-dimensional deconvolution layer. Input
data cube is W x 1 x 1 and kernel size is 3 x 1 x 1. Though the
calculation flow is different from convolution, the result formula is:

.. math:: DAOUT_{i} = \sum_{j = 0}^{2}{DAIN_{i + j - 2}*W_{2 - j}}

.. Formula for one-dimension deconvolution

The formula is very similar to convolution formula, except weight R/S
order is reversed. More generally, the formula of a WxHxC input data
cube with K SxRxC kernels is:

.. math:: DAOUT_{(w,\ h,\ k)} = \sum_{x = 0}^{S - 1}{\sum_{y = 0}^{R - 1}{\sum_{z = 0}^{C - 1}{DAIN_{(w + x + 1 - S,h + y + 1 - R,\ z)}*W_{(S - 1 - x,R - 1 - y,z,k)}}}}

..  Formula for 3D deconvolution

According to equation, the 3D deconvolution is equal to a convolution
with (S-1) and (R-1) zero padding and reversed R/S weight order

.. _fig_image13_1d_deconvolution:

.. figure:: ias_image13_1d_deconvolution.svg
  :align: center

  One-dimensional deconvolution, x stride = 1

If deconvolution X stride or Y stride is not 1, the calculation flow is
a bit different. We use set split to generate smaller kernel sets. Each
set of kernel acts like a convolution layer, which X and Y strides are
equal to 1. That is how to use several convolution layers to calculate
the result of a deconvolution layer.

After a serial convolution layer, all deconvolution result values are
ready but the mapping order is not expected result. If we append the
convolutional output cube one after another in C direction, then the
total output data cube is the Winograd channel-extended data cube. The
extension parameter is deconv_x_stride and deconv_y_stride.

So, NVDLA uses a special layer contract layer (performed by Rubik) to reorder these output
values to get the ideal deconvolutional output cube.

In conclusion, NVDLA supports deconvolution layer by below strategy:

-  NVDLA use two steps to perform a deconvolution layer which stride is
   bigger than 1

-  The first step is a serial convolution layers with order-reversed
   kernels.

-  The output of first step forms a Winograd channel-extended output
   data cube. Extension parameter is deconvolution x stride and
   deconvolution y stride.

-  The second step is running on RUBIK units.

-  Rubik unit does an inverse operation to Winograd channel-extended
   data cube.

-  After second HW-layer, output data cube is expected result.

Convolution with Image Input
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

NVDLA support convolution with image data. Here image data could be a
part or whole image surface. However, NVDLA can only support it for
direct convolution. **DC**, **Winograd and deconvolution layer cannot
use pixel formats**. Besides, multi-batch option is not supported for
image input.

Comparing to DC, image input case has some difference:

-  Channel pre-extension. The weight kernel should do channel
   pre-extension. It is unlike DC mode or Winograd mode.

-  Data mapping in convolution buffer. The image data mapping in
   convolution buffer is very different. All element of left/right
   padding and input pixel line are compactly residing in CBUF entries.
   See figure below. If channel size is 4, the element mapping order is
   R(Y)->G(U)->B(V)->A(X). If channel size is 3, the order is
   R(Y)->G(U)->B(V).

-  Distribution of stripe operation. The stripe operation length is
   fixed to 64. And stripe operation shall never across lines. So that
   every stripe operation is started from first byte of CBUF entry.

-  Use channel post-extension for speedup. Even with channel
   pre-extension, usually kernel channel size is less than 32.
   Therefore, channel post-extension is very useful for image input
   convolution layer.

.. _fig_image14_pixel_mapping_in_cbuf:

.. figure:: ias_image14_pixel_mapping_in_cbuf.svg
  :align: center

  Pixel mapping in convolution buffer

Channel Post-Extension
~~~~~~~~~~~~~~~~~~~~~~

Channel post-extension is an option of saving MAC performance for
convolution with image input case.

In convolution pipeline, one atomic operation requires 64 elements in
channel dimension (exclude Winograd case). If the channel size of input
data cube is less than 64, MACs are not 100% activation in each cycle.
Thus, MAC efficiency depends on channel size in DC mode and image input
mode.

The basic idea of channel post-extension is doing a vertical extension
to enlarge the channel size during runtime.

For example, an image input layer has 4x4x4 kernel size. If
post-extension is not enabled, the pre-extended channel size is 16 and
efficiency of MACs drops to 25%. However, if post-extension parameter is
set to 4, every atomic cycle convolution pipeline will fetch 4 neighbor
lines and combine them as a C=64 line. Then MAC efficiency rise back to
100%.

Some limitation of channel post-extension:

-  Channel post-extension is only for image input convolution.

-  Channel post-extension support 2-line extending and 4-line extending.

-  Channel post-extension is limited by pre-extended channel size and
   convolution x stride

.. table:: Limits of channel post-extension
 :name: tab_limits_of_channel_post_extension

 +----------------------+----------------------+----------------------+
 | Channel              | conv_x_stride limit  | pre-extended channel |
 | post-extension       |                      | size limit           |
 +======================+======================+======================+
 | 1-line               | No                   | No                   |
 +----------------------+----------------------+----------------------+
 | 2-lines              | (conv_x_stride \*    | <=32                 |
 |                      | ori_channel_size)    |                      |
 |                      | <=32                 |                      |
 +----------------------+----------------------+----------------------+
 | 4-lines              | (conv_x_stride \*    | <=16                 |
 |                      | ori_channel_size)    |                      |
 |                      | <=16                 |                      |
 +----------------------+----------------------+----------------------+

It’s necessary to mention that the channel pos-extension number (N)
doesn’t need to be less than kernel height (R), hardware can
automatically tailor the redundant lines to avoid them be involved to
computation. However, this also means the user shouldn’t expect N times
of MAC efficiency improvements for this case.

Multi-Batch Mode
~~~~~~~~~~~~~~~~

NVDLA engine also supports multi-batch to enhance the performance and
reduce the bandwidth, especially for FC (fully-connected) layers. The
output of one FC layer is a 1x1xC data cube. That means all weights in
one FC layer are used only once. One stripe operation in FC layer has
only one atomic operation. But convolution pipeline needs 16 cycles to
load weight for next atomic operation. That makes a lot of bubble in
pipeline and MAC efficiency falls to 6.25%. To save the efficiency,
NVDLA engine applies multi-batch mode.

The multi-batch is a special option for DC mode with multiple input
feature data cube. Convolution pipeline will fetch multiple input data
cubes with only one set of weight kernels. It also changes the atomic
operation. Small cubes from different input data cubes are loaded
interlaced for atomic operation one after another. The stripe operation
contains atomic operations for multiple batches. By that part of weight
loading cycles are hidden and the efficiency rises.

The length of stripe operation with different batch size are:

.. table:: Stripe length of different batch size
 :name: tab_stripe_length_multi_batch_mode

 +---------------+------+------+------+-------+------+------+
 | Batch Size    | 1    | 2    | 3    | 4     | 5    | 6    |
 +===============+======+======+======+=======+======+======+
 | Normal length | 16   | 8x2  | 8x3  | 4x4   | 4x5  | 4x6  |
 +---------------+------+------+------+-------+------+------+
 | Max length    | 32   | 16x2 | 16x3 | 8x4   | 8x5  | 8x6  |
 +---------------+------+------+------+-------+------+------+
 | Batch Size    | 7    | 8    | 9    | 10    | 11   | 12   |
 +---------------+------+------+------+-------+------+------+
 | Normal length | 4x7  | 2x8  | 2x9  | 2x10  | 2x11 | 2x12 |
 +---------------+------+------+------+-------+------+------+
 | Max length    | 8x7  | 4x8  | 4x9  | 4x10  | 4x11 | 4x12 |
 +---------------+------+------+------+-------+------+------+
 | Batch Size    | 13   | 14   | 15   | 16~32 |      |      |
 +---------------+------+------+------+-------+------+------+
 | Normal length | 2x13 | 2x14 | 2x15 | 1xN   |      |      |
 +---------------+------+------+------+-------+------+------+
 | Max length    | 4x13 | 4x14 | 4x15 | 1xN   |      |      |
 +---------------+------+------+------+-------+------+------+

.. _fig_image15_multi_batch:

.. figure:: ias_image15_multi_batch.svg
  :align: center

  Multi-batch mode

Dilation
~~~~~~~~

Dilation is an option that enlarges the kernel in R and S dimensions
with zero values. This function is enabled by SW according to algorithm.

Diagram below shows a case that dilation parameter = 3.

.. _fig_image16_weight_dilation:

.. figure:: ias_image16_weight_dilation.svg
  :align: center

  Weight dilation

NVDLA support dilation in both R and S dimensions.

Limits of dilation:

-  Dilation is available for DC mode only.

-  Dilation is not available for Winograd or image input mode.

Power Consideration
~~~~~~~~~~~~~~~~~~~

Convolution pipeline supports SLCG for each pipeline stage. If the
pipeline stage is idle and no valid HW-layer is available, the data path
of pipeline stage will be clock gated. But the regfile sub models inside
pipeline stage cannot be gated by SLCG.

Convolution DMA
---------------

.. overview-3:

Overview
~~~~~~~~

Convolution DMA (CDMA) is a pipeline stage of convolution pipeline. It
fetches data from SRAM/DRAM for convolution operation and stores them
into convolution buffer with particular order. The supported input
formats are:

-  Pixel data

-  feature data

-  Uncompressed/compressed weight

-  WMB

-  WGS

Two read channels connect from convolution DMA to AXI interface. They
are weight read channel and data read channel. To fetch input format
listed above, the channels are shared by one or more formats. The table
below records the sharing in all use cases.

.. table:: Channel sharing in CDMA
 :name: tab_channel_sharing_in_cdma

 +-------------+-------------+-------------+-------------+-------------+
 | Input       | Image Case  | Uncompresse | Uncompresse | Compressed  |
 | Format      |             | d           | d           | Weight Case |
 |             |             | Feature     | Weight Case |             |
 |             |             | Case        |             |             |
 +=============+=============+=============+=============+=============+
 | Pixel data  | data        | NA          | NA          | NA          |
 |             | channel     |             |             |             |
 +-------------+-------------+-------------+-------------+-------------+
 | Uncompresse | NA          | data        | NA          | NA          |
 | d           |             | channel     |             |             |
 | feature     |             |             |             |             |
 | data        |             |             |             |             |
 +-------------+-------------+-------------+-------------+-------------+
 | Uncompresse | NA          | NA          | weight      | NA          |
 | d           |             |             | channel     |             |
 | weight      |             |             |             |             |
 +-------------+-------------+-------------+-------------+-------------+
 | Sparse      | NA          | NA          | NA          | weight      |
 | compressed  |             |             |             | channel     |
 | weight      |             |             |             |             |
 +-------------+-------------+-------------+-------------+-------------+
 | WMB         | NA          | NA          | NA          | weight      |
 |             |             |             |             | channel     |
 +-------------+-------------+-------------+-------------+-------------+
 | WGS         | NA          | NA          | NA          | weight      |
 |             |             |             |             | channel     |
 +-------------+-------------+-------------+-------------+-------------+

Convolution DMA sends read requests only. All requests sent by
convolution DMA are 64-byte aligned.

.. _fig_image17_cdma:

.. figure:: ias_image17_cdma.png
  :align: center

  Convolution DMA

CDMA uses three sub modules, which are CDMA_DC, CDMA_WG and CDMA_IMG, to
fetch pixel data or feature data for convolution. The procedures of
these sub modules are similar. At any time, only one of the sub modules
is activated to fetch pixel/feature data.

Take CDMA_DC as an example to introduce the procedures:

-  Check status of convolution buffer for enough free space.

-  Generate reading transactions

-  Cache feature data in shared buffers

-  Reshape feature cubes into proper order

-  Generate convolution buffer write address

-  Write feature data into convolution buffer

-  Update status of convolution buffer in CDMA_STATUS module

Convolution DMA module use a dedicate sub module to handle the
requirement of Winograd. CDMA_WG has very similar structure and
functionality to CDMA_DC. But the feature data mapping in convolution
buffer is different. Thus CDMA_WG has a special fetching sequence.
Besides, CDMA_WG always do Winograd channel extension.

The module CDMA_IMG fetches pixel data from external memory. It
generates the address according to data format, reorders the pixel
elements and writes them into the proper entry of the convolution
buffer. The basic behavior of CDMA_IMG is like CDMA_DC, but tt only
services for pixel data.

Only CDMA_DC module supports multi-batch mode. That is fetching more
than one input feature data cube in one HW-layer to improve the
performance. The max batch size is up to 32.

CDMA also use a dedicate sub module to weight fetching. The sub module
is called CDMA_WT. CDMA_WT is very simple to other sub modules, except
that it can support three read steams at a time. If the input weight
format is uncompressed, it only fetches weight data. If the input weight
format is compressed, weight/WMB/WGS are all fetched. Please see `Data Formats
<http://nvdla.org/hw/format.html>`_ for more details of weight formats.

If the input weight data is compressed, two arbiters are enabled for
order of read streams. First a weighted round-robin arbiter grants a
request from weight stream and WMB stream. Then the winner competes with
WGS request steam by a static priority arbiter. WGS always has priority.
At last, the final winner is sent to weight channel for data fetching.

CDMA_WT is always trying to fill convolution buffer as much as possible,
until the free entries runs out or weight fetching is done.

CDMA maintains and communicates status of both weight buffer and input
data buffer in CBUF. There are two copies of status in CDMA and CSC. Two
modules exchange the update/release information like an asynchronous
FIFO, and decide when to fetch new feature/pixel/weight data and when to
release these data.

.. power-consideration-1:

Power Consideration
~~~~~~~~~~~~~~~~~~~

Convolution DMA applies SLCG in data path. The clock of data path of
convolution DMA is gated by SLCG when it is idle and no HW-layer is
available from programmable registers. While SLCG does not clock gate
the regfile sub module inside convolution DMA.

Convolution Buffer
------------------

.. overview-4:

Overview
~~~~~~~~

Convolution buffer (CBUF) is one stage of convolution pipeline. It has
totally 512KB SRAMs. The SRAMs cache input pixel data, input feature
data, weight data and WMB data from CDMA module, and are read by
convolution sequence generator module. CBUF has two write ports and
three read ports.

CBUF contains of 16 32KB banks. Each bank consists of two 512-bit-wise,
256-entry two-port SRAMs. These banks are acts as three logical circular
buffers. They are:

-  Input data buffer

-  Weight buffer

-  WMB buffer

If the weight format is compressed, bank15 is assigned for WMB buffer,
while two other buffers can use bank0~bank14. If weight format is
uncompressed, WMB buffer is not assigned with any bank. In this case
data buffer and weight buffer can fully use all 16 banks. If total
required banks are less than 16, remaining banks are used.

Each buffers act like circular buffers. New input data/weight/WMB has
incremental entry address. If the address reaches the max, it turns to
zero and increase again.

.. _fig_image18_cbuf:

.. figure:: ias_image18_cbuf.png
  :align: center

  Convolution buffer

.. power-consideration-2:

Power Consideration
~~~~~~~~~~~~~~~~~~~

Convolution buffer applies SLCG for registers in data path beyond SRAMs.
The clock of data path of convolution buffer is gated by SLCG when it is
idle and no HW-layer is available from programmable registers. While
SLCG does not clock gate the regfile sub module inside convolution
buffer.

Convolution Sequence Controller
-------------------------------

.. overview-5:

Overview
~~~~~~~~

Convolution sequence controller (CSC) is response to load input
feature/pixel data and weight data from CBUF and sends them to
convolution MAC in proper order. It’s the key module to perform
computing sequences of convolution which is mentioned in `Convolution Pipeline`_.

Convolution sequence controller (CSC) includes three sub modules, which
are CSC_SG, CSC_WL and CSC_DL. See :numref:`fig_image19_csc`.

CSC_SG is short of convolution sequence generator. This module generates
the whole sequence.

The working flow of CSC_SG is as below:

1. Poll for enough data and weights in CBUF

2. Generate a pair of sequence package, including weight loading package
   and data loading package. Each package represents for one stripe
   operation.

3. Push packages into two FIFO

4. Two counters for weight and feature/pixel are both down counting

5. When counters reach zero, check signals from convolution accumulator
   for any back pressure

6. If all conditions are ready, send weight and data packages in proper
   time to CSC_WL and CSC_DL.

.. _fig_image19_csc:

.. figure:: ias_image19_csc.png
  :align: center

  Convolution sequence controller

CSC_DL is short of convolution data loader. This module is the logic to
execute feature/pixel loading sequence. It receives packages from
sequence generator, loads feature/pixel from CBUF and sends them to
convolution MAC. Besides it maintains status of data buffer and
communicates with CDMA to keep the status up to date. For winograd mode,
it also does PRA (pre-addition) to transform input feature data.

CSC_WL is short of convolution weight loader. This module is the logic
to execute weight loading sequence. It receives packages from sequence
generator, loads weights from CBUF, does necessary decompression and
sends them to convolution MAC. Besides it maintains status of weight
buffer and communicates with CDMA_WT to keep the status up to date

.. power-consideration-3:

Power Consideration
~~~~~~~~~~~~~~~~~~~

Convolution sequence controller applies SLCG for registers in data path.
The clock of data path of convolution sequence controller is gated by
SLCG when it is idle and no HW-layer is available from programmable
registers. While SLCG does not clock gate the regfile sub module inside
convolution sequence controller.

Convolution MAC
---------------

.. overview-6:

Overview
~~~~~~~~

Convolution MAC (CMAC) module is one stage of convolution pipeline for
convolution operation. It receives input data and weight from
convolution sequence controller(CSC), does multiplication and addition
and output the result to convolution accumulator. When working in
Winograd mode convolution MAC takes response to do POA (post addition).

CMAC has 16 same sub modules called MAC cell. Each MAC cell contains 64
16-bit multipliers for int16/fp16. Besides it contains 72 adders for
int16/fp16 which are for POA. Each multiplier and adder can split into
two calculation unit for int8 format. The throughput of int8 is 2 times
of int16 in any mode. The output result is called partial sum. The
pipeline depth is 7 cycles.

One bypassed pipeline in Convolution MAC is used to deliver status. The
status includes start and end flag of operations. It takes status 4
cycles to go through pipeline, which is 3 cycles ahead of partial sum to
prefetch assembly buffer in CACC.

.. _fig_image20_cmac:

.. figure:: ias_image20_cmac.png
  :align: center

  Convolution MAC

For physical design limit, CMAC is divided into two parts, CMAC_A and
CMAC_B. Each part has individual CSB interface and regfile. But they are
considered as one pipeline stage in usage.

.. power-consideration-4:

Power Consideration
~~~~~~~~~~~~~~~~~~~

Convolution MAC (CACC) applies SLCG in data path. The clock of data path
of convolution MAC is gated by SLCG when it is idle and no HW-layer is
available from programmable registers. While SLCG does not clock gate
the regfile sub module inside convolution MAC.

Besides, convolution MAC can clock gate the MAC cells individually. When
the number of kernels is not enough to fill all the MAC cells, the empty
ones will be automatically clock gated.

Convolution Accumulator
-----------------------

.. overview-7:

Overview
~~~~~~~~

Convolution accumulator (CACC) is one stage of convolution pipeline
after CMAC. It is used to accumulate partial sums from convolution MAC,
and truncate the result before sending to SDP. Besides, the big buffer
in convolution accumulator can smooth the peak throughput of convolution
pipeline.

The components in CACC include assembly SRAM group, delivery SRAM group,
adder array, truncating array, valid-credit controller and a checker.

Here is the CACC working flow:

1. Prefetch accumulative sums from assembly SRAM group.

2. When partial sums arrive, send them to adder array along with
   accumulative sums. If the partial sums are from the first stripe
   operation, the accumulative sums should be 0.

3. Gather new accumulative sums from output side of adder array.

4. Stores into assembly SRAM group

5. Repeat step1~ step3 in terms of stripe operation until a channel
   operation is done.

6. If a channel operation is done, the output of adders is truncated.

7. Gather truncated results and store them into delivery SRAM group.

8. Load truncated results from delivery buffer group and send them to
   SDP

.. _fig_image21_cacc:

.. figure:: ias_image21_cacc.png
  :align: center

  Convolution accumulator

The assembly SRAM group contains 4 96Bx32 SRAMs and 4 64Bx32 SRAMs. The
buffer group is used to cache accumulative sums with high precision. For
direct convolution, assembly SRAM group acts as one 96Bx128 buffers for
int16/fp16 or one 136Bx128 buffer for int8. For Winograd convolution,
assembly SRAM acts as one 384Bx32 buffer for int16/fp16 or one 544Bx32
buffer for int8. It takes at least 11 cycles to do a read-store circle
for assembly group.

The delivery SRAM group contains 8 64Bx32 SRAMs. The buffer group is
used to cache truncated result to be delivered to SDP. The input varies
from 16 elements to 128 elements per cycle, while the output is always
16 elements per cycle.

The precision of accumulative sum is as below.

.. table:: CACC precision
 :name: tab_cacc_precision

 +----------------------+----------------------+----------------------+
 | Input Format         | Accumulative Sum     | Truncated Result     |
 +======================+======================+======================+
 | INT8                 | INT34                | INT32                |
 +----------------------+----------------------+----------------------+
 | INT16                | INT48                | INT32                |
 +----------------------+----------------------+----------------------+
 | FP16                 | FP44 (8b exponent,   | FP32 (IEEE754        |
 |                      | 38b signed decimal)  | standard)            |
 +----------------------+----------------------+----------------------+

In adder array, there are 64 INT48 adders, 64 INT34 adders and 64 FP48
adders. Part of them are activated in different mode

.. table:: Activated adders for different precision and mode
 :name: tab_adder_cacc

 +-----------------+-----------------+-----------------+-----------------+
 | Input Format    | Activated INT48 | Activated INT34 | Activated FP44  |
 | and Mode        | Adders          | Adders          | Adders          |
 +=================+=================+=================+=================+
 | INT8 DC/Image   | Adder 0~15      | Adder 0~15      | NA              |
 +-----------------+-----------------+-----------------+-----------------+
 | INT8 Winograd   | Adder 0~63      | Adder 0~63      | NA              |
 +-----------------+-----------------+-----------------+-----------------+
 | INT16 DC/Image  | Adder 0~15      | NA              | NA              |
 +-----------------+-----------------+-----------------+-----------------+
 | INT16 Winograd  | Adder 0~63      | NA              | NA              |
 +-----------------+-----------------+-----------------+-----------------+
 | FP16 DC/Image   | NA              | NA              | Adder 0~15      |
 +-----------------+-----------------+-----------------+-----------------+
 | FP16 Winograd   | NA              | NA              | Adder 0~63      |
 +-----------------+-----------------+-----------------+-----------------+

To support multi-batch option in DC mode, CACC applies data remapping
function in delivery SRAM group. That means when multi-batch is enabled,
the data ordering in delivery SRAM group may not match the sequence from
assembly SRAM group. Write controller of delivery SRAM will combine
atomic cubes if they will be in same 64 bytes package after further
calculation in SDP. This function allows SDP to send 64B aligned write
requests as many as possible when multi-batch is enabled. Below diagram
shows a case with batch size of 2.

.. _fig_image22_data_remapping_in_cacc:

.. figure:: ias_image22_data_remapping_in_cacc.svg
  :align: center

  Data remapping in CACC

The protocol between CMAC and CACC is valid-only protocol. In case of
overflow, CACC uses valid-credit protocol to back pressure CSC.

.. power-consideration-5:

Power Consideration
~~~~~~~~~~~~~~~~~~~

Convolution accumulator applies SLCG in data path. The clock of data
path of convolution accumulator is gated by SLCG when it is idle and no
HW-layer is available from programmable registers. While SLCG does not
clock gate the regfile sub module inside convolution accumulator.

Single Point Data Processor
---------------------------

.. overview-8:

Overview
~~~~~~~~

Single point data processor responses for performing post processing
operations on element level. In NVDLA version 1.0, point processing is
designed to accomplish following operations.

Bias Addition
~~~~~~~~~~~~~

For convolution layer, there’re always bias addition after convolution.
In NVDLA, we implement bias addition in SDP.

The mathematic formula for bias addition is:

..
  image23, image24, image25

.. math:: y = x + bias

x is the input data can either come from Convolution Pipeline or SDP
M-RDMA;

bias is the pre-trained parameter which can be one of 3 options:

a) Register: If bias is unique for entire data cube;

b) SDP B/N/E-RDMA per-channel mode: If bias is shared for all elements
   in the same channel;

c) SDP B/N/E-RDMA per-element mode: If bias is different
   element-by-element;

Non-Linear Function
~~~~~~~~~~~~~~~~~~~

Non-Linear function is to accomplish activation layer operation.

Based on current network analysis, there are three activation functions
are used:

-  ReLU, for an input :math:`x`, the output is :math:`max(x,0)`.

-  Sigmoid, for an input :math:`x`, the output is
   :math:`\frac{1}{1 + e^{- x}}`.

.. _fig_image26_sigmoid:

.. figure:: ias_image26_sigmoid.png
  :align: center

  Sigmoid Function

-  Hyperbolic tangent, for an input :math:`x`, the output is
   :math:`\frac{1 - e^{- 2x}}{1 + e^{- 2x}}`.

.. _fig_image27_hyperbolic:

.. figure:: ias_image27_hyperbolic.png
  :align: center

  Hyperbolic Function

In case of ReLU activation function, it could be implemented in hardware
logic. In cases of Sigmoid and hyperbolic tangent functions, the
non-linear functions, so they are expected to be implemented in look-up
table manner (see Section "LUT programming" of Programming Guide document for detail).

Batch Normalization
~~~~~~~~~~~~~~~~~~~

Batch normalization is widely used layer. It can be descripted by
formula below:

.. math:: x^{'} = \frac{x - \mu}{\theta}

Where, :math:`\mu` is the mean and :math:`\theta` is the standard variance and x is element of
feature data cubes.

SDP support batch normalization with given mean/standard variance
parameters. The parameters are obtained from training.

SDP can support per layer parameter or per channel parameter to do batch
normalization operation. When the parameter is per channel, they are
interleaved in memory (see `Data Formats <http://nvdla.org/hw/format.html>`_). 
And a DMA in SDP will fetch the
parameter and calculate the feature data cube from convolution pipeline.

Element-Wise Layer
~~~~~~~~~~~~~~~~~~

Element-wise layer refers to a type of operation between two feature
data cube which have the same W, H and C size. These two W x H x C
feature data cubes do element-wise addition, multiplication or max/min
comparison operation and output one W x H x C feature data cube.

.. _fig_image31_element_wise:

.. figure:: ias_image31_element_wise.svg
  :align: center

  Element-wise operation

SDP unit can support element-wise layer for all 3 types of data
precisions. Every element-wise layer on SDP is configured to do addition
or multiplication.

SDP support both online mode and offline mode for element-wise layer.
When online mode, one data cube comes from convolution pipeline, and the
other input data cube is fetched from memory. When offline mode, SDP
fetches both input data cubes from memory.

PReLU
~~~~~

Different from ReLU which clip negative values to 0, PReLU acts as:

.. _fig_image32_prelu:

.. figure:: ias_image32_prelu.svg
  :width: 20%
  :align: center

  PReLU

The scaling factor k can be either per cube constant or per-channel
variant.

SDP support it by update the multiplier behavior: If PReLU mode is
selected, multiplier will bypass the positive value and apply scaling on
negative values only. PReLU mode is supported by multiplier in all the 3
sub-modules.

Be noticed that:

1. BatchNorm and PReLU feature are exclusive for a specific sub-unit,
this is due to only one multiplier is available for a subunit;

2. If PReLU is enabled for one sub-unit, the ALU in that unit MUST be
bypassed. This is due to there’s only one truncate for a sub-unit and
negative/positive requires different truncate here.

Format conversion
~~~~~~~~~~~~~~~~~

NVDLA supports INT8/INT16/FP16 precisions, lower precision delivers high
performance while higher precision gives better inference results.

It’s possible that software need to different precision on different
hardware layers thus precision conversion is necessary.

SDP is responsible for precision conversion, the supported conversions
in one hardware layer are listed in :numref:`tab_precision_conversion_sdp` precision conversion for
SDP layer (offline). If SDP plays with CC, the supported format
conversion is listed in :numref:`tab_precision_conversion_conv`.

The precision conversion and normal SDP function are irrelevant, which
means SDP is able to do conversion and operation (e.g.: Bias addition,
BatchNorm, EW, etc) at the same time.

Comparison
~~~~~~~~~~

Comparison mode in SDP_Y takes 2 inputs then compare them. If any element in those input data
cube mismatch, it will be updated to status register after HWL done.

In order to save bandwidth, there won’t be any output write to external
memory for comparison mode.

Function Description
~~~~~~~~~~~~~~~~~~~~

Following diagram shows internal blocks of point processing sub-unit and
connections to other sub-units.

.. _fig_image33_sdp:

.. figure:: ias_image33_sdp.png
  :align: center

  Single point Data Processing block diagram

Function Blocks:

There are several function blocks, each of which aims to different main
purpose:

-  Block M is to select input data from MEM or Conv Core, which can be
   set from register

-  Block X1/X2 has the same architecture which supports: Bias addition,
   BatchNorm, PReLU, ReLU, Eltwise.

-  Block Y is primarily designed for element-wise, but it’s also able to
   support Bias addition, PReLU. An extra LUT operation can be appended
   before output, which is for any non-linear operation

-  Block C1/C2 is for additional scaling and offset to save bits but
   still try to keep accuracy

-  A Demux on the very end to send the output data to either WDMA for
   wring back to memory, or to PDP for pooling operation without writing
   back.

Most of function unit have bypass mode by configuration, so SW can
choose full function or partial to match all the operations needed in
one hardware layer.

The throughput for each sub-unit is:

+----------+-------------------+
| Sub-unit | Throughput        |
+==========+===================+
| X1/X2    | 16 elements/cycle |
+----------+-------------------+
| Y        | 4 elements/cycle  |
+----------+-------------------+

1. Working Mode: Flying:

   a. On-flying: source data is from Conv-Core

   b. Off-flying: source data is from Memory, which is read by M-RDMA

2. Bias Addition:

   a. operand data can be per element, per channel or per cube, the
      actual operation can be happened at any of X1/X2/Y based on
      software configuration

      i.  Bias data will be fetched from MEM if per element/channel, if
          truncate is enabled, all elements shares a same truncate value

      ii. Bias data will be set by register if per cube

   b. Multiplier will be bypassed

3. Batch Normalization

   a. operand data can be per element, per channel or per cube, the
      actual operation can be happened at X1/X2/Y based on software
      configuration

      i.  operand data will be fetched from MEM if per element/channel,
          if truncate is enabled, all elements shares a same truncate
          value

      ii. operand data will be set by register if per cube

   b. operand data for addition and for multiplier should be packed
      together and in same format of per element, per channel or per
      cube, see `Data Formats <http://nvdla.org/hw/format.html>`_ for detail.

   c. ReLU can be bypassed or operated

4. Element-Wise

   a. operand data can be per element, per channel or per cube

      i.  operand data will be fetched from MEM if per element/channel,
          if truncate is enabled, all elements shares a same truncate
          value

      ii. operand data will be set by register if per cube

   b. operand data should be either for max/min/sum or for multiplier

   c. LUT can be bypassed or operated

5. PReLU:

   a. Operand data can be per channel or per cube

      i.  Operand data will be fetched from MEM if per channel, if
          truncate is enabled, all elements shares a same truncate value

      ii. Operand data will be set by register if per cube;

   b. PReLU mode bit should be set as true for multiplier (after set
      this bit, hardware will bypass positive input samples, the scaling
      applied on negative input only)

   c. LUT can be bypassed or operated

6. Hybrid Mode (SW feature)

   Bias addition/BatchNorm operations are linear operation this means
   software can fuse those operation into one sub-module to optimize
   power/perf. Take BiasAddition + BatchNorm for example, if they’re
   working on separated submodule, the formula is: :math:`x^{'} = x + bias`,
   :math:`y = \frac{x^{'} - \mu}{\theta}`.

   If we fuse those 2 formulas as one: :math:`y = \frac{x + bias - \mu}{\theta} = \frac{x - (\mu - bias)}{\theta}`.

   As :math:`\mu, \theta, bias` are pre-trained parameters, software can fuse them into one cube
   thus it’s doable;

As a summary, the features supported by each sub-unit are listed in
table below:

.. table:: SDP supported use scenarios
 :name: tab_sdp_supported_use_scenarios

 +-----------------------+----+----+---+
 | Feature\Module        | X1 | X2 | Y |
 +=======================+====+====+===+
 | Bias addition         | Y  | Y  | Y |
 +-----------------------+----+----+---+
 | BatchNorm             | Y  | Y  | Y |
 +-----------------------+----+----+---+
 | Eltwise               | Y  | Y  | Y |
 +-----------------------+----+----+---+
 | PReLU                 | Y  | Y  | Y |
 +-----------------------+----+----+---+
 | ReLU                  | Y  | Y  | Y |
 +-----------------------+----+----+---+
 | Non-linear activation | N  | N  | Y |
 +-----------------------+----+----+---+

Data Sequence:

Take BIAS addition as an example, If BIAS/Operand Data is per element:

Point processing input/output sequence is determined by convolution
output sequences. In most of cases, input and output sequence orders in
all input/output interfaces are the same, and it is exactly the
convolution output sequence which is shown in following diagram.

.. _fig_image37_sdp_sequence:

.. figure:: ias_image37_sdp_sequence.png
  :align: center

  Point processing input/output sequences

Bias/Operand Data is Per Channel:

Data will be fetched from memory, and stick on one value for multiple
cycles when feature data is processing on the same surface, and update
to the value of the next surface when feature data is changing to next
surface.

Bias/Operand Data is Per Cube:

Data will be set by register, and will not change after the current
hardware lay start to operand till the end.

Buffer Size Estimation
~~~~~~~~~~~~~~~~~~~~~~

There are three major buffers in single data processing subunit, LUT in
activation block, read DMA buffer, and write DMA buffer. LUT size is
(65+ 257) \*2(BPE) = 644Bytes.

For feature read DMA buffer in M, there are two constraints for
determining its size. One is covering internal SRAM accessing latency,
currently the latency is 128 cycles. The other is accessing bandwidth.
Each partial feature data element is 16 bits, and SDP needs to process
16 elements per cycle, so the required bandwidth is 32 bytes. So, read
DMA buffer size is\ :math:`128 \times 32 = 4\ KBytes`.

Different from feature data, if BS/BN/EW has to support BatchNorm mode
which has 32bits per element thus the read DMA buffer size for those 2
modules are: 32(bits)*128(cycles)*16(elements)/8=8Kbytes.

.. power-consideration-6:

Power Consideration
~~~~~~~~~~~~~~~~~~~

Element wise/BatchNorm operation are not always included in a network,
which means for most of time, BS/BN/EW are not fully running thus clock
gating is mandatory.

Planar Data Processor
---------------------

.. overview-9:

Overview
~~~~~~~~

Planar processing responses for executing operations among width x
height plane. In NVDLA version 1.0, planar processing is designed to
accomplish pooling layer. In NVDLA version 1.0, max/min/mean pooling
methods are supported. Several neighboring input elements within a plane
will send to a non-linear function to compute one output element.
Following diagram shows an example for max-pooling, the maximum value
among 3x2 neighboring elements is the pooling result value.

.. _fig_image38_max_pooling:

.. figure:: ias_image38_max_pooling.png
  :align: center

  Max-pooling example

Following diagram shows internal blocks of planar processing sub-unit
and connections to other sub-units. The diagram is just for capturing
ideas, is not the actual RTL modules and hierarchies. Planar data
processing sub-unit receives data from SDP or MCIF/SRAMIF, and sends
data to MCIF/SRAMIF.

.. _fig_image39_pdp:

.. figure:: ias_image39_pdp.png
  :align: center

  Planar processing block diagram

.. _fig_image40_pdp:

.. figure:: ias_image40_pdp_processing.png
  :align: center

  Processing flow in one plane

Pooling operations are done within a plane. There is no interference
between different planes. :numref:`fig_image41_pdp_in_mode0` shows a complete scheme of
pooling in one plane. The offset of two neighboring kernel is called
stride. When stride is less than *R* and *S* of a kernel, there are
overlapped lines. Some line may be used by more than two neighboring
kernels. Input data is streaming in raster-scan order. For each pooling
kernel, the operated data is also streaming in raster scan order.

If an input data element is the first element of a kernel, it will be
stored to the share line buffer, data in the share line buffer is called
partial result. If an input data element is neither the first element
nor the last element of a kernel, it will be operated with the existed
partial result from share buffer, and the result will be stored to the
same entry of the original partial result. Partial result calculation is
done in pre-processing block.

1. In cases of max/min pooling schemes, the partial result is the
   maximum/minimum value of the input element and the original partial
   result.

2. In case of mean pooling scheme, the partial result is the sum of the
   input element and the original partial result.

If an input data element is the last element of a kernel, it will be
operated with the existed partial result from the share line buffer to
generate a pre-final result. Post-processing block fetch pre-final
result from shared line buffer, after proper operations, it generates
the final result, and then send out to SRAMIF/MCIF.

1. In cases of max/min pooling schemes, the pre-final result is the
   final result, no extra operation is needed.

2. In case of mean pooling scheme, the final result could be calculated
   by
   :math:`pre\_ final\_ result \times \frac{1}{\text{Kerne}l_{\text{width}} \times Kernel_{\text{height}}} = pre\_ final\_ result \times scale\_ factor\_ width \times scale\_ factor\_ height`.
   Division is hard for hardware implementation, so a pair of
   :math:`scale\_ factor` is used to transform division into
   multiplication.

The greatest number of kernels which share the same line of data is
determined by
:math:`\text{ceiling}\left( \frac{Kernel\_ Height}{Stride\_ H} \right)`.
The total buffer entry number needed within a plane
is\ :math:`width\_ out \times ceiling\left( \frac{Kernel\_ Height}{Stride\_ H} \right)`
, and in RTL design the assigned total buffer entry number
:math:`total\_ buf\_ entry` within one plane is as below, and 112bits
for each entry:

(a) if
    :math:`\text{ceiling}\left( \frac{Kernel\_ Height}{Stride\_ H} \right)`
    = 1, :math:`total\_ buf\_ entry`\ =16*4*8=512;

(b) if
    :math:`\text{ceiling}\left( \frac{Kernel\_ Height}{Stride\_ H} \right)`
    = 2, :math:`total\_ buf\_ entry`\ =16*4*4=256;

(c) if
    :math:`\text{ceiling}\left( \frac{Kernel\_ Height}{Stride\_ H} \right)`
    = 3 or 4, :math:`total\_ buf\_ entry`\ =16*4*2=128;

(d) if
    :math:`\text{ceiling}\left( \frac{Kernel\_ Height}{Stride\_ H} \right)`
    > 4, :math:`total\_ buf\_ entry`\ =16*4*1=64;

Since pooling operation is a down sampling method, there are a
significant amount of information are discarded, pooling in a large
kernels are too destructive. In current analyzed networks, there are
three most common cases, one is pooling size 3x3, with
stride 2x2. The other is pooling
size 2x2, with stride 2x2, and the last
is pooling size is 3x3, with stride 1x1.
There are two other less used cases: one is pooling
size 3x3, with stride 3x3. And the other
is pooling size 7x7, with stride 1x1.

.. table:: Pooling Kernel Type Summary
 :name: tab_pooling_kerne_type

 +---------+---------+---------+---------+---------+---------+---------+
 | Network | Total   | size 3x3| size 2x2| size 3x3| Other   | Other   |
 |         | Pooling | stride  | stride  | stride  | Layer   | Layer   |
 |         | Layer   | 2x2     | 2x2     | 1x1     | Number  | Pooling |
 |         | Number  | Number  | Number  | Number  |         | Format  |
 +=========+=========+=========+=========+=========+=========+=========+
 | AlexNet | 3       | 3       | 0       | 0       | 0       | NA      |
 +---------+---------+---------+---------+---------+---------+---------+
 | Overfea | 3       | 0       | 1       | 0       | 2       | size 3x3|
 | t-Accur |         |         |         |         |         | stride  |
 | ate     |         |         |         |         |         | 3x3     |
 +---------+---------+---------+---------+---------+---------+---------+
 | VGG 19  | 5       | 0       | 5       | 0       | 0       | NA      |
 +---------+---------+---------+---------+---------+---------+---------+
 | GoogLeN | 14      | 4       | 0       | 9       | 1       | size 7x7|
 | et      |         |         |         |         |         | stride  |
 |         |         |         |         |         |         | 1x1     |
 +---------+---------+---------+---------+---------+---------+---------+
 | NVDrive | 12      | 3       | 0       | 9       | 0       | NA      |
 | Net@960 |         |         |         |         |         |         |
 | x540    |         |         |         |         |         |         |
 +---------+---------+---------+---------+---------+---------+---------+

So 2 ~ 8 pooling kernel size (both in width and height) range and 1~8
stride range is enough for normal usage. In real RTL design, we set the
pooling kernel size range to 1~8, and set the stride range to 1 ~ 16.

There are two input paths for planar data processing sub-unit, one is
single point data processing sub-unit, and the other is external ram
(MC/SRAM). There is one output data path for planar processing
sub-unit, output data is always sent to external ram (MC/SRAM). In
common practices, a pooling layer is inserted after a convolutional
layer. To save memory accessing consumptions, planar data processing
sub-unit shall directly receive data from point processing unit if
following condition is meet. Suppose output width is
:math:`\text{Width}_{\text{output}}`, total buffer size in byte is
:math:`\text{Size}_{\text{buffer}}`, overlapped line number
:math:`\text{Num}_{overlapped\_ line}`, Data width in byte is
:math:`\text{Data}_{\text{width}}`, the number of spatial plane is
called ongoing channel number :math:`\text{Num}_{ongoing\_ channels}`,
normally, :math:`\text{Num}_{ongoing\_ channels}` should be equals to
kernel_per_group (16 for INT16/FP16, 32 for INT8 pipe). Below is the
planar processing on-fly operation condition.

.. math:: Width_{output} \leq \frac{{Size}_{{buffer}}}{{Data}_{{width}} \times {Num}_{ongoing\_ channels} \times {Num}_{overlapped\_ line}} = \frac{{Size}_{{buffer}}}{{Data}_{{width}} \times {Num}_{ongoing\_ channels} \times f(ceil\left( \frac{{Height}_{{poolin}g_{{kernel}}}}{{Strid}e_{h}} \right))}

.. Planar processing on-fly operation condition

If
:math:`\text{ceil}\left( \frac{\text{Height}_{\text{poolin}g_{\text{kernel}}}}{\text{Strid}e_{h}} \right) = 1`
,
:math:`f\left( \text{ceil}\left( \frac{\text{Height}_{\text{poolin}g_{\text{kernel}}}}{\text{Strid}e_{h}} \right) \right) = 1`

If
:math:`\text{ceil}\left( \frac{\text{Height}_{\text{poolin}g_{\text{kernel}}}}{\text{Strid}e_{h}} \right) = 2`
,
:math:`f\left( \text{ceil}\left( \frac{\text{Height}_{\text{poolin}g_{\text{kernel}}}}{\text{Strid}e_{h}} \right) \right) = 2`

If
:math:`\text{ceil}\left( \frac{\text{Height}_{\text{poolin}g_{\text{kernel}}}}{\text{Strid}e_{h}} \right) = 3\ or\ 4`
,
:math:`f\left( \text{ceil}\left( \frac{\text{Height}_{\text{poolin}g_{\text{kernel}}}}{\text{Strid}e_{h}} \right) \right) = 4`

If
:math:`\text{ceil}\left( \frac{\text{Height}_{\text{poolin}g_{\text{kernel}}}}{\text{Strid}e_{h}} \right) > 4`
,
:math:`f\left( \text{ceil}\left( \frac{\text{Height}_{\text{poolin}g_{\text{kernel}}}}{\text{Strid}e_{h}} \right) \right) = 8`

When data is from point processing sub-unit, input data sequence is the
same as convolution output sequences which is shown in following
diagram.

.. _fig_image41_pdp_in_mode0:

.. figure:: ias_image41_pdp_in_mode0.png
  :align: center

  Planar processing input sequence, mode 0

And output sequence is shown in following diagram.

.. _fig_image42_pdp_out_mode0:

.. figure:: ias_image42_pdp_out_mode0.png
  :align: center

  Planar processing output sequence, mode 0

If planar processing on-fly operation condition is not meet, planar processing shall work in off-fly
mode, it receives data from PDMA, and the ongoing channel number is
always 16. There are two sub-cases, one is non-split-width, and the
other is split-width. Input data sequence is shown in following diagram

.. _fig_image43_pdp_in_mode1_2:

.. figure:: ias_image43_pdp_in_mode1_2.png
  :align: center

  Planar processing input sequence, mode 1 and 2

And output data sequence is shown in following diagram.

.. _fig_image44_pdp_out_mode1_2:

.. figure:: ias_image44_pdp_out_mode1_2.png
  :align: center

  Planar processing output sequence, mode 1 and 2

+----------------+------------------------------+-------------+
| Operation mode | Data Source                  | Split-Width |
+================+==============================+=============+
| Mode 0         | Single-point Data Processing | No          |
+----------------+------------------------------+-------------+
| Mode 1         | MC/SRAM                      | No          |
+----------------+------------------------------+-------------+
| Mode 2         | MC/SRAM                      | Yes         |
+----------------+------------------------------+-------------+

.. buffer-size-estimation-1:

Buffer Size Estimation
~~~~~~~~~~~~~~~~~~~~~~

There are three major buffers in planar data processing subunit, share
line buffer, read DMA buffer, and write DMA buffer. For share line
buffer, its size determines PDP would on-fly co-working with SDP or not.
Based on input data cube
height\ :math:`\text{Height}_{\text{input data cube}}`, pooling kernel
height\ :math:`\text{Height}_{\text{pooling kernel}}`, pooling kernel
stride in height
direction\ :math:`\text{stride}_{\text{pooling kernel}}`, output data
cube width\ :math:`\text{Width}_{\text{output data cube}}`, group size
(16 elements of int16/FP16 or 32 elements of int8, ~32
byte)\ :math:`\ \text{Group}_{\text{size}}` and bytes_per_element(14/8
for INT8, 28/8 for INT16, 28/8 for FP16).

.. math:: Buffer\ Size = \text{Width}_{\text{output data cube}}*\frac{\text{Height}_{\text{pooling kernel}}}{\text{stride}_{\text{pooling kernel}}}*\text{Group}_{\text{size}}*bytes\_ per\_ element

If those share line buffer size is less than the consumption size, PDP
have to work in off-fly mode, so there will be performance drop since
extra-time is needed to store data to MC/SRAM, and then fetch back to
PDP for pooling processing.

.. table:: Pooling Share Line Buffer Consumption Summary
 :name: tab_pooling_buffer_size

 +---------+---------+---------+---------+---------+---------+---------+
 | Layer   | Channel | Kernel  | Kernel  | Output  | Minimum | Maximum |
 |         | Number  | Size    | Stride  | Width   | Size    | Size    |
 +=========+=========+=========+=========+=========+=========+=========+
 | AlexNet | 96      | 3       | 2       | 27      | 1728    | 5184    |
 | – pool1 |         |         |         |         |         |         |
 +---------+---------+---------+---------+---------+---------+---------+
 | AlexNet | 128     | 3       | 2       | 13      | 832     | 3328    |
 | – pool2 |         |         |         |         |         |         |
 +---------+---------+---------+---------+---------+---------+---------+
 | AlexNet | 128     | 3       | 2       | 6       | 384     | 1536    |
 | – pool5 |         |         |         |         |         |         |
 +---------+---------+---------+---------+---------+---------+---------+
 | Overfea | 96      | 3       | 3       | 36      | 1152    | 6912    |
 | t-Accur |         |         |         |         |         |         |
 | ate     |         |         |         |         |         |         |
 | – layer |         |         |         |         |         |         |
 | 3       |         |         |         |         |         |         |
 +---------+---------+---------+---------+---------+---------+---------+
 | Overfea | 256     | 2       | 2       | 15      | 480     | 7680    |
 | t-Accur |         |         |         |         |         |         |
 | ate     |         |         |         |         |         |         |
 | – layer |         |         |         |         |         |         |
 | 6       |         |         |         |         |         |         |
 +---------+---------+---------+---------+---------+---------+---------+
 | Overfea | 1024    | 3       | 3       | 5       | 160     | 10240   |
 | t-Accur |         |         |         |         |         |         |
 | ate     |         |         |         |         |         |         |
 | – layer |         |         |         |         |         |         |
 | 19      |         |         |         |         |         |         |
 +---------+---------+---------+---------+---------+---------+---------+
 | VGG 19  | 64      | 2       | 2       | 112     | 3584    | 14336   |
 | – pool1 |         |         |         |         |         |         |
 +---------+---------+---------+---------+---------+---------+---------+
 | VGG 19  | 128     | 2       | 2       | 56      | 1792    |   14336 |
 | – pool2 |         |         |         |         |         |         |
 +---------+---------+---------+---------+---------+---------+---------+
 | VGG 19  | 256     | 2       | 2       | 28      | 896     |   14336 |
 | – pool3 |         |         |         |         |         |         |
 +---------+---------+---------+---------+---------+---------+---------+
 | VGG 19  | 512     | 2       | 2       | 14      | 448     |   14336 |
 | – pool4 |         |         |         |         |         |         |
 +---------+---------+---------+---------+---------+---------+---------+
 | VGG 19  | 512     | 2       | 2       | 7       | 224     | 7168    |
 | – pool5 |         |         |         |         |         |         |
 +---------+---------+---------+---------+---------+---------+---------+
 | GoogLeN | 64      | 3       | 2       | 56      | 3584    | 7168    |
 | et      |         |         |         |         |         |         |
 | -       |         |         |         |         |         |         |
 | pool1/3 |         |         |         |         |         |         |
 | x3_s2   |         |         |         |         |         |         |
 +---------+---------+---------+---------+---------+---------+---------+
 | GoogLeN | 192     | 3       | 2       | 56      | 3584    |   21504 |
 | et      |         |         |         |         |         |         |
 | -       |         |         |         |         |         |         |
 | pool2/3 |         |         |         |         |         |         |
 | x3_s2   |         |         |         |         |         |         |
 +---------+---------+---------+---------+---------+---------+---------+
 | GoogLeN | 192     | 3       | 1       | 28      | 2688    |   32256 |
 | et      |         |         |         |         |         |         |
 | -       |         |         |         |         |         |         |
 | incepti |         |         |         |         |         |         |
 | on_3a/p |         |         |         |         |         |         |
 | ool     |         |         |         |         |         |         |
 +---------+---------+---------+---------+---------+---------+---------+
 | GoogLeN | 256     | 3       | 1       | 28      | 2688    |   43008 |
 | et      |         |         |         |         |         |         |
 | -       |         |         |         |         |         |         |
 | incepti |         |         |         |         |         |         |
 | on_3b/p |         |         |         |         |         |         |
 | ool     |         |         |         |         |         |         |
 +---------+---------+---------+---------+---------+---------+---------+
 | GoogLeN | 480     | 3       | 2       | 14      | 896     |   13440 |
 | et      |         |         |         |         |         |         |
 | -       |         |         |         |         |         |         |
 | pool3/3 |         |         |         |         |         |         |
 | x3_s2   |         |         |         |         |         |         |
 +---------+---------+---------+---------+---------+---------+---------+
 | GoogLeN | 480     | 3       | 1       | 14      | 1344    |   40320 |
 | et      |         |         |         |         |         |         |
 | -       |         |         |         |         |         |         |
 | incepti |         |         |         |         |         |         |
 | on_4a/p |         |         |         |         |         |         |
 | ool     |         |         |         |         |         |         |
 +---------+---------+---------+---------+---------+---------+---------+
 | GoogLeN | 512     | 3       | 1       | 14      | 1344    |   43008 |
 | et      |         |         |         |         |         |         |
 | -       |         |         |         |         |         |         |
 | incepti |         |         |         |         |         |         |
 | on_4b/p |         |         |         |         |         |         |
 | ool     |         |         |         |         |         |         |
 +---------+---------+---------+---------+---------+---------+---------+
 | GoogLeN | 512     | 3       | 1       | 14      | 1344    |   43008 |
 | et      |         |         |         |         |         |         |
 | -       |         |         |         |         |         |         |
 | incepti |         |         |         |         |         |         |
 | on_4c/p |         |         |         |         |         |         |
 | ool     |         |         |         |         |         |         |
 +---------+---------+---------+---------+---------+---------+---------+
 | GoogLeN | 512     | 3       | 1       | 14      | 1344    |   43008 |
 | et      |         |         |         |         |         |         |
 | -       |         |         |         |         |         |         |
 | incepti |         |         |         |         |         |         |
 | on_4d/p |         |         |         |         |         |         |
 | ool     |         |         |         |         |         |         |
 +---------+---------+---------+---------+---------+---------+---------+
 | GoogLeN | 528     | 3       | 1       | 14      | 1344    |   44352 |
 | et      |         |         |         |         |         |         |
 | -       |         |         |         |         |         |         |
 | incepti |         |         |         |         |         |         |
 | on_4e/p |         |         |         |         |         |         |
 | ool     |         |         |         |         |         |         |
 +---------+---------+---------+---------+---------+---------+---------+
 | GoogLeN | 832     | 3       | 2       | 7       | 448     |   11648 |
 | et      |         |         |         |         |         |         |
 | -       |         |         |         |         |         |         |
 | pool4/3 |         |         |         |         |         |         |
 | x3_s2   |         |         |         |         |         |         |
 +---------+---------+---------+---------+---------+---------+---------+
 | GoogLeN | 832     | 3       | 1       | 7       | 672     |   34944 |
 | et      |         |         |         |         |         |         |
 | -       |         |         |         |         |         |         |
 | incepti |         |         |         |         |         |         |
 | on_5a/p |         |         |         |         |         |         |
 | ool     |         |         |         |         |         |         |
 +---------+---------+---------+---------+---------+---------+---------+
 | GoogLeN | 832     | 3       | 1       | 7       | 672     |   34944 |
 | et      |         |         |         |         |         |         |
 | -       |         |         |         |         |         |         |
 | incepti |         |         |         |         |         |         |
 | on_5b/p |         |         |         |         |         |         |
 | ool     |         |         |         |         |         |         |
 +---------+---------+---------+---------+---------+---------+---------+
 | GoogLeN | 1024    | 7       | 1       | 1       | 224     |   14336 |
 | et      |         |         |         |         |         |         |
 | -       |         |         |         |         |         |         |
 | pool5/7 |         |         |         |         |         |         |
 | x7_s1   |         |         |         |         |         |         |
 +---------+---------+---------+---------+---------+---------+---------+

In the above table, most of the
minimum cases are less than 7Kbytes. So as a result of balancing
performance and the share line buffer size is set as 7Kbyte.

For read DMA buffer, there are two constraints for determining its size.
One is covering MC accessing latency, currently the latency is 128
cycles. The other is accessing bandwidth, the peak performance case is 8
Bytes per cycle (8 elements in int8, 4 elements in int16/fp16). So read
DMA buffer size is\ :math:`128 \times 8 = 1KBytes`.

.. power-consideration-7:

Power Consideration
~~~~~~~~~~~~~~~~~~~

Planar processing sub-unit targets for pooling layer in NVDLA 1.0, based
on analysis on current network, planar processing usage is not so
popular.

.. table:: Pooling Layer Percentage Summary
 :name: tab_pooling_layer_percentage

 +-----------------+-----------------+-----------------+-----------------+
 | Network         | Total Pooling   | Total Layer     | Percentage      |
 |                 | Layer Number    | Number\*        |                 |
 +=================+=================+=================+=================+
 | AlexNet         | 3               | 13              | 23%             |
 +-----------------+-----------------+-----------------+-----------------+
 | Overfeat-Accura | 3               | 12              | 25%             |
 | te              |                 |                 |                 |
 +-----------------+-----------------+-----------------+-----------------+
 | VGG 19          | 5               | 24              | 21%             |
 +-----------------+-----------------+-----------------+-----------------+
 | GoogLeNet       | 14              | 74              | 19%             |
 +-----------------+-----------------+-----------------+-----------------+

\* Total Layer Number = Convolution (including FC) + Pooling + LRN

Base on pooling layer number percentage (we don’t have calculation time
percentage yet), it’s highly possible that planar processing sub-unit is
idle most of the time. Sub-unit level clock gating is a must.

Cross Channel Data Processor
----------------------------

.. overview-10:

Overview
~~~~~~~~

Channel processing responses for executing operations along channel
direction. In NVDLA version 1.0, channel processing is designed to
address local response normalization layer. The local response
normalization performs a kind of lateral inhibition by normalizing over
local input region along channel direction. The normalization function
is shown as follow

.. math:: \text{Result}_{w,h,c} = \frac{\text{Source}_{w,h,c}}{{(j + \frac{\alpha}{n}\sum_{i = max(0,c - \frac{n}{2})}^{min(C - 1,\ c + \frac{n}{2})}\text{Source}_{w,h,i}^{2})}^{\beta}}

.. 19 Local response normalization formula

Local region shape is always\ :math:`1 \times 1 \times n`. Number
:math:`n` is configurable, and its range
is\ :math:`\lbrack 3,5,7,9\rbrack`. Arithmetic such as division and
fractional exponents are not friendly with ASIC design. The above equation
could be decomposed into

.. math:: \text{Result}_{w,h,c} = \text{Source}_{w,h,c} \times f(\sum_{i = max(0,c - \frac{n}{2})}^{min(C - 1,\ c + \frac{n}{2})}\text{Source}_{w,h,i}^{2})

.. math:: f\left( x \right) = \frac{1}{{(j + \frac{\alpha}{n} \times x)}^{\beta}}

..  RESMO Function in Local Response Normalization Formula

Be noticed the
:math:`\sum_{i = max(0,c - \frac{n}{2})}^{min(C - 1,\ c + \frac{n}{2})}\text{Source}_{w,h,i}^{2}`
and :math:`\text{Source}_{w,h,c} \times f(x)` can be bypassed by
programming corresponding register so that CDP can be treated as a
standalone LUT function.

Look-up table approach is adopted for the RESMO
(reciprocation-exponent-sum-multi operation)\ :math:`f\left( x \right)`.

.. _fig_image45_cdp_curve:

.. figure:: ias_image45_cdp_curve.svg
  :align: center

  Curve for reciprocation-exponent-sum-multi operation

Following diagram shows internal blocks of channel data processing
sub-unit and connections to other sub-units. The diagram is just for
capturing ideas, is not the actual RTL modules and hierarchies.

.. _fig_image46_cdp:

.. figure:: ias_image46_cdp.png
  :align: center

  Cross Channel Data Processing Block diagram

Channel processing sub-unit always works independently with other
processing sub-units. It receives input data from and send output data
to PDMA. Due to memory accessing constraint, the input data sequence is
in a particular orders. The input sequence is shown in following
diagram, and output sequence is the same as input sequence.

.. _fig_image47_cdp_seq:

.. figure:: ias_image47_cdp_seq.png
  :align: center

  Channel Processing input/output sequence

Following table shows LRN layers parameters in current well know network

.. table:: LRN Layer Parameter Summary
 :name: tab_lrn_layer

 +-------------------+------------------------+-------------------+--------+------+
 | Network           | Total LRN Layer Number | Local Size Number | Alpha  | beta |
 +===================+========================+===================+========+======+
 | AlexNet           | 2                      | 5                 | 0.0001 | 0.75 |
 +-------------------+------------------------+-------------------+--------+------+
 | Overfeat-Accurate | 0                      | NA                | 0.0001 | 0.75 |
 +-------------------+------------------------+-------------------+--------+------+
 | VGG-19            | 0                      | NA                | 0.0001 | 0.75 |
 +-------------------+------------------------+-------------------+--------+------+
 | GoogLeNet         | 2                      | 5                 | 0.0001 | 0.75 |
 +-------------------+------------------------+-------------------+--------+------+

Data elements on stripe edge may be used by to neighboring stripes.
Those data needs to be buffered, buffer entry number shall
be\ :math:`\left\lbrack \text{Max}\left( \text{loca}l_{\text{regio}n_{\text{size}}} \right) - 1 \right\rbrack \times 8 = 7 \times 8 = 56\ byte`.

.. buffer-size-estimation-2:

Buffer Size Estimation
~~~~~~~~~~~~~~~~~~~~~~

There are three major buffers in cross-channel data processing subunit,
LUT in activation block, read DMA buffer, and write DMA buffer. LUT size
is the same as SDP (644Bytes).

For read DMA buffer, there are two constraints for determining its size.
One is covering MC accessing latency, currently the latency is 128
cycles. The other is accessing bandwidth, the peak performance case is 8
Bytes per cycle (8 elements in int8, 4 elements in int16/fp16). So read
DMA buffer size is\ :math:`128 \times 8 = 1KBytes`.

.. power-consideration-8:

Power Consideration
~~~~~~~~~~~~~~~~~~~

Channel processing sub-unit targets for LRN layer in NVDLA 1.0, based on
analysis on current network, channel processing usage is not so popular.

.. table:: Local Response Layer Percentage
 :name: tab_lrn_percentage

 +-------------------+------------------------+----------------------+------------+
 | Network           | Total LRN Layer Number | Total Layer Number\* | Percentage |
 +===================+========================+======================+============+
 | AlexNet           | 2                      | 13                   | 15%        |
 +-------------------+------------------------+----------------------+------------+
 | Overfeat-Accurate | 0                      | 12                   | 0%         |
 +-------------------+------------------------+----------------------+------------+
 | VGG 19            | 0                      | 24                   | 0%         |
 +-------------------+------------------------+----------------------+------------+
 | GoogLeNet         | 2                      | 74                   | 3%         |
 +-------------------+------------------------+----------------------+------------+

\* Total Layer Number = Convolution (including FC) + Pooling + LRN

Base on local response normalization layer number percentage (we don’t
have calculation time percentage yet), it’s highly possible that cross
channel data processing sub-unit is idle most of the time. Sub-unit
level clock gating is a must.

RUBIK
-----

.. overview-11:

Overview
~~~~~~~~

RUBIK module is similar to BDMA. It transforms data mapping format
without any data calculation. RUBIK has 3 working modes, they are:

-  contract data cube

-  split feature data cube into multi-planar formats

-  merge multi-planar formats to data cube

Since the module is to transform feature data cubes, we call it RUBIK
unit.

.. _fig_image48_cdp:

.. figure:: ias_image48_cdp.png
  :align: center

  RUBIK

Contract
~~~~~~~~

A SW deconvolution layer always uses several HW-layers or two phases.
Phase I is generate result by convolution pipeline. And phase II is
contract mode by RUBIK.

Normally, a SW deconvolution layer has deconvolution x stride and y
stride that are greater than 1. And with these strides the output of
phase I HW-layer is a channel-extended data cube. Contract mode in RUBIK
transforms mapping format to de-extend the cube. Figure below show a
remapping example which x stride is 2 and y stride is 3.

.. _fig_image49_rubik_contract:

.. figure:: ias_image49_rubik_contract.svg
  :align: center

  Contract mode in RUBIK

The formula of input cube size and output size are:

.. math:: W^{'} = W*deconv\_ x\_ stride

.. math:: H^{'} = H*deconv\_ y\_ stride

.. math:: C^{'} = \frac{C}{deconv\_ x\_ stride*deconv\_ y\_ stride\ }

..  Formula of data cube size in contract mode

RUBIK engine does contract slice by slice. I take one Wx1xC input slice
and convert it to a W’xH’xC’ output sub cube. Then the next input slice.
It never sends request across line boundary.

When doing contract, the input/output start address and line stride
shall align to 32 bytes. It always tries to send 256 byte requests. The
memory efficiency is between 80%~100% which is affected by start
address. If all address stride and start address are 256 byte aligned,
the memory efficiency reaches 100%.

Requirement of contract mode:

-  The channel size shall be divisible by deconvolution x stride, y
   stride and 32 bytes. As formula below:

.. math:: C\ \%\ \left( \text{decon}v_{x_{\text{stride}}}*deconv_{y_{\text{stride}}}*32 \right) == 0

-  Each dimension of input and output data cube, like input data width,
   output data width, input channel size, should not exceed 8192 in one
   contract layer.

Split and Merge
~~~~~~~~~~~~~~~

Split and merge are two opposite operation modes in RUBIK. Split
transforms a data cube into M-planar formats (NCHW). The number of plane
is equal to channel size. Merge transforms a serial of planes to a
feature data cube. The transform is showed in figure below.

.. _fig_image50_rubik_split_and_merge:

.. figure:: ias_image50_rubik_split_and_merge.svg
  :align: center

  Split and merge modes in RUBIK

The M-planar format is similar to image formats. It’s a pitch linear
format which contains T_R16_I, T_R8_I or T_R16_F data. Each plane
contains only 1 channel data or single element. The line stride and
planar stride of all planes(M-planar) shall align to 64 bytes. It’s
unlike other data formats for NVDLA.

.. power-consideration-9:

Power Consideration
~~~~~~~~~~~~~~~~~~~

RUBIK unit applies SLCG in data path. The clock of data path of RUBIK is
gated by SLCG when it is idle and no HW-layer is available from
programmable registers. While SLCG does not clock gate the regfile sub
module inside the sub unit.

MCIF 
-----

MCIF is to arbitrate requests from several internal sub modules and
convert to AXI protocol to connect to external DRAM. The interface
between NVDLA internal sub module and MCIF is an internal interface
defined by NVDLA team, and will not expose externally.

.. _fig_image51_mcif:

.. figure:: ias_image51_mcif.png
  :align: center

  MCIF

MCIF will support both AXI read and AXI write channel, but some NVDLA
sub-module will only have read requirement, so the interface between
sub-module and MCIF will support read, write or both. CDMA0 and CDMA1 in
above diagram will need read only, and other 5 will need both read and
write.

SRAMIF 
-------

SRAMIF(previous named as SRAMIF) is to connect several internal
sub-module to SRAM.

.. _fig_image52_sramif:

.. figure:: ias_image52_sramif.png
  :align: center

  SRAMIF

SRAMIF will support both AXI read and AXI write channel, but some NVDLA
sub-module will only have read requirement, so the interface between
sub-module and SRAMIF will support read, write or both. CMDA0~1 will
need read channel only, while the other 5 will need both read and write.

Result Statistics
-----------------

To perform better calculation accuracy with limited precision data type
like int8, NVDLA engine involved a large number of converters in many
pipeline stages. Please see Section "Precision programming" of Programming Guide document for more
details.

In the runtime, conversion parameters can be both static value and
dynamic value. To support the latter ones, SW needs NVDLA HW to give
rough statistic of output feature data cube and calculate the parameters
accordingly.

To achieve that, NVDLA implements result statistic registers in almost
all the pipeline stages. These registers record:

-  Number of results that is equals to max non-infinity values.

-  Number of INF/NaN on input port

-  Number of INF on output port

Based on these statistic record, SW can tell rough situation of output
feature data cube and figure out proper parameters for next layer.

The pipeline stages involved in result statistic are:

-  Convolution DMA

-  Convolution accumulator

-  Single data processor

-  Planar data processor

-  Cross channel data processor

Here’s a list of statistic counting registers and its valid condition:

+----------------------------------+-------------------------------------+
| Register                         | Valid condition                     |
+==================================+=====================================+
| CDMA. D_INF_INPUT_DATA_NUM       | CDMA. IN_PRECISION==FP16            |
|                                  |                                     |
| CDMA. D_INF_INPUT_WEIGHT_NUM     |                                     |
+----------------------------------+-------------------------------------+
| CDMA. D_NAN_INPUT_DATA_NUM       | CDMA. IN_PRECISION==FP16            |
|                                  |                                     |
| CDMA. D_NAN_INPUT_WEIGHT_NUM     |                                     |
+----------------------------------+-------------------------------------+
| SDP_RDMA.D_STATUS_NAN_INPUT_NUM  | SDP_RDMA.IN_PRECISION==FP16 &&      |
| SDP_RDMA. D_STATUS_INF_INPUT_NUM |                                     |
|                                  | SDP_RDMA.PERF_NAN_INF_COUNT_EN==YES |
+----------------------------------+-------------------------------------+
| SDP. D_STATUS_NAN_INPUT_NUM      | Not used                            |
|                                  |                                     |
| SDP. D_STATUS_INF_INPUT_NUM      |                                     |
+----------------------------------+-------------------------------------+
| SDP. D_STATUS_NAN_OUTPUT_NUM     | SDP.OUT_PRECISION==FP16 &&          |
|                                  |                                     |
|                                  | SDP. PERF_NAN_INF_COUNT_EN=YES      |
+----------------------------------+-------------------------------------+
| CDP. D_INF_INPUT_NUM             | CDP. INPUT_DATA_TYPE=FP16           |
|                                  |                                     |
| CDP. D_NAN_INPUT_NUM             |                                     |
|                                  |                                     |
| CDP. D_NAN_OUTPUT_NUM            |                                     |
+----------------------------------+-------------------------------------+
| PDP. D_INF_INPUT_NUM             | PDP. INPUT_DATA =FP16               |
|                                  |                                     |
| PDP. D_NAN_INPUT_NUM             |                                     |
|                                  |                                     |
| PDP. D_NAN_OUTPUT_NUM            |                                     |
+----------------------------------+-------------------------------------+

Pipelines of NVDLA core
-----------------------

All sub units in NVDLA core logic is introduced in sections above. Some
sub units are combined as one pipeline; some are working as individual
pipelines. All of possible pipeline working modes are summarized in
table below.

.. table:: pipeline working mode of NVDLA core
 :name: tab_pipelines_nvdla_core

 +-----------------+-----------------+-----------------+-----------------+
 | Pipeline        | Sub Units       | Interface       | Support Layers  |
 | Working Mode    |                 |                 |                 |
 +=================+=================+=================+=================+
 | Convolution     | CDMA, CBUF,     | MCIF/SRAMIF     | direct          |
 | pipeline        | CSC, CMAC,      |                 | convolution,    |
 |                 | CACC, SDP       |                 | Winograd, image |
 |                 |                 |                 | input layer,    |
 |                 |                 |                 | element-wise    |
 |                 |                 |                 | layer,          |
 |                 |                 |                 | batch-normaliza |
 |                 |                 |                 | tion,           |
 |                 |                 |                 | activation      |
 |                 |                 |                 | (relu, prelu,   |
 |                 |                 |                 | sigmoid, etc.)  |
 |                 |                 |                 | layer           |
 +-----------------+-----------------+-----------------+-----------------+
 | Convolution &   | CDMA, CBUF,     | MCIF/SRAMIF     | direct          |
 | pooling         | CSC, CMAC,      |                 | convolution \+  |
 | pipeline        | CACC, SDP, PDP  |                 | pooling layer   |
 |                 |                 |                 |                 |
 |                 |                 |                 | (multi-batch    |
 |                 |                 |                 | mode is not     |
 |                 |                 |                 | supported)      |
 +-----------------+-----------------+-----------------+-----------------+
 | Offline SDP     | SDP             | MCIF/SRAMIF     | activation      |
 | pipeline        |                 |                 | layer,          |
 |                 |                 |                 | element-wise    |
 |                 |                 |                 | layer,          |
 |                 |                 |                 | batch-normaliza |
 |                 |                 |                 | tion            |
 |                 |                 |                 | layer, format   |
 |                 |                 |                 | conversion      |
 |                 |                 |                 | layer,          |
 |                 |                 |                 | comparison      |
 |                 |                 |                 | layer           |
 +-----------------+-----------------+-----------------+-----------------+
 | Offline PDP     | PDP             | MCIF/SRAMIF     | Pooling layer   |
 | pipeline        |                 |                 |                 |
 +-----------------+-----------------+-----------------+-----------------+
 | CDP pipeline    | CDP             | MCIF/SRAMIF     | LRN layer       |
 +-----------------+-----------------+-----------------+-----------------+
 | BDMA pipeline   | BDMA            | MCIF/SRAMIF     | Data            |
 |                 |                 |                 | transmission    |
 |                 |                 |                 | layer           |
 +-----------------+-----------------+-----------------+-----------------+
 | Rubik pipeline  | Rubik           | MCIF/SRAMIF     | Data transform  |
 |                 |                 |                 | layers          |
 +-----------------+-----------------+-----------------+-----------------+
