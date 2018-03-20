=================
Programming Guide
=================

(Notice: This version of the *Programming Guide* is only for the nvdlav1
release.)

Precision Preservation
----------------------

Though most software-based DNN implementations are FP32-based, many studies have already
shown that lower precision is sufficient for inference.  As such, NVDLA chooses to
support INT8/INT16/FP16 as a trade-off between precision and
performance/area.  At the same time, NVDLA adopts technologies to
keep the precision loss under control.  Below, we give a diagram with an overview of NVDLA's
precision-preservation architecture.

.. _fig_image53_nvdla_precision:

.. figure:: ias_image53_nvdla_precision.svg
  :align: center

  NVDLA precision-preservation architecture

In total, there are four types of approaches to precision control in the NVDLA
pipeline:

-  Convertor:

   The formula for a convertor in INT8 and INT16 is: :math:`y = saturation\_round{(x - offset_{int}) * scaling_{int} >> shifter_{uint}}`

   `offset`, `scaling`, and `shifter` are programmable registers to allow
   software to control the output dynamic range. Saturation is dependent on the number of output bits.

   For INT8 and INT16, `offset` and `scaling` are treated as signed integers, and the exact
   number of bits is depends on the input operands.
   `shifter` is a 5 bits unsigned integer (always specifying a right shift); the
   rounding method used after the shift is to “round half away from zero”.

   For FP16, the dynamic range that can be represented by FP16 representable is large, and so
   convertor and shifter logic is not implemented in hardware.

   The convertor is able to keep the best possible precision even if input data are not
   symmetric to 0, or dynamic range is not 2\ :sup:`N`; NVDLA uses it
   to convert internal precision (high) to external (low, typically
   INT8/INT16/FP16).

-  Truncate:

   Truncation is enabled for INT8/INT16 only.  The formula for truncation is: :math:`y = saturate\_round (x[msb : lsb])`

   `lsb` is a programmable register; `msb` is defined as `lsb` + `output_bits`.

   Similarly to the convertor, the rounding method used in truncation is also “round
   half away from zero”.

   Truncation is used in NVDLA internal pipe where the :math:`y` has enough bits
   (compared to convertor case); it is the result of a trade-off
   between precision and area.

-  Shifter:

   The shifter exists to make sure that the bias can be added to convolution
   results; it has the formula :math:`y = saturate ( x << shifter )`.

   `shifter` is a programmable register. The saturate function depends on the number of output bits.

-  LUT:

   Look-up tables are used to deal with non-linear function in networks; these functions include
   sigmoid/tanh activation, or local response normalization as mentioned
   in `Data Formats <http://nvdla.org/hw/format.html>`_.  We use an innovative 2 level hybrid LUT to mimic those
   non-linear functions; for more detail, see see `LUT programming`_.

FP16 error threshold
~~~~~~~~~~~~~~~~~~~~

We know that the output of computations on floating-point data are highly depending on computation order.
As a result, when comparing the results of tests that are computed in FP16 space, we should allow some certain threshold of error when
comparing NVDLA's output against reference.  Below is a summary of the thresholds that are applied for each module.

+-------------+---------------------------------------------------------------------------------+
| Sub-module  | Threshold                                                                       |
+=============+=================================================================================+
| CC DC       | :math:`fabs(a-b) \leq 2^{max\_exp-20} * R * S * C * 2 * scale + 2^{exp(a)-10}`  |
+-------------+---------------------------------------------------------------------------------+
| CC Winograd | :math:`fabs(a-b) \leq 2^{max\_exp-20} * 18 * C * 2 * scale + 2^{exp(a)-10}`     |
+-------------+---------------------------------------------------------------------------------+
| SDP         | Bit-by-bit identical                                                            |
+-------------+---------------------------------------------------------------------------------+
| CDP         | :math:`(fabs(a-b) \leq 0.0001) \&\& (\frac{fabs(a-b)}{max\_value} \leq 0.001)`  |
+-------------+---------------------------------------------------------------------------------+
| PDP         | :math:`(fabs(a-b) \leq 0.0001) \&\& (\frac{fabs(a-b)}{max\_value} \leq 0.001)`  |
+-------------+---------------------------------------------------------------------------------+

In the above table, note that:

- :math:`exp(a)` in DC/Winograd is the operation to extract exponential
  field of an input: :math:`exp(a) = ((a>>10) \& 0x1f) - 15` .

- :math:`max\_exp: in DC/Winograd is the maximum exponential value of :math:`wt*data`
  inside a convolution kernel.  As data moves in a sliding window,
  max_exp is different element-by-element:

  .. math:: max\_exp=\max_{s=0...S-1\\r=0...R-1\\c=0...C-1}(exp(data_{r,s,c})\&(\sim\ 0x3)+exp(wt_{r,s,c})\&(\sim\ 0x3))

- **max_value** in CDP is the maximum value inside one square sum window.

- **max_value** in PDP is the maximum value inside one pooling window.

Convertor programming
~~~~~~~~~~~~~~~~~~~~~

Programming interface
^^^^^^^^^^^^^^^^^^^^^

As mentioned above, for a given convertor, there are 3 parameters:
`offset`, `scaling` and `shifter`.  Depending on the use cases, those
parameters have different encodings:

+-----------+----------+-----------+---------------+---------------+
| Parameter | INT->INT | INT->FP16 | FP16->INT     | FP16->FP16    |
+===========+==========+===========+===============+===============+
| Offset    | INT32    | INT32     | Not supported | Not supported |
+-----------+----------+-----------+---------------+---------------+
| Scaling   | INT16    | INT16     | Not supported | Not supported |
+-----------+----------+-----------+---------------+---------------+
| Shifter   | UINT5    | UINT5     | Not supported | Not supported |
+-----------+----------+-----------+---------------+---------------+

Convolution convertors
^^^^^^^^^^^^^^^^^^^^^^

The following is a list of place where convertors or truncation might be used for a convolution
layer (please refer to :numref:`fig_image53_nvdla_precision` about
where’re the convertors in system):

+-----------------+-----------------+-----------------+-----------------+
| Convertor       | Functionality   | Limitation      | Owner           |
+=================+=================+=================+=================+
| cc_cvt          | For image       | INT8/INT16 only | HW              |
|                 | input, this     |                 |                 |
|                 | convertor       | For FP16,       |                 |
|                 | responsible for | subtract mean   |                 |
|                 | mean            | data only       |                 |
|                 | subtraction and |                 |                 |
|                 | 8 bit           |                 |                 |
|                 | conversion;     |                 |                 |
|                 |                 |                 |                 |
|                 | For feature     |                 |                 |
|                 | input, this     |                 |                 |
|                 | convertor won’t |                 |                 |
|                 | be used         |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| wt_cvt          | Convert weight  | Offset is not   | SW              |
|                 | data to         | allowed         |                 |
|                 | INT8/16/FP16    |                 |                 |
|                 | representable   |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| pra_trunc       | Truncate the    | Used for        | HW              |
|                 | winograd        | winograd mode   |                 |
|                 | pre-transformed | and             |                 |
|                 | results to      | CSC.PROC_PRECIS |                 |
|                 | INT8/16/FP16    | ION=INT8/INT16  |                 |
|                 | representable   | only            |                 |
+-----------------+-----------------+-----------------+-----------------+
| cc_out_trunc    | Truncate the    | CACC.PROC_PRECI | HW              |
|                 | data to         | SION=INT8/INT16 |                 |
|                 | INT32/FP32      | only            |                 |
|                 | before sending  |                 |                 |
|                 | to SDP          |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| bs_cvt          | Convert bias    | N/A             | SW              |
|                 | data to         |                 |                 |
|                 | INT8/16/FP16    |                 |                 |
|                 | representable   |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| bs_shifter      | Shifter the     | SDP.PROC_PRECIS | HW              |
|                 | input bias to   | ION=INT8/INT16  |                 |
|                 | make it addable | only            |                 |
|                 | with            |                 |                 |
|                 | convolution     |                 |                 |
|                 | pipeline        |                 |                 |
|                 | results         |                 |                 |
+-----------------+-----------------+-----------------+-----------------+

.. math:: \begin{equation}\begin{cases}
    SF_{bs}*2^{bs\_trunc}=\frac{SF_{in}*SF_{wt}}{2^{pra\_trunc+cc\_out\_trunc}},&\text{conv=winograd}\\
    SF_{bs}*2^{bs\_trunc}=\frac{SF_{in}*SF_{wt}}{2^{cc\_out\_trunc}},&\text{conv=DC}
    \end{cases}\end{equation}

..

In case of input data encoded with offset (*x’=(x-offset)*SF*), this
offset should be carefully considered for cases below:

-  Padding:

Convolution supports zero-padding, however, if the input encoded with
“offset”, it means x=0 becomes x’=(0-offset)*SF=-offset*SF thus
hardware should do “valued padding” instead of “zero padding”.
Convolution has a register named as:

PADDING_VALUE, this register should be set as –offset*SF for INT8/16
pipeline. However, for FP16, we assume there’s no offset thus
PADDING_VALUE should be set as 0;

-  Activation:

As discussed above, activation such as ReLU is a piece wise
function:

.. math:: \begin{equation}\begin{cases}
   y=x,&\text{x>0}\\
   y=0,&\text{otherwise}
   \end{cases}\end{equation}

0 plays a important role to decide activation output, unfortunately,
if “offset” is enabled on input convolution data, the “0” is no
longer 0 in the encoded activation data:

Let’s deduce the CC output (activation layer input) offset based on
convolution definition:

Given: :math:`In_{int}=SF_{in}*(In-Offset_{in}),\ Wt_{int}=SF_{wt}*Wt,\ CC_{FP}=\sum{In*Wt},`

.. math:: \begin{align*}
   CC_{int} & = \frac{\sum{In_{int}*Wt_{int}}}{2^{pra\_trunc+cc\_out\_trunc}} \\
   & = \frac{\sum{SF_{in}*(In-offset_{in})*SF_{wt}*Wt}}{2^{pra\_trunc+cc\_out\_trunc}} \\
   & = \frac{SF_{in}*SF_{wt}*\sum{(In-offset_{in})*Wt}}{2^{pra\_trunc+cc\_out\_trunc}} \\
   & = \frac{SF_{in}*SF_{wt}*(\sum{In*wt}-offset_{in}*\sum{Wt})}{2^{pra\_trunc+cc\_out\_trunc}} \\
   & = \frac{SF_{in}*SF_{wt}*(CC_{FP}-offset_{in}*\sum{Wt})}{2^{pra\_trunc+cc\_out\_trunc}}
   \end{align*}

(The truncate for activation/weight are merged to :math:`SF_{in}`, :math:`SF_{wt}` in formula above
to simplify deduction)

So, the CC output offset is: :math:`\frac{SF_{in}*SF_{wt}*offset_{in}*\Sigma{W_t}}{2^{pra\_trunc+cc\_out\_trunc}}`.

Please be noticed: The formula above is assuming no quantization
error, in practice, there’ll be quantization error on weight thus
actual offset is :math:`\frac{SF_{in}*SF_{wt}*offset_{in}*\Sigma{W^{'}_t}}{2^{pra\_trunc+cc\_out\_trunc}}`.

Where :math:`W^{'}_t` is the low precision version of weight which takes weight
quantization error into consideration.

:math:`\Sigma{W^{'}_t}` is different channel-by-channel which means :math:`\frac{SF_{in}*SF_{wt}*offset_{in}*\Sigma{W^{'}_t}}{2^{pra\_trunc+cc\_out\_trunc}}` also vary channel by
channel thus per-channel operation has to be adopted to compensate
the CC output offset. This compensation is done by ALU module in
X1/X2/Y in SDP.

SDP convertors
^^^^^^^^^^^^^^

SDP has kinds of use scenarios, table below lists how those use
scenarios maps to SDP sub-modules (For the meaning of X/Y, please refer
to :numref:`fig_image53_nvdla_precision`)

+-------------------------------+------------+
| Use scenario                  | Sub-module |
+===============================+============+
| Bias addition                 | X or Y     |
+-------------------------------+------------+
| Batch Normalization           | X or Y     |
+-------------------------------+------------+
| Element-wise                  | X or Y     |
+-------------------------------+------------+
| Activation(ReLU/PReLU)        | X or Y     |
+-------------------------------+------------+
| Activation(Sigmoid/TanH, etc) | Y          |
+-------------------------------+------------+
| Precision conversion          | X or Y     |
+-------------------------------+------------+

Let’s review those cases one by one:

.. bias-addition-1:

Bias addition
'''''''''''''

This already covered by `Convolution convertors`_

.. batch-normalization-1:

Batch normalization
'''''''''''''''''''

Here’s a list of convertor/shifters needed to realize batch
normalization function in SDP:

+-----------------+-----------------+-----------------+-----------------+
| Convertor       | Functionality   | Limitation      | Owner           |
+=================+=================+=================+=================+
| bn_m_cvt        | Convert the     | N/A             | SW              |
|                 | offline trained |                 |                 |
|                 | batch           |                 |                 |
|                 | normalization   |                 |                 |
|                 | mean data to    |                 |                 |
|                 | INT8/16/FP16    |                 |                 |
|                 | representable   |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| bn_m_shifter    | Shift the       | For             | HW              |
|                 | bn_m_cvt        | SDP.PROC_PRECIS |                 |
|                 | converted       | ION=INT8/INT16  |                 |
|                 | values to have  | only            |                 |
|                 | the same        |                 |                 |
|                 | scaling factor  |                 |                 |
|                 | as input        |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| bn_v_cvt        | Convert the     | Offset is not   | SW              |
|                 | offline trained | allowed         |                 |
|                 | batch           |                 |                 |
|                 | normalization   |                 |                 |
|                 | 1/variance to   |                 |                 |
|                 | INT8/16/FP16    |                 |                 |
|                 | representable   |                 |                 |
+-----------------+-----------------+-----------------+-----------------+

The input of batch normalization should be either from CONV/MC or
previous pipeline stages thus we should assume :math:`O_{in}, SF_{in}` are applied on input.

In order to make mean addable with input data, formula below should be
satisfied:

.. math:: SF_{in} = SF_{bs\_m\_cvt} * 2^{bn\_m\_shifter}

Element wise
''''''''''''

Here’s a list of convertor/shifters needed to related to element wise
operation in SDP:

+-----------------+-----------------+-----------------+-----------------+
| Convertor       | Functionality   | Limitation      | Owner           |
+=================+=================+=================+=================+
| ew_cvt          | The convertor   | For             | HW              |
|                 | applied on      | SDP.PROC_PRECIS |                 |
|                 | element-wise    | ION=INT8/INT16  |                 |
|                 | input, as       | only            |                 |
|                 | element-wise    |                 |                 |
|                 | are cube-based, |                 |                 |
|                 | the             |                 |                 |
|                 | element-wise    |                 |                 |
|                 | hardware layer  |                 |                 |
|                 | are the output  |                 |                 |
|                 | of upstream     |                 |                 |
|                 | hardware layers |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| ew_inv_cvt      | Align the       | For             | HW              |
|                 | offset/scaling  | SDP.PROC_PRECIS |                 |
|                 | factors to meet | ION=INT8/INT16  |                 |
|                 | the requirement | only            |                 |
|                 | of different    |                 |                 |
|                 | element wise    |                 |                 |
|                 | operation(see   |                 |                 |
|                 | below). If the  |                 |                 |
|                 | requirement     |                 |                 |
|                 | already         |                 |                 |
|                 | satisfied, this |                 |                 |
|                 | convertor can   |                 |                 |
|                 | be bypassed.    |                 |                 |
+-----------------+-----------------+-----------------+-----------------+

Since there might be 2 convertors applied on E-RDMA stream, if original
input is x, the output from ew_inv_cvt is:

.. math:: x'=\{(x-O_{ew\_cvt})*SF_{ew\_cvt}-O_{ew\_inv\_cvt}\}*SF_{ew\_inv\_cvt}=\{x-(O_{ew\_cvt}+\frac{O_{ew\_inv\_cvt}}{SF_{ew\_cvt}})\}*SF_{ew\_cvt}*SF_{ew\_inv\_cvt}

In order to make element-wise acts as we supposed, the convertor
parameter should be carefully configured based on different element-wise
operation (Assume convertor parameter from BN module is: :math:`O_{in}, SF_{in}`):

-  MAX

   The offset/scaling applied on input stream and E-RDMA stream should
   be the same, which means:

.. math:: O_{in}==O_{ew\_cvt}+\frac{O_{ew\_inv\_cvt}}{SF_{ew\_cvt}}

.. math:: SF_{in}==SF_{ew\_cvt}*SF_{ew\_inv\_cvt}

-  SUM

   The scaling factor applied on both stream should be the same:

.. math:: SF_{in} == SF_{ew\_cvt} * SF_{ew\_inv\_cvt}

-  PROD

   The offset applied on E-RDMA stream should be 0:

.. math:: O_{ew\_cvt} + \frac{O_{ew\_inv\_cvt}}{SF_{ew\_cvt}} == 0

Activation (ReLU/PReLU)
'''''''''''''''''''''''
The input offset of ReLU, PReLU already
eliminated in ALU unit of X1/X2/Y thus the 0s in ReLU/PReLU is real
“0”, so, we don’t need to worry modules;

Activation (Sigmoid/TanH, etc.)
'''''''''''''''''''''''''''''''
If complex activation function (e.g.: sigmoid or TanH) are used, LUT
has to be used to mimic the curve of those functions. The LUT
coverage has to be precisely matched with the input convertor
parameter to make it acts as you want.

Let’s use an example to explain this match process: suppose [100,
300] is the most interesting data range, user will program LUT
(suppose we have 257 LUT entries) as:

LUT[0]=f(100),

LUT[1]=f(100+200/256)

…

LUT[256]=f(300)

This means, if you want to get the correct LUT output, the LUT input
has to be :math:`x^{'} = (x - O) * SF`, where O=100, SF=200/256

So, software has to carefully program the convertors before LUT to
achieve this.

Precision conversion
''''''''''''''''''''
SDP supports various format conversions, when
conversion from high precision to low (e.g.: INT16->INT8,
FP16->INT16/8), a convertor is suggested to avoid the interested
data range be rounding/saturated.

The conversion can be done by any of the convertors in SDP pipeline
(except ew_inv_cvt).

CDP convertors
^^^^^^^^^^^^^^

CDP has convertors listed below:

+-----------------+-----------------+------------------+-----------------+
| Convertor       | Functionality   | Limitation       | Owner           |
+=================+=================+==================+=================+
| cdp_in_cvt      | Convert the     | For              | HW              |
|                 | input data      | CDP.INPUT_DATA_T |                 |
|                 | compatible with | YPE=INT8/INT16   |                 |
|                 | LUT             | only             |                 |
|                 | requirement,    |                  |                 |
|                 | which means,    |                  |                 |
|                 | the output of   |                  |                 |
|                 | this convertor  |                  |                 |
|                 | should be:      |                  |                 |
|                 | x*2\ :sup:`N`   |                  |                 |
+-----------------+-----------------+------------------+-----------------+
| cdp_lut_cvt     | Each LUT entry  | No offset        | SW              |
|                 | has 16bits (can | allowed          |                 |
|                 | be interpreted  |                  |                 |
|                 | as INT16 or     |                  |                 |
|                 | FP16 based on   |                  |                 |
|                 | pipeline), the  |                  |                 |
|                 | original f(x)   |                  |                 |
|                 | has to be       |                  |                 |
|                 | converted to    |                  |                 |
|                 | specified       |                  |                 |
|                 | format to keep  |                  |                 |
|                 | a high          |                  |                 |
|                 | precision       |                  |                 |
+-----------------+-----------------+------------------+-----------------+
| cdp_out_cvt     | Convert the     | For              | HW              |
|                 | results to      | CDP.INPUT_DATA_T |                 |
|                 | INT8/16/FP16    | YPE=INT8/INT16   |                 |
|                 | before output   | only             |                 |
|                 | to external     |                  |                 |
+-----------------+-----------------+------------------+-----------------+

Suppose the CDP input has, in order to make LUT input has the form of
x*2\ :sup:`M`, cdp_in_cvt has to be programmed as:

.. math:: O_{cdp\_in\_cvt} = -O_{in} * SF_{in}

.. math:: SF_{cdp\_in\_cvt} = \frac{2^M}{SF_{in}}

Value M should be selected by precision study.

Suppose CDP output is encoded as :math:`O_{out}, SF_{out}`, cdp_lut_cvt and cdp_out_cvt has to be
programmed as:

.. math:: O_{out} == \frac{O_{cdp\_out\_cvt}}{SF_{cdp\_lut\_cvt} * 2^M}

.. math:: SF_{out} == SF_{cdp\_lut\_cvt} * SDP_{cdp\_out\_cvt} * 2^M

PDP convertors
^^^^^^^^^^^^^^

There’s no convertor instanced in PDP. But be noticed that the PDP
padding value is intended to compensate the input offset, for FP16 pipe,
they’re ignored as we assume there’s no offset for FP16 pipe;

Convertor statistics
^^^^^^^^^^^^^^^^^^^^

NVDLA implemented counters to evaluate number of samples overflowed
during convertor. The overflow is defined as:

.. math:: INT32: x < -2147483648 || x > 2147483647
.. math:: INT16: x < -32768 || x > 32767
.. math:: INT8: x < -128 || x > 127
.. math:: FP16: fabs(x) >= 65504

Here’s a list of saturation counters in NVDLA pipeline:

+---------------------------+--------------------------------------+
| Register                  | Valid condition                      |
+===========================+======================================+
| CACC. D_OUT_SATURATION    | Always enabled                       |
+---------------------------+--------------------------------------+
| SDP.D_PERF_OUT_SATURATION | PERF_SAT_EN=YES &&                   |
|                           |                                      |
|                           | PROC_PRECISION== OUT_PRECISION==FP16 |
+---------------------------+--------------------------------------+
| CDP.D_OUT_SATURATION      | Always enabled                       |
+---------------------------+--------------------------------------+

LUT programming
~~~~~~~~~~~~~~~

LUT are instanced in SDP/CDP in NVDLA, it’s used to mimic the non-linear
functions (Sigmoid/TanH/LRN, etc.) of a network. As we know, the LUT
precision is highly depends on LUT entries and slope variation of the
curve: The more LUT entries, the higher precision. On the other hand,
the strong slope variation of the curve, the hard to mimic.

It’s worth to mention SDP/CDP shares the same LUT logic, the only
difference between them are the bit-depth as SDP pipeline is 32bits but
CDP pipeline is 37bits.

We proposed an innovated 2 level hybrid LUT architecture to keep very
high precision by limited LUT entries:

There’re 2 highlights of this implementation:

-  2 level:

.. _fig_image85_lut_architecture:

.. figure:: ias_image85_lut_architecture.svg
  :align: center

  LUT architecture

There’re 2 tables (X/Y table), the typical configuration is use one of
them as raw table to cover entire dynamic range and the other work as
density table to cover a small portion of the dynamic range. Due to the
coverage difference, raw table has low sample rate but density table has
relative high sample rate, this is inspired by the attribute of
LRN/Sigmoid/TanH curve:

.. _fig_image86_non_linear_lrn:

.. figure:: ias_image86_non_linear_lrn.png
  :align: center

  Non-linear function: LRN

.. _fig_image87_non_linear:

.. figure:: ias_image87_non_linear_sigmoid.svg
  :align: center

  Non-linear function: Sigmoid

We can see from figures above, for those functions, only a small portion
has significant slope variation and the others portion almost without
too much change thus 2 level LUT is an economy option to mimic those
functions.

As there might be overlap between density/raw table, we have a
programmable register: “priority” (pri) to allow software control which
LUT table output should be taken as final output when one sample fits to
both tables. Of course, the suggestion is to use density output all the
time.

-  Hybrid working mode

We noticed for LRN, the input dynamic range is very high (0~10^8), but
most of the samples are within a small data range:

.. _fig_image88_histogram_of_lrn_lut_input:

.. figure:: ias_image88_histogram_of_lrn_lut_input.png
  :align: center

  Histogram of LRN LUT input

Histogram above are collected from “pool1/norm1” layer of GoogleNet, we
viewed the same data by different x-axis coordinate system (linear and
exponential).

We can see the linear view merges >50% samples to one point in histogram
while exponential view distinguishes those samples to different
histogram points which gives much better resolution. Same strategy can
be adopted to LUT: If the LUT is working on exponential mode, we have a
very high sample rate on low range values and low sample rate on high
range values (it’s fair since they have low frequency in histogram).
This is the idea of “exponential” mode. Currently, only X table is able
to work on exponential mode and when this mode is enabled, the coverage
is fixed as :math:`2^{exp\_start} ~ 2^{exp\_start + tlb\_entry}`

Table below summarized the LUT attributes of NVDLA (The X/Y are used to
denote different table, when mapping to hardware, X corresponding to
LE(Linear/Exponent) while Y corresponding to LO(Linear only)):

+---------------------------+----------------------+
| Attributes                | Description          |
+===========================+======================+
| X table entries           | 65                   |
+---------------------------+----------------------+
| Y table entries           | 257                  |
+---------------------------+----------------------+
| Bits per entry            | 16                   |
+---------------------------+----------------------+
| X supported working modes | Exponential/Linear   |
+---------------------------+----------------------+
| Y supported working modes | Linear               |
+---------------------------+----------------------+
| Interpolation methods     | Linear interpolation |
+---------------------------+----------------------+
| Out-of-range behavior     | Linear interpolation |
+---------------------------+----------------------+

Recommended LUT configuration for typical use scenarios:

+-----------------------------------+-----------------------------------+
| Use scenario                      | Configuration                     |
+===================================+===================================+
| LRN                               | X – Exponential mode (worked as   |
|                                   | raw table)                        |
|                                   |                                   |
|                                   | Y – Linear mode (worked as        |
|                                   | density table)                    |
+-----------------------------------+-----------------------------------+
| Activation (Sigmoid/TanH, etc.)   | X – Linear mode (worked as        |
|                                   | density table)                    |
|                                   |                                   |
|                                   | Y – Linear mode (worked as raw    |
|                                   | table)                            |
+-----------------------------------+-----------------------------------+

However, this is not mandatory, software can program LUT work as any of
cases below (X/Y can be reversed, which means totally 6 cases):

.. _fig_image90_lut_coverage:

.. figure:: ias_image90_lut_coverage.svg
  :align: center

  LUT coverage

As shown in :numref:`fig_image85_lut_architecture`, there’re couple parameters for LUT, let’s
discuss how to configure them based on different modes.

Exponential mode
^^^^^^^^^^^^^^^^

If LUT is working on exponential mode and LUT storage has example below
(actually, exp_start is programmable):

+----------------------------------------------------+
| exp_start=-32                                      |
|                                                    |
| LUT[0]=f(2\ :sup:`exp_start`)=f(2\ :sup:`-32`)     |
|                                                    |
| LUT[1]=f(2\ :sup:`exp_start+1`)=f(2\ :sup:`-31`)   |
|                                                    |
| …                                                  |
|                                                    |
| LUT[64]= f(2\ :sup:`exp_start+64`)=f(2\ :sup:`32`) |
+----------------------------------------------------+

Suppose LUT input is: :math:`x' = (x - O_{in}) * SF_{in}`, per LUT storage, the index should be:

.. math:: index = (log_2(x - linear\_start)) - exp\_start = log_2 (\frac{x'}{SF_{in}} + O_{in} - linear\_start) - exp\_start

SF\ :sub:`in` must be 2\ :sup:`M`, then, formula above changed as:
.. math:: index = log_2 (\frac{x'}{SF_{in}} + O_{in} - linear\_start) - exp\_start = log_2 (x' - SF_{in}(linear\_start - O_{in})) - (M + exp\_start)

*exp_start* is a value related to LUT storage (in our example, we use
-32 here) while *M* is related to upstream convertor setting.

The mapping between register and symbols above are lists below (only X
table supports exponential mode):

+-------------------------+---------------------------------------------------+
| Register                | Symbol                                            |
+=========================+===================================================+
| LE_INDEX_OFFSET         | M+exp_start                                       |
+-------------------------+---------------------------------------------------+
| LUT_LE_INDEX_SELECT     | Not used                                          |
+-------------------------+---------------------------------------------------+
| S_LUT_LE_START_LOW/HIGH | :math:`SF_{in}(linear\_start - O_{in})`           |
+-------------------------+---------------------------------------------------+

Linear mode
^^^^^^^^^^^

If LUT is working on linear mode and LUT is supposed to cover min~max,
then, LUT entry storage should be (suppose entry_num=257):

+-----------------------------------+
| step=(max-min)/(entry_num-1)      |
|                                   |
| LUT[0]=f(0*step + min)            |
|                                   |
| LUT[1]=f(1*step + min)            |
|                                   |
| …                                 |
|                                   |
| LUT[256]=f(256*step + min)=f(max) |
+-----------------------------------+

Suppose LUT input is: :math:`x' = (x - O_{in}) * SF_{in}`, per LUT storage, the index should be:

.. math:: index=\frac{x-min}{step}=\frac{\frac{x'}{SF_{in}}+O_{in}-min}{step}=\frac{x'+O{in}*SF_{in}-min*SF_{in}}{(max-min)*SF_{in}}*(entry\_num-1)

Denote:

.. math:: SF_{lut}=\frac{entry_num-1}{(max-min)*SF_{in}},\ O_{lut}=min*SF_{in}-O_{in}*SF_{in},\ index=(x'-O_{lut})*SF_{lut}

This requires multiplier, in order to make hardware simpler, we require:

.. math:: SF_{lut} = 2^M

Then, hardware just need to right/left shifter to get correct index,
however, this implies:

.. math:: (max - min) * SF_{in} = 2^{C-M}, where\ C=log_2(entry\_num - 1)

This can be guaranteed by the convertor before LUT (cdp_in_cvt in CDP;
X/X/Y multiplier in SDP).

The mapping between symbols above and the actual registers are (X could
be LE/LO):

+------------------------+-------------------------+
| Register               | Symbol                  |
+========================+=========================+
| LUT_LE/LO_INDEX_SELECT | -M                      |
+------------------------+-------------------------+
| X_START_LOW/HIGH       | :math:`O_{lut}`         |
+------------------------+-------------------------+
| LE_INDEX_OFFSET        | Not used                |
+------------------------+-------------------------+

Out-of-range control
^^^^^^^^^^^^^^^^^^^^

Suppose one LUT has coverage between [min, max]. If one input sample
bigger than max or smaller than min, we call it out-of-range sample.

NVDLA supports linear interpolation of those out-of-range samples. The
mathematic formula for the interpolation is (x is the input sample
value):

.. math:: \begin{equation}\begin{cases}
   y_0+(x-min)*k_{underflow},&\text{x<min}\\
   y_n+(x-max)*k_{overflow},&\text{x>max}\\
   \end{cases}\end{equation}

From hardware perspective, the interpolation is:

.. math:: \begin{equation}\begin{cases}
   LUT[0]+(X-START)*UFLOW_{SCALE}/UFLOW_{SHIFT},&\text{X<START}\\
   LUT[N]+(X-END)*OFLOW_{SCALE}/OFLOW_{SHIFT},&\text{X>END}\\
   \end{cases}\end{equation}

Take underflow as an example, given (:math:`2^M` is the scaling applied on LUT
input, SF is the scaling applied on LUT entries):

.. math:: X = x * 2^M
.. math:: START = min * 2^M
.. math:: LUT[0] = y_0 * SF

Hardware output:

.. math:: \begin{align*}
   LUT[0] + (X-START)*UFLOW\_SCALE/UFLOW\_SHIFT & = y_0 * SF + (x-min) * 2^M * UFLOW\_SCALE/UFLOW\_SHIFT \\
   & = SF*(y_0 + \frac{(x-min) * 2^M * UFLOW\_SCALE/UFLOW\_SHIFT}{SF})
   \end{align*}

Thus:

.. math:: \frac{2^M * UFLOW\_SCALE/UFLOW\_SHIFT}{SF} = k_{underflow}

The mapping between the symbols in above formula and registers are:

+----------------------+--------------------------------+----------------------+
| Register             | Symbol                         | NOTE                 |
+======================+================================+======================+
| LUT[0]               | :math:`y_0`                    | No register,         |
|                      |                                | directly use LUT     |
|                      |                                | content              |
+----------------------+--------------------------------+----------------------+
| LUT[N], (N is the    | :math:`y_n`                    | No register,         |
| last entry of the    |                                | directly use LUT     |
| LUT)                 |                                | content              |
+----------------------+--------------------------------+----------------------+
| X_START_LOW/HIGH     | Min*2\ :sup:`M`                | Same bits/encoding   |
|                      |                                | as the pipeline.     |
|                      |                                | (i.e.: for INT, it   |
|                      |                                | will be treat as INT |
|                      |                                | and for FP16, it     |
|                      |                                | will be treated as   |
|                      |                                | FP16)                |
+----------------------+--------------------------------+----------------------+
| X_END_LOW/HIGH       | Max*2\ :sup:`M`                | Same bits/encoding   |
|                      |                                | as the pipeline.     |
+----------------------+--------------------------------+----------------------+
| SLOPE_UNDERFLOW_SCAL | :math:`SF*k_{underflow}*2^{-M}`| 16bits for SCALE     |
| E                    |                                | will be treated as   |
|                      |                                | INT/FP16 for         |
| SLOPE_UNDERFLOW_SHIF |                                | INT/FP16 pipeline    |
| T                    |                                | respectively;        |
|                      |                                |                      |
|                      |                                | SHIFT is 5 bit       |
|                      |                                | signed int, won’t be |
|                      |                                | used for FP16 pipe;  |
|                      |                                |                      |
|                      |                                | (if shift > 0,       |
|                      |                                | k=SCALE>>SHIFT;      |
|                      |                                | otherwise,           |
|                      |                                | k=SCALE<<SHIFT)      |
+----------------------+--------------------------------+----------------------+
| SLOPE_OVERFLOW_SCALE | :math:`SF*k_{overflow}*2^{-M}` | Same as UNDERFLOW    |
|                      |                                |                      |
| SLOPE_OVERFLOW_SHIFT |                                |                      |
+----------------------+--------------------------------+----------------------+

LUT storage programming
^^^^^^^^^^^^^^^^^^^^^^^

Traditionally, in order to program an LUT entry, you have to specify
both LUT entry address and its value, this requires 2 register write
operation. NVDLA simplifies this process by introducing hardware
automatic address incremental mechanism, which means, when you need to
program an LUT table, you just have to write your code as below (take LE
table program for example):

.. code:: c

  /\* program raw table \*/                                    
  reg = (FIELD_ENUM(S_LUT_ACCESS_CFG, LUT_TABLE_ID, LE)        
        << SHIFT(S_LUT_ACCESS_CFG, LUT_TABLE_ID)) \|           
        (FIELD_ENUM(S_LUT_ACCESS_CFG, LUT_ACCESS_TYPE, WRITE)  
        << SHIFT(S_LUT_ACCESS_CFG, LUT_ACCESS_TYPE));          
  reg_write(S_LUT_ACCESS_CFG, reg);                            
  for(i = 0; i < LUT_LE_TABLE_ENTRIES; i\+\+) {                
      reg_write(S_LUT_ACCESS_DATA, lut->le_table[i]);          
  }                                                            
                                                               

If the address beyond the total LUT entry (e.g.: The
LUT_RAW_TABLE_ENTRIES in pseudo code above exceed the actual LUT entry),
the hardware behavior is undefined.

NVDLA supports read back the programmed LUT entries from arbitrary
entry. The S_LUT_ACCESS_CFG just need program once then the address will
increase automatically. **Please be noticed that programming of
S_LUT_ACCESS_CFG has to be non-post write for LUT read case;**

There’re 2 constrains for LUT programming:

-  Make sure always write LUT from first entry and update entire table;

-  There’s only one LUT storage shared for both register groups, make
   sure update LUT are happened when corresponding sub-unit is IDLE;

Hit/Miss behavior
^^^^^^^^^^^^^^^^^

For a given input sample, if only one table is hit, the final output
will be the output of hit table; However, the X/Y table programming is
so flexible then leads to different hit/miss cases:

.. _fig_image109_lut_hit_miss:

.. figure:: ias_image109_lut_hit_miss.png
  :align: center

a) One input sample might be hit in both table; (Case 1)

b) One input sample might miss in both table due to overflow; (Case1, 2,
   3)

c) One input sample might miss in both table due to underflow; (Case 1,
   2, 3)

d) One input sample might miss in both table due to one table overflow
   while the other underflow (Case 3)

For all the cases above, hardware need a way to choose how to get
the final output thus we expose programmable registers below to
allow software program the priority:

+-----------------------------------+-----------------------------------+
| Register Name                     | Description                       |
+===================================+===================================+
| Priority                          | One bit register to indicate      |
|                                   | which table output should be      |
|                                   | selected as the final output when |
|                                   | both hit or hybrid miss happens   |
|                                   | (case a, d);                      |
|                                   |                                   |
|                                   | 0 means X table is selected;      |
|                                   |                                   |
|                                   | 1 means Y table is selected;      |
+-----------------------------------+-----------------------------------+
| OverflowPriority                  | One bit register to indicate      |
|                                   | which table output should be      |
|                                   | selected as final output when     |
|                                   | overflow for both table happens.  |
|                                   | (case b)                          |
|                                   |                                   |
|                                   | 0 means X table is selected;      |
|                                   |                                   |
|                                   | 1 means Y table is selected;      |
+-----------------------------------+-----------------------------------+
| UnderflowPriority                 | One bit register to indicate      |
|                                   | which table output should be      |
|                                   | selected as final output when     |
|                                   | underflow for both table happens  |
|                                   | (case c)                          |
|                                   |                                   |
|                                   | 0 means X table is selected;      |
|                                   |                                   |
|                                   | 1 means Y table is selected;      |
+-----------------------------------+-----------------------------------+

LUT Statistics
^^^^^^^^^^^^^^

When one hardware layer completes, hardware will report statistics below
to help software understand whether the LUT table is reasonably
programmed.

+-----------------------------------+-----------------------------------+
| Statistic register                | Description                       |
+===================================+===================================+
| XHitNum                           | Number of samples hit on X only   |
+-----------------------------------+-----------------------------------+
| YHitNum                           | Number of samples hit on Y only   |
+-----------------------------------+-----------------------------------+
| UnderflowNum                      | Number of samples underflow for   |
|                                   | both X and Y table                |
+-----------------------------------+-----------------------------------+
| OverflowNum                       | Number of samples overflow for    |
|                                   | both X and Y table                |
+-----------------------------------+-----------------------------------+
| PriorityNum                       | Number of samples both hit on X/Y |
|                                   | table or Hybrid miss on X/Y table |
|                                   | (Actually, this counter has 2     |
|                                   | different meanings which          |
|                                   | corresponding to a and d case in  |
|                                   | above section, but since they’re  |
|                                   | mutual exclusive, we just use one |
|                                   | register for them. Software is    |
|                                   | able to distinguish the different |
|                                   | meanings since it knows each LUT  |
|                                   | coverage)                         |
+-----------------------------------+-----------------------------------+

For each register group, we have dedicated statistic registers above,
those counters will be available for read when one hardware completes
(by set the producer pointer to this register group). Those statistics
won’t be erased until the corresponding register group is enabled (op_en
be set)

BDMA programming
----------------

Background
~~~~~~~~~~

We suggest NVDLA memory accesses are based on internal SRAM to achieve best
performance and we designed BDMA for this purpose.

The supported memory transfers are:

+---------------+------------------+
| Source type   | Destination type |
+===============+==================+
| External DRAM | Internal SRAM    |
+---------------+------------------+
| External DRAM | External DRAM    |
+---------------+------------------+
| Internal SRAM | External DRAM    |
+---------------+------------------+
| Internal SRAM | Internal SRAM    |
+---------------+------------------+

Programming
~~~~~~~~~~~

The programming model for BDMA is different from others due to special
use scenario on BDMA. Take convolution as an example, in order to make a
convolution layer operation happen, BDMA has to transfer
input_feature/weight/mean/bias into internal SRAM. If BDMA also uses the
traditional programming model, CPU will act as:

Issue setting for single transfer of input feature

Wait for interrupt

Issue setting for single transfer for weight

Wait for interrupt

…

The total time is:

4(Weight/Image/Mean/Bias) \* CPU_ISR_Time + 4*TransactionTime;

This process is boring and many interactions between CPU and BDMA are
needed. In order to improve the efficiency, a new programming model for
BDMA is listed as below:

1. CPU issue setting for single transfer of input feature (set interrupt
   flag as false)

2. Pooling BDMA if there’s empty slot for program (BDMA support 20
   register entries thus most of time, polling always return true)

3. CPU issue setting for single transfer of weight (set interrupt flag
   as false, if it’s not the last transfer request)

4. Repeat 2~3 until all data transfer request are done and set interrupt
   flag as true for last request

5. Wait for interrupt

The total time for outstanding based programming model is:

1*CPU_ISR_Time + 4*Transaction_Time;

We introduce 2 terminologies to describe procedure above:

-  Operation: Each individual BDMA transaction is called as operation.
   One operation may or may not trigger interrupt depending on software
   setting. take example above, transfer of activation, weight, mean,
   bias are 4 different BDMA operation.

-  Group: group is consisted by one or more BDMA operations depending on
   software configuration. Set GRP<0|1>_LAUNCH as YES is treated as end
   of a group.

During one BDMA group register programming, hardware acts as:

-  Software program one BDMA operation then set the EN bit

-  Hardware “cache” the corresponding BDMA registers to its internal
   slot, no actual memory transaction carried out. There’re totally 20
   slots thus we can support 20 BDMA operations in one group as maximum;

-  Software poll the free slots by read STATUS.FREE_SLOTS, if it’s
   bigger than 0, it means software is allowed to program the next BDMA
   operation;

-  For the last BDMA operation in one group, software has to set
   CFG_LAUNCH<0|1>.GRP<0|1>_LAUNCH = YES;

-  Hardware will actually kick of all the “cached” BDMA operations in
   this group (by detect INTERRUPT=YES).

-  After all BDMA operation done, corresponding interrupt will be
   generated.

For below section, if there’s no special declaration, all address refers
to data address in SRAM.

Buffer allocation
~~~~~~~~~~~~~~~~~

Before introduce buffer allocation formula, we need to understand the
related register definition:

+-----------------------------------+-----------------------------------+
| Register                          | Description                       |
+===================================+===================================+
| CFG_LINE                          | Indicate the valid data size per  |
|                                   | line. This register should be     |
|                                   | configured as:                    |
|                                   |                                   |
|                                   | valid_bytes_per_line/32-1         |
+-----------------------------------+-----------------------------------+
| CFG_LINE_REPEAT                   | Number of lines per surface. This |
|                                   | register should be configured as: |
|                                   |                                   |
|                                   | surface_height-1                  |
+-----------------------------------+-----------------------------------+
| CFG_SRC/DST_LINE                  | Number of bytes per src/dst line  |
|                                   | (padding are included). It should |
|                                   | be configured as:                 |
|                                   |                                   |
|                                   | total_bytes_per_line              |
+-----------------------------------+-----------------------------------+
| CFG_SURF_REPEAT                   | Number of surfaces in one data    |
|                                   | cube                              |
+-----------------------------------+-----------------------------------+
| CFG_SRC/DST_SURF                  | Number of bytes per surface (line |
|                                   | padding are included). It should  |
|                                   | be configured as:                 |
|                                   |                                   |
|                                   | total_bytes_per_surface           |
+-----------------------------------+-----------------------------------+

Given the register definition above, the formula for buffer allocation
are:

.. math:: src\_cube\_size = CFG\_SRC\_SURF * CFG\_SRC\_REPEAT
.. math:: dst\_cube\_size = CFG\_DST\_SURF * CFG\_DST\_REPEAT

The formula for actual bytes transferred is:
.. math:: actual\_size = (CFG\_LINE - 1) * 32 * CFG\_LINE\_REPEAT * CFG\_SURF\_REPEAT

Rubik programming
-----------------

Features
~~~~~~~~

+-----------------------------------+-----------------------------------+
| Mode                              | Description                       |
+===================================+===================================+
| Contract                          | Worked as final phase of          |
|                                   | deconvolution to reorder the      |
|                                   | output layout;                    |
+-----------------------------------+-----------------------------------+
| Split                             | Convert the feature format to     |
|                                   | M-planar format                   |
+-----------------------------------+-----------------------------------+
| Merge                             | Convert the M-planar format to    |
|                                   | feature format.                   |
+-----------------------------------+-----------------------------------+

.. programming-1:

Programming
~~~~~~~~~~~

.. contract-1:

Contract
^^^^^^^^

1) Config the RUBIK_MODE= CONTRACT

2) Configure the input cube information:

   D_DAIN_RAM_TYPE: The input memory type;
   
   D_DATAIN_SIZE_0/1: The input W/H/C;
   
   D_DAIN_ADDR_HIGH/LOW: The input cube start address;
   
   D_DAIN_LINE/SURF_STRIDE: The input cube line/surface stride;

3) Configure the output cube information:

+-----------------------------------+-----------------------------------+
| Register                          | Value                             |
+===================================+===================================+
| D_DATAOUT_SIZE_1                  | (DATAIN_CHANNEL+1)/((             |
|                                   | DECONV_X_STRIDE+1)*(              |
|                                   | DECONV_Y_STRIDE+1))-1             |
+-----------------------------------+-----------------------------------+
| D_DAOUT_ADDR_HIGH/LOW             | The output cube start address     |
+-----------------------------------+-----------------------------------+
| D_DAOUT_LINE/SURFACE_STRIDE       | The output cube line/surface      |
|                                   | stride                            |
+-----------------------------------+-----------------------------------+
| D_CONTRACT_STRIDE_0               | Ceil((DATAOUT_CHANNEL+1) \* BPE / |
|                                   | 32) \* DAIN_SURF_STRIDE           |
+-----------------------------------+-----------------------------------+
| D_CONTRACT_STRIDE_1               | (DECONV_Y_STRIDE+1) \*            |
|                                   | DAOUT_LINE_STRIDE                 |
+-----------------------------------+-----------------------------------+

4) Configure the stride information:

   D_DECONV_STRIDE: The x/y stride relationship between input/output
   cube. It’s not necessary to configure those values the same as
   deconvolution stride.

5) Configure the op_en to kick-off the hardware layer;

Split/Merge
^^^^^^^^^^^

Most of the configurations are the same as Contract mode except:

1) RUBIK_MODE should be SPLIT/MERGE;

2) D_DAIN_PLANAR_STRIDE has to be configured for merge mode;

3) Registers below are not necessary to program for split mode:

   D_CONTRACT_STRIDE_0/1

   D_DAIN_PLANAR_STRIDE

   D_DAOUT_SURF_STRIDE

   D_DECONV_STRIDE

4) Registers below are not necessary to program for merge mode:

   D_CONTRACT_STRIDE_0/1

   D_DAIN_SURF_STRIDE

   D_DAOUT_PLANAR_STRIDE

   D_DECONV_STRIDE

For split mode, DATAOUT_CHANNEL is used to specify number of channels
needs to split thus it equals to output planar number.

Convolution pipeline programming
--------------------------------

.. features-1:

Features
~~~~~~~~

From algorithm wise, convolution pipeline in NVDLA supports algorithm
features below:

.. table:: List of algorithm features supported by convolution pipeline
 :name: tab_algorithm_features_cc

 +-----------------------------------+-----------------------------------+
 | Feature                           | Description                       |
 +===================================+===================================+
 | Convolution                       | Convolution layer functionality.  |
 |                                   | It supports image input and       |
 |                                   | feature input                     |
 +-----------------------------------+-----------------------------------+
 | Deconvolution                     | Deconvolution layer               |
 |                                   | functionality; It supports        |
 |                                   | feature input only.               |
 |                                   | (Actually, deconvolution is a     |
 |                                   | NVDLA software feature instead of |
 |                                   | hardware)                         |
 +-----------------------------------+-----------------------------------+
 | Dilation                          | A technology to expand kernel     |
 |                                   | coverage without introduce more   |
 |                                   | network parameters.               |
 +-----------------------------------+-----------------------------------+
 | Padding                           | Padding size on the               |
 |                                   | left/right/top/bottom of input    |
 |                                   | data cube                         |
 +-----------------------------------+-----------------------------------+
 | conv_stride                       | The number of input element       |
 |                                   | should be skipped in x/y          |
 |                                   | direction after one output        |
 |                                   | element be calculated             |
 +-----------------------------------+-----------------------------------+

From performance wise, convolution pipeline implements features below to
accelerate convolution process:

.. table:: List of performance features supported by convolution pipeline
 :name: tab_performance_features_cc

 +-----------------------------------+-----------------------------------+
 | Feature                           | Description                       |
 +===================================+===================================+
 | Winograd                          | A fast convolution method (2.25x  |
 |                                   | throughput than direct            |
 |                                   | convolution), NVDLA support       |
 |                                   | equivalent kernel size = 3x3 only |
 |                                   | (equivalent means kernel after    |
 |                                   | channel extension)                |
 +-----------------------------------+-----------------------------------+
 | Channel Post-extension            | A method to improve MAC           |
 |                                   | efficiency when channel size is   |
 |                                   | too small (For image input only). |
 +-----------------------------------+-----------------------------------+
 | Multi-Batch mode                  | A method to improve MAC           |
 |                                   | efficiency when atomic number in  |
 |                                   | one stripe operation is too small |
 |                                   | (e.g.: InnerProduct layer).       |
 +-----------------------------------+-----------------------------------+
 | Weight compression                | A method to save weight data      |
 |                                   | loading bandwidth.                |
 +-----------------------------------+-----------------------------------+

Besides hardware features, different working modes will impact
performance as well:

.. table:: List of working modes supported by convolution pipeline
 :name: tab_working_modes_cc

 +-----------------------------------+-----------------------------------+
 | Working mode                      | Description                       |
 +===================================+===================================+
 | Full input & weight               | If both weight/feature can be     |
 |                                   | fitted to CONV_BUF, this mode     |
 |                                   | delivers best performance         |
 +-----------------------------------+-----------------------------------+
 | Full input, partial weight        | If feature can be fitted to       |
 |                                   | CONV_BUF while only part of       |
 |                                   | weight can be fitted to CONV_BUF  |
 |                                   |                                   |
 |                                   | Comparing with full feature &     |
 |                                   | weight, it has the same           |
 |                                   | performance for single hardware   |
 |                                   | layer, but weight can’t be        |
 |                                   | re-used.                          |
 +-----------------------------------+-----------------------------------+
 | Split H                           | A software feature which utilize  |
 |                                   | multiple HWLs to process an input |
 |                                   | data cube. It will be used when   |
 |                                   | above cases are failed to match.  |
 +-----------------------------------+-----------------------------------+

Here’s the detailed explanation about those working modes:

-  \ **Full input & weight mode**

Condition: Both input feature and weight cube can be fitted in CONV_BUF

Fit case: small sized input/weight data

Data refetch: No

Weight refetch: No

Output sequence: K’(32 or 16)W HK

In this mode, entire input/weight will be loaded to CONV_BUF which means
CONV_BUF should be large enough to store W*H*C+R*S*C*K data elements
thus:

.. math:: banks\_for\_data = ceil(\frac{entry\_per\_slice*H}{256})
.. math:: banks\_for\_weight = ceil(\frac{R * S * C * K * BPE}{256*128})

-  \ **Full input, partial weight mode**

Condition: Entire input feature data and part of weight data
(2*kernel_per_group) can be filled in CONV_BUF

Fit case: small sized input and small/middle sized weight data

Data refetch: No

Weight refetch: No

Output sequence: K’(32 or 16)W HK

Full input feature mode is a most common case for many networks. Because
the output sequence goes at K direction at last phase, it can be easily
connected to pooling logic without big buffering requirement. Below
formula should be satisfied when planning CONV_BUF layout:

.. math:: banks\_for\_data = ceil(\frac{entry\_per\_slice*H}{256})
.. math:: banks\_for\_weight >= ceil(\frac{R * S * C * 2 * kernel_per_group * BPE}{256*128})

The reason for 2*kernel_per_group is to keep CDMA and CMAC working at
the same time to hide kernel loading latency, however,
1*kernel_per_group also workable but the performance is reduced.

-  **Split H**

We can see only full mode is supported by convolution pipeline. If one
network layer has large input which exceed the CONV_BUF capacity,
software has to split the big input cube into smaller cubes in vertical
direction. This mechanism called “Split H mode”.

Be noticed that there must be max(R-stride_y, 0) overlapped lines between 2 consecutive
cube to make sure the convolution results are expected.

Strategy selection
~~~~~~~~~~~~~~~~~~

Convolution pipeline has different features/working modes, we should
follow the rule below to mapping the network parameter into hardware
layers:

1. Decide the algorithm features (:numref:`tab_algorithm_features_cc`) from network definition;

2. Select the hardware performance optimization features (:numref:`tab_performance_features_cc`):

a) If this is the first layer (image input) and any item in :numref:`tab_limits_of_channel_post_extension`
is satisfied, channel post extension should be used.

b) If this is the feature input and *ceil(R/stride_y) == 3 &&
ceil(S/stride_x) == 3* is true, winograd mode should be used;

c) If this is inner product layer and CONV_BUF is big enough to maintain
BATCH_NUMBER input cubes, multi-batch mode should be chosen. “Big
enough” here means:

.. math:: ceil(BATCH\_NUMBER * entry\_per\_slice * H / 256) <= BANKS\_FOR\_DATA

d) If *(compressed_weight_size+wmb_size+wgs_size) < weight_size* and
there’s no conflict with :numref:`tab_weight_formats`, weight compress should be used;

3. Decide the working modes by comparing actual data/weight size with
available CONV_BUF banks. The priority is: “Full weight&input” > “Full
input & Partial weight” > “Split H”. When split H mode used, it’s better
split H into smaller one to make sure weight are all kept in CONV_BUF
thus weight can be re-used.

.. programming-2:

Programming
~~~~~~~~~~~

Register definition
^^^^^^^^^^^^^^^^^^^

Before introduce the convolution pipeline programming, it’s necessary to
explain the meaning of the registers and how they’re calculated.

CC has 5 pipelines, each pipeline stage has its own registers. For any
register, if it has the same name across pipeline stage, it means they
have the same value.

Most of the registers in those groups are straightforward thus we just
highlight the registers which might confuse people in this section:

-  *<CDMA|CSC>.WEIGHT/DATA_SKIP_RELEASE:* Indicate whether or not skip
   release of the slices in CONV_BUF. If SKIP_RELEASE=false, different
   strategy are applied on feature/weight:

   -  For feature release, software is able to control how much slices
      should be released by specify D_RELEASE;

   -  For weight release, only release all or release none is supported;

-  *<CDMA|CSC>.WEIGHT/DATA_REUSE*: Indicate whether or not re-use the
   weight/data from previous hardware-layer. If this flag is set, CDMA
   fetch will be fully(partially) skipped (depending on CDMA_HEIGHT of
   Nth layer and D_RELEASE/CSC_HEIGHT of N-1th layer: if
   CDMA_HEIGHT\ :sub:`N` <= (CSC_HEIGHT-D_RELEASE):sub:`N-1`, the
   N\ :sup:`th` CDMA fetch will be skipped).

-  CDMA.LINE_STRIDE/LINE_STRIDE_UV: Those 2 registers are used for
   PITCH_LINEAR only, the value of those registers should be larger than
   the actual data per line.

Actual data per line is different according to different input format
and pixel format, please refer to: LINE_STRIDE/LINE_STRIDE_UV about its
calculation.

Besides, the requirement of alignment in :numref:`tab_requirements_of_alignment`
should also be satisfied.

-  CDMA.PIXEL_SIGN_OVERRIDE:

This field take effect for image input only.

The override field does not directly change the sign bit of input
values. It co-works with CDMA convertor. When convertor in CDMA is
enabled, original values will be extended to int17 and then be
calculated with offset and scaling factor.

For example, if input format is R_8 and override field is UNSIGNED, the
input value 0x87 will be extended as 0x00087 and sent into convertor.
And if input format is R_8 and override field is SIGNED, the input value
0x87 will be extended as 0x1ff87 and sent into convertor.

In conclusion:

-  Sign override register field only affects INT/UINT pixel formats.

-  Sign override register field should co-work with CDMA convertor.

-  If CDMA convertor is not enabled, all values are treated as
   int8/int16/fp16, no matter how sign override is set.

-  CDMA.D_DAIN_MAP:

   -  If LINE_STRIDE equals to bytes_per_line, it means this data cube
      is “LINE_PACKED”

   -  If D_SURF_STRIDE equals to LINE_STRIDE*H, it means the data cube
      is “SURF_PACKED”

-  <CDMA|CSC>.D_BANK: Indicate number of banks allocated for
   data/weight. Please refer to: 10.1.3 about the calculation.

-  <CDMA|CSC>.D_ENTRY_PER_SLICE: Entry per slice means how many CONV_BUF
   entries a slice occupied, it’s decided by multiple factors:
   convolution mode, width, channel size, stride, etc. Please refer to:
   ENTRY_PER_SLICE for detail.

-  *CDMA.FETCH_GRAIN*: This is the threshold to trigger CDMA working:
   CDMA won’t work until the empty entries in CONV_BUF reaches
   (fetch_grain+1)*ENTRY_PER_SLICE. The values of this register is a
   trade-off of fetch efficiency and fetch delay: a large value will
   benefit fetch efficiency since CDMA have larger room when sending
   request, however, if this value is too large, CDMA will wait for a
   quite long time to wait CONV_BUF release enough entries.

For LINE_UNPACKED mode, this register will be ignored by hardware and
behaves as this register set to 0.

-  *<CDMA|CSC>.WEIGHT_BYTES*: It should be configured as:
   weight_size=R*S*C*BPE*K. Regardless of weight compress mode or
   uncompressed mode.

-  *CDMA.PIXEL_X/Y_OFFSET*: Configuration of those 2 registers is
   depending on PIXEL_MAPPING:

   -  *PITCH_LINEAR*: The address configured to D_DAIN_ADDR_HIGH/LOW_0
      should be 32bytes aligned, however, the start address of an ROI
      might not aligned to that address. Then, PIXEL_X_OFFSET is
      introduced.

+-----------------------------------------------------------------------+
| D_DAIN_ADDR_HIGH/LOW_0 = roi_address &(~0x1F); // The nearest 32bytes |
| aligned address;                                                      |
|                                                                       |
| PIXEL_X_OFFSET=(roi_address&0x1F)/bytes_per_pixel // The offset in    |
| unit of pixel                                                         |
|                                                                       |
| PIXEL_Y_OFFSET = 0; // The 32bytes aligned address and roi address    |
| should be in the same line                                            |
+-----------------------------------------------------------------------+

.. _fig_image116_pitch_linear_roi:

.. figure:: ias_image116_pitch_linear_roi.png
  :align: center

-  CSC.WEIGHT/DATAIN_SIZE_EXT: The input weight/feature cube size seen
   from CSC. SW should configure those values based on formula below:

DATAIN_SIZE_EXT: (W/H/C is the width/height/channel of input data cube)

+-----------------+-----------------+-----------------+-----------------+
| Mode            | Width           | Height          | Channel         |
+=================+=================+=================+=================+
| Winograd        | ceil((W+(PL+PR) | ceil((H+PT+PB)/ | C*stride_x*stri |
|                 | )/stride_x)     | stride_y)       | de_y            |
+-----------------+-----------------+-----------------+-----------------+
| Image input     | W               | H               | C               |
+-----------------+-----------------+-----------------+-----------------+
| Direct          | W               | H               | C               |
+-----------------+-----------------+-----------------+-----------------+

WEIGHT_SIZE_EXT (S/R/C is the width/height/channel of input weight cube
and let C’ be 32bytes aligned version of C, which means: C’=ceil(C, 16)
for INT/FP16 and C’=ceil(C, 32)):

+-----------------+-----------------+-----------------+-------------------+
| Mode            | Width           | Height          | Channel           |
+=================+=================+=================+===================+
| Winograd        | 4 (The size     | 4 (The size     | C’\*stride_x\*str |
|                 | after           | after           | ide_y             |
|                 | pre-transform)  | pre-transform)  |                   |
+-----------------+-----------------+-----------------+-------------------+
| Image input     | 1               | R               | C\*S              |
+-----------------+-----------------+-----------------+-------------------+
| Direct_CONV     | S               | R               | C                 |
+-----------------+-----------------+-----------------+-------------------+

-  CSC.CONV_STRIDE_X/Y_EXT: The stride size seen from CSC. (SX/SY is the
   stride size configured for CDMA: D_CONV_STRIDE)

+-------------+----------+----------+
| Mode        | Stride_X | Stride_Y |
+=============+==========+==========+
| Winograd    | 1        | 1        |
+-------------+----------+----------+
| Image input | SX       | SY       |
+-------------+----------+----------+
| Direct_CONV | SX       | SY       |
+-------------+----------+----------+

-  CSC.D_ATOMICS: Hardware uses this register to decide stripe size:

.. code:: c

  int calc_stripe_size(int atomics, int processed)     
  {                                                    
      int stripe_size;                                     
      int remain_atomics = atomics - processed;            
      if ( remain_atomics < 32 && remain_atomics >= 16 ) { 
          stripe_size = remain_atomics;                        
      } else {                                             
          assert(remain_atomics > 16);                         
          stripe_size = 16;                                    
      }                                                    
                                                           
      return stripe_size;                                  
  }                                                    

The register value of D_ATOMICS itself is calculated by:

.. code:: c

  int calc_atomics(int out_width, int out_height) 
  {                                               
      return out_width*out_height-1;                  
  }                                               

-  CSC.D_RELEASE: Hardware uses this field to decide how many input
   slices should be released after current hardware layer.

-  <CDMA|CSC>.ZERO_PADDING_VALUE: see `Convolution convertors`_. Be noticed both CDMA
   and CSC has this register, but they has different meaning:

For CDMA, the padding value in register will be operated w/ CDMA input
convertor, the convert output is the actual padding value applied;

For CSC, the padding value in register will be directly applied w/o any
more operation;

-  CACC.D_DATAOUT_MAP:

This register is used to control the data reordering logic in CACC,
the configuration of this register should follow the table
below:

+--------------------+-------------+-------------+
| Configure          | Line_Packed | Surf_Packed |
+====================+=============+=============+
| 1x1                | True        | True        |
+--------------------+-------------+-------------+
| Multi-Batch mode   | False       | False       |
+--------------------+-------------+-------------+
| Direct convolution | False       | False       |
+--------------------+-------------+-------------+
| Winograd           | False       | False       |
+--------------------+-------------+-------------+

-  CACC. D_DATAOUT_SIZE_0

   This register is used to set the output size of convolution:

+-----------+--------------------------+---------------------------+
| CONV_MODE | DATAOUT_WIDTH            | DATAOUT_HEIGHT            |
+===========+==========================+===========================+
| DC        | S’=(S-1)*dilation_x + 1  | R’=(R-1)*dilation_y + 1   |
|           |                          |                           |
|           | (LP+RP-S’)/stride_x + 1  | (TP+H+BP-R’)/stride_y + 1 |
+-----------+--------------------------+---------------------------+
| IMG       | (LP+W+RP-S)/stride_x + 1 | (TP+H+BP-R)/stride_y + 1  |
+-----------+--------------------------+---------------------------+
| Winograd  | CSC.WIDTH_EXT – 4        | CSC.HEIGHT_EXT - 4        |
+-----------+--------------------------+---------------------------+

.. deconvolution-1:

Deconvolution
~~~~~~~~~~~~~

Deconvolution is a software feature, but it’s necessary to mention the
basic flow here to help user understand how it’s supported.

There’re 2 phases:

-  Convolution:

This phase includes conv_stride_x \* conv_stride_y hardware layers.

1) Software should split the kernels to conv_stride_x*conv_stride_y sets.
   Suppose the original kernel size is:
   RxSxC, the splitted kernel size is:

   S’=ceil(S/stride_x)

   R’=ceil(R/stride_y)

   C’=C

   K’=K

2) Kick-off convolution hardware layers based on different kernel set.
   The output cube size of each hardware layer is:

   W’ = (W-S’)+1

   H’=(H-R’)+1

   C’=K

-  Reorder:

The output cube from phase I is not the order we want, Rubik engine
should be employed to reorder it.

There’re 2 options about how those hardware layers should be scheduled:

a) Finish all stride_x*stride_y hardware layers then start rubik, total
   hardware layers is: stride_x*stride_y (convolution) + 1 (rubik);

b) Finish stride_x convolution hardware layers then start rubik, total
   hardware layers is: (stride_x + 1)*stride_y;

Generally, b) is the suggested scheduling strategy because:

1) It has better performance, here’s a timeline diagram which shows
   method a) vs b). It shows b) is (stride_x*stride_y-1)*t1 quicker than
   a).

.. _fig_image117_deconv_scheduling:

.. figure:: ias_image117_deconv_schedluing.svg
  :align: center

2) Method b) has smaller memory footprint requirement (W’, H’ are the
   output width/height of each convolution hardware layer).

+-----------------+--------------------+--------------------+--------------------+
| Method          | Convolution        | Rubik output       | Total              |
|                 | output buffer      | buffer             |                    |
+=================+====================+====================+====================+
| Method a)       | W’\*H’\*K\*stride_ | W’\*H’\*K\*stride_ | 2\*W’\*H’\*K\*strid|
|                 | x\*stride_y        | x\*stride_y        | e_x\*stride_y      |
+-----------------+--------------------+--------------------+--------------------+
| Method b)       | W’\*H’\*K\*stride_ | W’\*H’\*K\*stride_ | W’\*H’\*K\*stride_ |
|                 | x\*2               | x\*stride_y        | x\*(stride_y+2)    |
|                 |                    |                    |                    |
|                 | (x2 is not         |                    |                    |
|                 | mandatory but      |                    |                    |
|                 | suggested for      |                    |                    |
|                 | performance)       |                    |                    |
+-----------------+--------------------+--------------------+--------------------+

For most case, stride_y>2 thus method b) has smaller memory requirement.

SDP programming
---------------

Not all the use scenarios in :numref:`tab_sdp_supported_use_scenarios` are necessary to explain, we’ll
discuss bias addition/batch-norm/element-wise operations below (other
features are precision related which already covered by `Precision Preservation`_):

.. bias-addition-2:

Bias addition
~~~~~~~~~~~~~

As mentioned in :numref:`tab_sdp_supported_use_scenarios`, bias addition can be done by any of SDP
sub-module, let’s take using X1 sub-module for bias addition as an
example to explain the programming sequence:

-  Software has to prepare bias data cube, it has to be INT16 for
   INT8/16 pipeline and FP16 for FP16 pipeline.

-  Configure the SDP RDMA (most of the registers are intuitional, will
   highlights bias specific registers only ):

   a. We use bias addition, so, BRDMA_DATA_USE=ALU should be configured

   b. BRDMA_DATA_MODE configuration is based on bias mode

-  Configure the SDP BS sub-module:

   a. D_DP_BS_CFG

      BS_BYPASS=NO

      BS_ALU_BYPASS=NO

      BS_ALU_ALGO = SUM

      BS_MUL_BYPASS = YES

   b. D_DP_BS_ALU_CFG

      For per-element/kernel bias, operands should come from MC:

      BS_ALU_SRC = MEM

      For per cube bias, operands should come from register:

      BS_ALU_SRC = REG

      BS_ALU_SRC_VALUE = ?? (The value you want)

      BS_ALU_SHIFT_VALUE: Based on precision study results

.. batch-normalization-2:

Batch normalization
~~~~~~~~~~~~~~~~~~~

Batch normalization can be realized by any of X/Y, let’s still use
X1 sub-module as an example to show the steps to program batch
normalization:

-  Software has to tightly pack mean/variance into one data cube
   (M0V0M1V1…), if mean/variance are 2 bytes per element there’ll be 4
   bytes for a mean/variance pair. Those 2 bytes will be interpreted as
   INT16 for INT8/16 pipe and FP16 for FP16 pipe.

-  Configure the SDP RDMA (most of the registers are intuitional, will
   highlights batch-norm specific registers only ):

   a. Both ALU/MUL will be used for batch normalization, so,
      BRDMA_DATA_USE=BOTH should be configured

   b. BRDMA_DATA_MODE configuration is based on batch normalization mode

-  Configure the SDP BS sub-module:

   a. D_DP_BS_CFG

      BS_BYPASS=NO

      BS_ALU_BYPASS=NO

      BS_ALU_ALGO = SUM

      BS_MUL_BYPASS = NO

   b. D_DP_BS_ALU_CFG

      BS_ALU_SRC = MEM (Bias data always from MC regardless of
      per-kernel/element)

      BS_ALU_SHIFT_VALUE: Based on precision study results

   c. D_DP_BS_MUL_CFG

      BS_MUL_SRC=MEM

      BS_MUL_SHIFT_VALUE: Based on precision study results

For any case when both MUL/ALU are used, we can support combinations
below:

+-----------------+-----------------+
| ALU             | MUL             |
+=================+=================+
| REG             | MC              |
+-----------------+-----------------+
| MC              | REG             |
+-----------------+-----------------+
| MC, Per-channel | MC, Per-channel |
+-----------------+-----------------+
| MC, Per-element | MC, Per-element |
+-----------------+-----------------+
| REG             | REG             |
+-----------------+-----------------+

.. element-wise-1:

Element-wise
~~~~~~~~~~~~

Element-wise can be realized by any of SDP sub-unit, again, let’s still
use X1 module as an example about the element-wise configuration steps:

-  Different from bias/batch-norm, the element-wise input cube is from
   upstream hardware layer thus software didn’t need do anything to
   prepare surface

-  Configure the SDP RDMA (most of the registers are intuitional, will
   highlights element-wise specific registers only ):

   a. BRDMA_DATA_USE=? Is based on element-wise type. For PROD eltwise
      operation, it should be MUL, otherwise, use ALU;

   b. BRDMA_DATA_MODE= PER_ELEMENT

-  Configure the SDP BS sub-module:

   a. D_DP_BS_CFG

      BS_BYPASS=NO

      BS_ALU_BYPASS=? (For eltwise=MAX/SUM)

      BS_ALU_ALGO : Based on element-wise operation type

      BS_MUL_BYPASS = ? (No, For eltwise=PROD)

   b. D_DP_BS_ALU_CFG

      BS_ALU_SRC = MEM

      BS_ALU_SHIFT_VALUE: Based on precision study results

   c. D_DP_BS_MUL_CFG

      BS_MUL_SRC = MEM

      BS_MUL_SHIFT_VALUE: Based on precision study results

Compare mode
~~~~~~~~~~~~

Normal comparision
^^^^^^^^^^^^^^^^^^

SDP implemented compare mode in Y module to support software based
redundant computing.

+-----------------------------------+-----------------------------------+
| Use scenarios                     | Description                       |
+===================================+===================================+
| Offline vs offline                | Both of the 2 data stream are     |
|                                   | come from MC/SRAM                 |
|                                   |                                   |
|                                   | The is used to support            |
|                                   | postprocessor modules (CDP/PDP)   |
|                                   | redundant computing               |
+-----------------------------------+-----------------------------------+

In this mode, SW will schedule 3 HWLs:

1\ :sup:`st` HWL to run any module then output result to addr0;

2\ :sup:`nd` HWL to run exact the same setting as 1\ :sup:`st` layer
then output to addr1;

3\ :sup:`rd` HWL to run SDP_Y in compare mode which has configuration
as:

D_SRC_BASE_ADDR_LOW/HIGH = addr0

D_EW_BASE_ADDR_LOW/HIGH = addr1

D_DP_BS_CFG.BS_BYPASS=YES

D_DP_BN_CFG.BN_BYPASS=YES

D_DP_EW_CFG. EW_BYPASS = NO

D_DP_EW_CFG. EW_ALU_BYPASS=NO

D_DP_EW_CFG. EW_ALU_ALGO=EQL

After 3\ :sup:`rd` HWL execution done, SW should check D_STATUS to see
whether difference found.

**NOTE: When SDP EQL mode is enabled, D_FEATURE_MODE_CFG.WINOGAD has to
be OFF and D_FEATURE_MODE_CFG.BATCH_NUMBER has to be 0**

Batch mode comparison
^^^^^^^^^^^^^^^^^^^^^

Batch mode is a special case of offline/offline comparison, as SDP_Y
RDMA doesn’t support load multiple data cubes in one HWL, batch mode has
to be handled in a special way. There’re 2 cases: In order to facilitate
further discussion, we denote symbols below:

*Dimension: WxHxC*

*Batch_Num: N*

*Batch stride: BATCH_STRIDE*

There’re 2 cases depending on the attributes of each data cube:

-  If the data cube are line packed and surface packed:

For thise case, we’ll treat N data cubes as one super cube:

W’= ceil(C/KPG)*W*H, KPG= is_int8 ? 32:16;

H’=N

C’=KPG

line_stride: BATCH_STRIDE

surface_stride: BATCH_STRIDE*N

-  Otherwise:

As there’re bubbles between each data cube and the contents of those
bubbles are un-determistic, we have to compare those cube one by one
thus N HWL are necessary.

PDP programming
---------------

The most complex logic for PDP programming is deciding which working
mode can be used. PDP supports 3 different working modes:

+-----------------------------------+-----------------------------------+
| Mode                              | Attribute                         |
+===================================+===================================+
| On-the-fly                        | Input data comes from SDP,        |
|                                   | recommended whenever possible     |
+-----------------------------------+-----------------------------------+
| Offline - No split width          | Comparing with on-the-fly, this   |
|                                   | mode need one SDP write and one   |
|                                   | PDP read, this increased the      |
|                                   | memory traffic                    |
+-----------------------------------+-----------------------------------+
| Offline – split width             | Comparing with “no split width”,  |
|                                   | this mode need over-fetch between |
|                                   | overlapped region thus bandwidth  |
|                                   | further increased                 |
+-----------------------------------+-----------------------------------+

The working mode selection strategy is:

-  As mentioned in Section "Planar Data Processor" of Unit Description document, PDP has 7KB internal buffer to save
   intermediate results during pooling, thus the maximum supported
   output width is a fixed number. (Refer to: 10.1.4:
   calculate_pdp_max_width)

-  Calculate the actual pooling output:

.. code:: c

  pooled_width = static_cast<int>(ceil(static_cast<float>(width + pad_left + pad_right - kernel_w) / stride_w)) + 1;
  if ((pooled_width - 1) \* stride_w >= width + pad_left) {       
      --pooled_width;                                                 
  }                                                               

-  Decide working mode

.. code:: c

  typedef enum {                                                        
      PDP_FLYING_MODE,                                                      
      PDP_OFFLINE_MODE,                                                     
  } pdp_mode;                                                           
  static pdp_mode get_pdp_mode( int width_output, int max_fly_width, bool is_full_conv )
  {
      // convolution mode should also be taking into consideration: If software split
      // convolution layer into different hardware layers, PDP can't working on-the-fly
      return (width_output <= max_fly_width) && is_full_conv ? PDP_FLYING_MODE : PDP_OFFLINE_MODE;                                   
  }                                                                     

-  If PDP working offline mode, we need to calculate splitted width and
   split number as well (please see: 10.1.4 for detail)

   Be noticed: The pseudo code in: 10.1.3 just configured to make
   hardware work, if possible, software should try to make sure the
   starting address (in/out or both) of each splitted band be 256 align,
   this will greatly improve NVDLA memory throughput.

On-the-fly processing
~~~~~~~~~~~~~~~~~~~~~

The programming sequence for on-the-fly PDP mode is (most of the
registers are intuitional, will highlights on-the-fly mode specific
registers only):

-  PDP-RDMA is not necessary to config because our input is from SDP;

-  D_OPERATION_MODE_CFG

   POOLING_METHOD: Based on pooling method used in algorithm

   FLYING_MODE= ON_FLYING

   SPLIT_NUM=0

Offline processing without split width
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The programming sequence for this mode is:

-  Appropriate address/memory type should be set to PDP-RDMA;

-  D_OPERATION_MODE_CFG

   POOLING_METHOD: Based on pooling method used in algorithm

   FLYING_MODE= OFF_FLYING

   SPLIT_NUM=0

-  D_PARTIAL_WIDTH_IN

   PARTIAL_WIDTH_IN_FIRST=info->first_in_width

-  D_PARTIAL_WIDTH_OUT

   PARTIAL_WIDTH_OUT_FIRST=info->first_out_width

Offline processing with split width
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The programming sequence for this mode is:

-  Appropriate address/memory type should be set to PDP-RDMA;

-  D_OPERATION_MODE_CFG

   POOLING_METHOD: Based on pooling method used in algorithm

   FLYING_MODE= OFF_FLYING

   SPLIT_NUM=info->split_num

-  D_PARTIAL_WIDTH_IN

   PARTIAL_WIDTH_IN_FIRST=info->first_in_width

   PARTIAL_WIDTH_IN_MID=info->split_num==1 ? 0:info->mid_in_width

   PARTIAL_WIDTH_IN_LAST= info->last_in_width

-  D_PARTIAL_WIDTH_OUT

   PARTIAL_WIDTH_OUT_FIRST=info->first_out_width

   PARTIAL_WIDTH_OUT_MID= info->split_num==1 ? 0:info->mid_out_width

   PARTIAL_WIDTH_OUT_LAST= info->last_out_width

When hardware processing done, there’ll be interrupt fired from PDP
submodule to inform CPU that PDP hardware layer is done for any of above
mode.

.. register-definition-1:

Register definition
~~~~~~~~~~~~~~~~~~~

Beside working modes, it’s also necessary to mention some of the
interested registers:

a. D_POOLING_PADDING_CFG: The padding size on left/right/top/bottom. If
   greater than 0, D_POOLING_PADDING_VALUE_*_CFG will be appended to
   input data. This register will be take into account for AVE/MAX/MIN
   mode;

b. D_POOLING_PADDING_VALUE_*_CFG: The padded value. This register will
   be took into account for AVE mode only;

CDP programming
---------------

CDP always working on offline, there’s no special mode for CDP and the
precision related configuration already discussed.
So, skip the CDP programming here.

After hardware layer processing done, there’ll be interrupt fired to
CPU.

Debug features
--------------

NVDLA implemented debug registers to facilitate silicon debug. Those
registers are dedicated per register group and won’t be cleared until
the corresponding group starts. It will be incremented by 1 when certain
condition meets.

Those registers can be classified as 2 groups below:

Precision debug
~~~~~~~~~~~~~~~

If saturation counter (see `Convertor statistics`_) exceed threshold (defined by
software), this means convertor parameters (scaling, offset) are
in-properly set;

If LUT overflow/underflow counter (`LUT statistics`_) exceed threshold (defined
by software), this means LUT is in-properly set;

Performance debug
~~~~~~~~~~~~~~~~~

NVDLA is a fix function engine, the latency is predictable inside each
sub-unit, but the read/write response from out-side is not deterministic
thus we implemented performance registers below to help SW analysis the
bottleneck of un-expected performance drop.

+----------------------+----------------------+----------------------+
| Sub unit             | Register name        | Description          |
+======================+======================+======================+
| CDMA                 | D_PERF_ENABLE        | Control register to  |
|                      |                      | enable/disable perf  |
|                      |                      | Counter              |
+----------------------+----------------------+----------------------+
|                      | D_PERF_DAT_READ_STAL | Count stall cycles   |
|                      | L                    | of data read DMA for |
|                      |                      | one layer            |
+----------------------+----------------------+----------------------+
|                      | D_PERF_WT_READ_STALL | Count total latency  |
|                      |                      | of data read DMA for |
|                      |                      | one layer            |
+----------------------+----------------------+----------------------+
|                      | D_PERF_DAT_READ_LATE | Count stall cycles   |
|                      | NCY                  | of weight read DMA   |
|                      |                      | for one layer        |
+----------------------+----------------------+----------------------+
|                      | D_PERF_WT_READ_LATEN | Count total latency  |
|                      | CY                   | of weight read DMA   |
|                      |                      | for one layer        |
+----------------------+----------------------+----------------------+
| SDP                  | D_PERF_ENABLE        | Control register to  |
|                      |                      | enable/disable perf  |
|                      |                      | Counter              |
+----------------------+----------------------+----------------------+
|                      | D_PERF_MRDMA_READ_ST | Count stall cycles   |
|                      | ALL                  | of M read DMA for    |
|                      |                      | one layer            |
+----------------------+----------------------+----------------------+
|                      | D_PERF_BRDMA_READ_ST | Count stall cycles   |
|                      | ALL                  | of B read DMA for    |
|                      |                      | one layer            |
+----------------------+----------------------+----------------------+
|                      | D_PERF_NRDMA_READ_ST | Count stall cycles   |
|                      | ALL                  | of N read DMA for    |
|                      |                      | one layer            |
+----------------------+----------------------+----------------------+
|                      | D_PERF_ERDMA_READ_ST | Count stall cycles   |
|                      | ALL                  | of E read DMA for    |
|                      |                      | one layer            |
+----------------------+----------------------+----------------------+
|                      | D_PERF_WDMA_WRITE_ST | Count stall cycles   |
|                      | ALL                  | of write DMA for one |
|                      |                      | layer                |
+----------------------+----------------------+----------------------+
| CDP                  | D_PERF_ENABLE        | Control register to  |
|                      |                      | enable/disable perf  |
|                      |                      | Counter              |
+----------------------+----------------------+----------------------+
|                      | D_PERF_READ_STALL    | Count stall cycles   |
|                      |                      | of read DMA for one  |
|                      |                      | layer                |
+----------------------+----------------------+----------------------+
|                      | D_PERF_WRITE_STALL   | Count stall cycles   |
|                      |                      | of wirte DMA for one |
|                      |                      | layer                |
+----------------------+----------------------+----------------------+
| PDP                  | D_PERF_ENABLE        | Control register to  |
|                      |                      | enable/disable perf  |
|                      |                      | Counter              |
+----------------------+----------------------+----------------------+
|                      | D_PERF_READ_STALL    | Count stall cycles   |
|                      |                      | of read DMA for one  |
|                      |                      | layer                |
+----------------------+----------------------+----------------------+
|                      | D_PERF_WRITE_STALL   | Count stall cycles   |
|                      |                      | of wirte DMA for one |
|                      |                      | layer                |
+----------------------+----------------------+----------------------+
| RUBIK                | D_PERF_ENABLE        | Control register to  |
|                      |                      | enable/disable perf  |
|                      |                      | Counter              |
+----------------------+----------------------+----------------------+
|                      | D_PERF_READ_STALL    | Count stall cycles   |
|                      |                      | of read DMA for one  |
|                      |                      | layer                |
+----------------------+----------------------+----------------------+
|                      | D_PERF_WRITE_STALL   | Count stall cycles   |
|                      |                      | of wirte DMA for one |
|                      |                      | layer                |
+----------------------+----------------------+----------------------+
| BDMA                 | CFG_STATUS_PERF_STAL | Control register to  |
|                      | L_COUNT_EN           | enable/disable perf  |
|                      |                      | Counter              |
+----------------------+----------------------+----------------------+
|                      | STATUS_PERF_GRP0_REA | Count stall cycles   |
|                      | D_STALL              | of read DMA for      |
|                      |                      | group0               |
+----------------------+----------------------+----------------------+
|                      | STATUS_PERF_GRP0_WRI | Count stall cycles   |
|                      | TE_STALL             | of wirte DMA for     |
|                      |                      | group0               |
+----------------------+----------------------+----------------------+
|                      | STATUS_PERF_GRP1_REA | Count stall cycles   |
|                      | D_STALL              | of read DMA for      |
|                      |                      | group1               |
+----------------------+----------------------+----------------------+
|                      | STATUS_PERF_GRP1_WRI | Count stall cycles   |
|                      | TE_STALL             | of read DMA for      |
|                      |                      | group1               |
+----------------------+----------------------+----------------------+

For each sub-unit, we have “EN” register to allow software
enable/disable those counting register to save power.

Limitation
----------

Though we’ve already highlight hardware restrictions in the chapters
above, but we’d like to centralize the limitations here to facilitate
users quick check illegal settings.

Data Format
~~~~~~~~~~~

-  The “Invalid case” in :numref:`tab_precision_conversion_conv` to :numref:`tab_precision_conversion_poolong` are not allowed;

-  The alignment for address/line_stride/surf_stride in ::numref:`tab_requirements_of_alignment` should
   be satisfied when allocating buffer;

-  LINE_STRIDE: line stide has to bigger than the actual size per line,
   please refer to: 10.1.1 for minimal line_stride calculation;

-  For 1x1xC cube, it should always be line_packed and surf_packed.

CSB_MASTER
~~~~~~~~~~

-  Any read access or write access to reserved register address
   (0x14000~0x3FFFF) is forbidden. CSB master do not
   support for these addresses. Any access to these addresses may cause
   unknow result.

BDMA
~~~~

-  When both group0 and group1 are both busy, no more command is allowed
   even if there are free slot

-  All operations in one BDMA HWL should has the same destination memory
   type (DST_RAM_TYPE)

.. convolution-pipeline-1:

Convolution
~~~~~~~~~~~~~~~~~~~~

General
^^^^^^^

-  There’re multiple pipeline stages in convolution pipeline, the
   op_enable programming sequence has to be in reverse order, e.g.:
   CONV_ACCUCONV_MACCONV_SCCONV_BUFCONV_DMA

-  WMB and weight data MUST has the same RAM type.

-  If weight_format=compressed, banks_for_data+banks_for_weight must be
   less than 16 (Bank 15 is reserved for WMB).

-  WEIGHT_BANK should be allocated large enough to store one kernel
   group weight plus 128Bytes; For compression mode, BANK for WMB is
   fixed as 1, this means WMB for one kernel group should always less
   than 32KB-128B so that additional 128Bytes can be stored in that
   bank.

-  CSC:: RLS_SLICES: This register should never exceed
   DATAIN_HEIGHT_EXT, Even with the partial release in pervious layer,
   the unreleased slices will be counted into datain_height_ext of CSC
   register (but not in datain_height of CDMA register).

   For example, in first layer we input 10 slices and release 6 slices,
   there are 4 slices remain in CBUF.

And with second layer we fetch new 7 slices from CDMA and combined with
remain slices to do convolution. The setting of CDMA datain_height
should be 7 and CSC datain_height_ext should be (7+4) = 11. And at this
time rls_slices should not more than 11.

-  The right/bottom padding should be carefully configured to make sure
   all the data will be used for convolution, which means:

.. math:: (Output\_Width - 1) * stride\_x + S == PL + Input\_Width + PR
.. math:: (Output\_height - 1) * stride\_y + R == PT + Input\_Height + PB

Where, PL/PT are the left/top padding which are get from network
definition; PR/PB are the right/bottom padding which are configured by
user;

-  Data re-use can be take effect when all conditions below are meet:

   -  Skip_rls is set as true for previous layer;

   -  Conv_mode and DATA_BANK are kept unchanged comparing with previous
      layer;

-  Left/Right padding should be less than S, Top/Bottom padding should
   be less than R

Image
^^^^^

-  For image input, pixel_y_offset should be set as 0 for pitch linear;

-  If channel post extension enabled, the limitations in :numref:`tab_limits_of_channel_post_extension`  has
   to be meet;

-  Dilation is not supported

DC
^^

-  No special limitation;

Winograd
^^^^^^^^

-  Output width and height must be 4 divisible and >= 8;

-  The equivalent kernel size should be 3x3;

Multi-batch
^^^^^^^^^^^

-  The start address of each input feature cube has to be carefully
   arranged to make sure their offset is a fixed number as BATCH_STRIDE.

Supported feature crossing:

+---------+---------+---------+---------+---------+---------+---------+
|         | Channel | Multi-b | Deconv  | Image   | Dilatio | Winogra |
|         | -post   | atch    |         | Input   | n       | d       |
|         | extensi |         |         |         |         |         |
|         | on      |         |         |         |         |         |
+=========+=========+=========+=========+=========+=========+=========+
| Channel |         | N       | N       | Y       | N       | N       |
| -post   |         |         |         |         |         |         |
| extensi |         |         |         |         |         |         |
| on      |         |         |         |         |         |         |
+---------+---------+---------+---------+---------+---------+---------+
| Multi-b | N       |         | Y       | N       | Y       | N       |
| atch    |         |         |         |         |         |         |
+---------+---------+---------+---------+---------+---------+---------+
| Deconv  | N       | Y       |         | N       | N       | N       |
+---------+---------+---------+---------+---------+---------+---------+
| Image   | Y       | N       | N       |         | N       | N       |
| In      |         |         |         |         |         |         |
+---------+---------+---------+---------+---------+---------+---------+
| Dilatio | N       | Y       | N       | N       |         | N       |
| n       |         |         |         |         |         |         |
+---------+---------+---------+---------+---------+---------+---------+
| Winogra | N       | N       | N       | N       | N       |         |
| d       |         |         |         |         |         |         |
+---------+---------+---------+---------+---------+---------+---------+

LUT
~~~

-  For linear mode, the table start/end should meet the requirements
   below:

   LE_END-LE_START == 1<<(LE_INDEX_SELECT+6)

   LO_END-LO_START == 1<<(LO_INDEX_SELECT+8)

-  For linear mode, the “select” field shouldn’t exceed the bit-depth of
   hardware thus we have limitations below:

+-------+-----------------+-----------------+
|       | SDP             | CDP             |
+=======+=================+=================+
| INT8  | LE: [-6~25]     | LE: [-6~15]     |
|       |                 |                 |
|       | LO: [-8~23]     | LO: [-8~13]     |
+-------+-----------------+-----------------+
| INT16 | LE: [-6~25]     | LE: [-6~31]     |
|       |                 |                 |
|       | LO: [-8~23]     | LO: [-8~29]     |
+-------+-----------------+-----------------+
| FP16  | LO: [-128, 119] | LO: [-128, 119] |
|       |                 |                 |
|       | LE: [-128, 121] | LE: [-128, 121] |
+-------+-----------------+-----------------+

For FP16 above, another constrain should take into consideration:
LX_START/END registers are FP32 and:

LE_END = LE_START + pow(2, LE_INDEX_SELECT +6)

In order to make sure LE_END larger than LE_START, constrain below
should be satisfied:

LE_START/pow(2, LE_INDEX_SELECT +6) < pow(2, 24), thus:

LE_START < pow(2, LE_INDEX_SELECT+30)

For the same reason, LO_START < pow(2, LO_INDEX_SELECT+32)

-  For exponential mode, the table start/end should meet the
   requirements below:

   LE_END-LE_START==(1<<(LE_INDEX_OFFSET+64)).

If the value calculated by formula below exceed the INT32/FP32
representable, use INT32_MAX or FP32_MAX instead.

-  For exponential mode, we also have constrain on LE_INDEX_OFFSET:

+-------+-------------+-------------+
|       | SDP         | CDP         |
+=======+=============+=============+
| INT8  | [-64, 31]   | [-64, 20]   |
+-------+-------------+-------------+
| INT16 | [-64, 31]   | [-64, 36]   |
+-------+-------------+-------------+
| FP16  | [-126, 127] | [-126, 127] |
+-------+-------------+-------------+

SDP
~~~

General
^^^^^^^

-  When SRC is configured as REG, corresponding RDMA shouldn’t be
   enabled.

-  If EQL mode is enabled, Y ALU convertor must be bypassed (except
   FP16) and all the operations after ALU should be bypassed.

-  If PReLU is enabled for one sub-unit, the ALU in that unit MUST be
   bypassed.

-  For PROC_PRECISION==FP16:

   If EW_ALU_BYPASS==NO && D_DP_EW_ALU_CFG. EW_ALU_SRC==MEM, then,
   EW_ALU_CVT_BYPASS must be NO;
   
   If EW_MUL_BYPASS==NO && D_DP_EW_MUL_CFG. EW_MUL_SRC==MEM, then,
   EW_MUL_CVT_BYPASS must be NO;

DC
^^

-  Precision conversion is not allowed if SDP output to PDP or EQL mode;

-  For INT16INT8, HW has no requirement on channel size configuration,
   but if C is not 32 elements aligned, HW will read/write the
   additional memory thus SW has to guarantee the allocated src/dst data
   cube is big enough;

Winograd/Batch
^^^^^^^^^^^^^^

-  SDP has to work on the fly with CC

-  SDP_Y can’t work at EQL mode (EW_ALU_ALGO != EQL)

-  If multi-batch is enabled, registers below has to be 64 bytes
   aligned:

   | DST_BASE_ADDR
   | DST_LINE_STRIDE
   | DST_SURFACE_STRIDE

   BS/BN/EW_BS_BASE_ADDR_LOW/HIGH

CDP
~~~

-  Maximum supported local_size is 9.

PDP
~~~

-  PL/PR should be carefully programmed to make sure each input sample
   are used:

   (PL+W+PR-Kernel_W)%stride_x == 0

-  PL/PR should be less than kernel_width;

-  For any mode, first/mid/last_out_width should be less than maximum
   flying width (see 10.1.4)

-  For non-split mode, CUBE_IN_WIDTH + PL should be equals to
   (CUBE_OUT_WIDTH-1)*stride_x + kernel_width;

-  For split mode:

For split_num =2:

-  First_out_width + last_out_width should be equals to CUBE_OUT_WIDTH;

-  First_in_width + PL should be equals to (first_out_width-1)*stride_x
   + kernel_width;

-  Last_in_width + PR + overlap should be equals to
   (last_out_width-1)*stride_x + kernel_width;

-  if kernel >=kernel_stride, kernel_w – stride_x should be <=
   first_in_width; Otherwise, stride_x – kernel_w < last_in_width;

For split_num > 2:

-  first_out_width + last_out_width + mid_out_width*(split_num-2) should
   be equals to CUBE_OUT_WIDTH;

-  first_in_width + PL should be equals to (first_out_width-1)*stride_x
   + kernel_w;

-  mid_in_width + overlap should be equals to (mid_out_width-1)*stride_x
   + kernel_w;

-  last_in_width + PR + overlap should be equals to
   (last_out_width-1)*stride_x + kernel_w;

-  if kernel_w >=kernel_stride, kernel_w – stride_x should be <=
   <first|mid>_in_width; Otherwise, stride_x – kernel_w should be <
   <last|mid>_in_width;

-  Maximum supported pooling kernel size is 8

.. rubik-1:

Rubik
~~~~~

-  For contract mode, the address/line_stride for both input/output
   should be 32bytes aligned;

-  For split/merge mode, the address/line_stride should be 64bytes
   aligned for planar data(output of split mode, input of merge mode)

-  deconv_x_stride \* datain_width should be <=8192

