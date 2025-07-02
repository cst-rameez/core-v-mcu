..
   Copyright (c) 2023 OpenHW Group
   Copyright (c) 2024 CircuitSutra

   SPDX-License-Identifier: Apache-2.0 WITH SHL-2.1

.. Level 1
   =======

   Level 2
   -------

   Level 3
   ~~~~~~~

   Level 4
   ^^^^^^^
.. _apb_timer:

APB Timer
=========

APB Timer primarily generates interrupts to the Core complex or CPU subsystem with a finite configurable delay. It manages the interrupt generation through various configurations of prescaler, reference clock and timer counters. 

Features
---------
-  Multiple trigger input sources

-  Two 32-bit configurable prescaler

-  Configurable interrupts

-  Support for two independent 32-bit timers or a single 64-bit timer

-  Supports one-shot and compare-clear modes.

-  Configurable control operations of timer: (Start, Stop and Reset)


Block Architecture
------------------
APB timer can either be configured as two 32-bit independent timers or a single 64 bit timer. 
It has two timers, Timer_hi and Timer_lo which are responsible to generate irq_lo_o or irq_hi_o interrupt signals to Core Complex/CPU. 
Working of Timer_lo and Timer_hi is functionally identical in the 32 bit mode, whereas Timer_lo and Timer_hi are cascaded in 64 bit mode.
The APB timer can be configured using CSRs. The CSRs of the APB timer are accessible using the APB bus.

The figure below is a high-level block Diagram of APB Timer:

.. figure:: apb_timer_block_diagram.png
   :name: apb_timer_block_diagram
   :align: center
   :alt: 

Working of APB Timer
~~~~~~~~~~~~~~~~~~~~
We will discuss the detailed working of APB Timer as a 32 bit Timer and also as a 64 bit Timer.

32 bit Timer
~~~~~~~~~~~~~
if the MODE_64_BIT in CFG_REG_LO CSR is '0', APB timer can be configured as a 32-bit timer i.e. Timer_hi/Timer_lo in one of the below following ways.

- Only Timer_hi can be configured.
- Only Timer_lo can be configured. 
- Both Timer_hi/Timer_lo can be configured independently at the same time.

Timer_hi and Timer_lo have the same design, with matching CSRs and input/output signals that differ only by '_hi' and '_lo' suffixes, making their control and behavior consistent.

Timer_lo can be enabled via the FW (ENABLE_BIT bitfield in CFG_REG_LO CSR is '1') or the external device input (When event_lo_i input signal and IEM_BIT bitfield in CFG_REG_LO CSR is '1').
The Timer counter checks whether the prescaler is enabled or not via the PRESCALER_EN_BIT bitfield in CFG_REG_LO CSR.

If the prescaler is not enabled. For every positive edge of the clock, Timer counter will start incrementing its internal counter till it reaches the TIMER_CMP_LO and generates the irq_lo_o interrupt.
 
If the prescaler is enabled. For every positive edge of the clock, the prescaler will start incrementing its internal counter till it reaches the PRESCALER_COMP bitfield in CFG_REG_LO CSR and it sets the prescaler_lo_target_reached to '1'.

For every instance of prescaler_lo_target_reached is '1', Timer counter will be enabled and it increments the internal counter by '1' until it reaches the TIMER_CMP_LO and generates the irq_lo_o interrupt.

If the CMP_CLR_BIT in CFG_REG_LO CSR is '1' then the Timer counter is reseted and it starts counting again to generate irq_lo_o and the same process is repeated.
If the ONE_SHOT_BIT in CFG_REG_LO is '1' then the Timer counter is disabled.

The figure below is a high-level block Diagram of a 32 bit APB Timer:

.. figure:: 32_bit_apb_timer.png
   :name: 32_bit_apb_timer
   :align: center
   :alt: 

Prescaler
^^^^^^^^^
Prescaler's main objective is to scale down the frequency of the input clock with the PRESCALER_COMP amount of times. where PRESCALER_COMP is the bitfield of CFG_REG_LO CSR.
Prescaler generates prescaler_lo_target_reached event after PRESCALER_COMP number of clock cycles. where PRESCALER_COMP is the bitfield of CFG_REG_LO CSR.
if the REF_CLK_EN_BIT in the CFG_REG_LO CSR is '1', then the prescaler will be in sync with the rising edge of the reference clock.

Prescaler maintains a prescaler counter whose initial value is '0'. 
For every positive edge of the HCLK clock, if prescaler is enabled, prescaler counter is incremented by value '1' until it reaches the PRESCALER_COMP bitfield of CFG_REG_LO CSR value.
Once the prescaler counter reaches the PRESCALER_COMP bitfield of CFG_REG_LO CSR value then prescaler_lo_target_reached event is '1'.
In the next positive edge of the selected clock, the prescaler is resetted, which means the prescaler counter and prescaler_lo_target_reached are resetted to '0'.
prescaler counter starts incrementing and the same process repeats to set the prescaler_lo_target_reached multiple times.

**Reset Prescaler:**

When the prescaler is resetted, the prescaler counter and prescaler_lo_target_reached are reset to '0'. 
Prescaler is resetted, if any of the below conditions is satisfied: 

- When PRESCALER_EN_BIT in the CFG_REG_LO CSR is '1' and prescaler_lo_target_reached is '1' for one clock cycle.
- When the RESET_BIT in the CFG_REG_LO CSR is '1'.
- When the RESET_LO in the TIMER_RESET_LO CSR is '1'. 

**Enable Prescaler:**

When the prescaler is enabled, it will start its operation and it can be enabled in the below condition:

- When PRESCALER_EN_BIT and ENABLE_BIT of CFG_REG_LO is '1' and stoptimer_i is '0'.

**Disable Prescaler:**

When the prescaler is disabled, it will pause its operation, that is the prescaler counter will not be set to '0'.
Prescaler is disabled, if any of the below conditions is satisfied: 

- When PRESCALER_EN_BIT in the CFG_REG_LO CSR is '0'.
- When ENABLE_BIT of CFG_REG_LO is '0'.
- When the input signal stoptimer_i is '1'.

Timer Counter 
^^^^^^^^^^^^^

Timer counter primarily generates the output interrupts irq_lo_o or irq_hi_o for the Core complex. 
if the IRQ_BIT is '1', 32 bit Timer generates irq_lo_o interrupt after TIMER_CMP_LO number of clock cycles. where TIMER_CMP_LO is a timer compare value for Timer_lo in TIMER_CMP_LO CSR.
if the REF_CLK_EN_BIT in the CFG_REG_LO CSR is '1', then the timer counter will be in sync with the rising edge of the reference clock.

Timer maintains a counter whose initial value is '0' and FW can overwrite/program this  counter to any value by configuring TIMER_VAL_LO CSR. 
For every positive edge of the selected clock, if timer counter is enabled, counter is incremented by value '1' until it reaches the TIMER_CMP_LO value.
Once the counter reaches TIMER_CMP_LO value and if the IRQ_BIT is '1', then the irq_lo_o interrupt will be asserted.

if one shot mode (ONE_SHOT_BIT bitfield of CFG_REG_LO CSR is '1') is enabled then in the next clock cycle, the timer counter is disabled then the timer will pause its operation. (i.e the timer counter will not be set to '0')

if the compare clear mode (CMP_CLR_BIT bitfield of CFG_REG_LO CSR is '1') is enabled then in the next clock cycle, then the counter of Timer_lo is resetted to '0'. 
The timer counter starts incrementing and the same process repeats to generate the irq_lo_o interrupt multiple times.

**Reset Timer counter:**

When the Timer counter is resetted, the counter is resetted to '0'. 
Timer counter is resetted, if any of the below conditions is satisfied: 

- When CMP_CLR_BIT in the CFG_REG_LO CSR is '1' and the counter reaches TIMER_CMP_LO value. (i.e Compare clear mode is enabled)
- When the RESET_BIT in the CFG_REG_LO CSR is '1'.
- When the RESET_LO in the TIMER_RESET_LO CSR is '1'. 

**Enable Timer counter:**

Timer counter will be enabled and it will start its operation if any of the below conditions is satisfied:

- when ENABLE_BIT of CFG_REG_LO is '1', PRESCALER_EN_BIT of CFG_REG_LO is '0' and stoptimer_i is '0' (If prescaler of Timer_lo is disabled).
- when PRESCALER_EN_BIT and ENABLE_BIT of CFG_REG_LO is '1' and prescaler_lo_target_reached is '1' and stoptimer_i is '0' (If prescaler of Timer_lo is enabled).
- The ENABLE_BIT of CFG_REG_LO is set by the HW to '1' and enables the timer, if the any of the below conditions is satisfied:
   - When the event_lo_i signal is '1' and IEM_BIT of CFG_REG_LO is '1'.
   - When TIMER_START_LO CSR is having any value other than '0'.

**Disable Timer counter:**

When the Timer counter is disabled, it will pause its operation, that is the counter will not be set to '0'.
Timer counter is disabled, if any of the below conditions is satisfied: 

- When ENABLE_BIT of CFG_REG_LO CSR is '0'.
- When the input signal stoptimer_i is '1'.

busy_o pin is used to provide the status of APB Timer will be driven high if either Timer_lo or Timer_hi is enabled.

64 bit Timer
~~~~~~~~~~~~~
if the MODE_64_BIT in CFG_REG_LO CSR is '1', APB timer can be configured as 64-bit timer i.e. Timer_lo and Timer_hi are cascaded with one another.   
As it acts as a single 64 bit Timer, for control operations CFG_REG_LO CSR, event_lo_i input signal and irq_lo_o output signals are used.
all the [*]_LO CSRs for Timer_lo and TIMER_CMP_HI, TIMER_VAL_HI, TIMER_RESET_HI CSRS and RESET_HI bitfield in CFG_REG_HI are also used for Timer_hi.


64 bit Timer i.e. cascaded Timer_lo and Timer_hi can be enabled via the FW (ENABLE_BIT bitfield in CFG_REG_LO CSR is '1') or the external device input (When event_lo_i input signal and IEM_BIT bitfield in CFG_REG_LO CSR is '1').
The Timer counter of Timer_lo checks whether the prescaler is enabled or not via the PRESCALER_EN_BIT bitfield in CFG_REG_LO CSR.

If the prescaler is not enabled. For every positive edge of the clock, Timer counter will start incrementing its Timer_lo counter till it reaches the TIMER_CMP_LO and Timer_hi counter reaches the TIMER_CMP_HI and it generates the irq_lo_o interrupt.
 
If the prescaler is enabled. For every positive edge of the clock, the prescaler will start incrementing its internal counter till it reaches the PRESCALER_COMP bitfield in CFG_REG_LO CSR and it sets the prescaler_lo_target_reached to '1'.
For every instance of prescaler_lo_target_reached is '1', Timer counter will be enabled and it increments the Timer_lo internal counter by '1' until it reaches the TIMER_CMP_LO and Timer_hi counter reaches the TIMER_CMP_HI then generates the irq_lo_o interrupt.

If the CMP_CLR_BIT in CFG_REG_LO CSR is '1' then the Timer counter is reseted and it starts counting again to generate irq_lo_o and the same process is repeated.
If the ONE_SHOT_BIT in CFG_REG_LO is '1' then the Timer counter is disabled.


The figure below is a high-level block Diagram of a 64 bit APB Timer:

.. figure:: 64_bit_apb_timer.png
   :name: 64_bit_apb_timer
   :align: center
   :alt: 

Prescaler
^^^^^^^^^
Only the prescaler of Timer_lo is used in 64 bit Timer. The working of the prescaler in 64 bit Timer is exactly the same as of the prescaler in 32 bit timer.
Please refer to the Prescaler sub-section in 32 bit Timer for more information.  

Timer Counter 
^^^^^^^^^^^^^
64 bit Timer uses Timer counters in Timer_lo and Timer_hi in a cascaded manner to primarily generate the output interrupts irq_lo_o for the Core complex. 
if the REF_CLK_EN_BIT in the CFG_REG_LO CSR is '1', then the timer counter will be in sync with the rising edge of the reference clock.

Timer_lo maintains a counter whose initial value is '0' and FW can overwrite/program this counter to any value by configuring TIMER_VAL_LO CSR. 
Timer_hi maintains a counter whose initial value is '0' and FW can overwrite/program this counter to any value by configuring TIMER_VAL_HI CSR. 

For every positive edge of the selected clock, if Timer_lo is enabled, counter of Timer_lo is incremented by value '1' until it reaches the 0xFFFFFFFF then Timer_hi is enabled and counter of Timer_hi is incremented by value '1'.
That means for every 0xFFFFFFFF number of times positive edge of the selected clock, the counter of Timer_hi is incremented by '1'. The same process is repeated till the counter of Timer_hi reaches TIMER_CMP_HI value.
Afterthe TIMER_CMP_LO number of positive edges of the selected clock, the counter of Timer_lo reaches TIMER_CMP_LO value.
When the counter of Timer_lo reaches TIMER_CMP_LO value and the counter of Timer_hi reaches TIMER_CMP_HI value and if the IRQ_BIT is '1', then the irq_lo_o interrupt will be asserted.

If one shot mode (ONE_SHOT_BIT bitfield of CFG_REG_LO CSR is '1') is enabled then in the next clock cycle, then both the Timer_lo and Timer_hi are disabled and the timer will pause its operation. 
(i.e. the timer counter will not be set to '0' and it will retain the current value)

If the compare clear mode (CMP_CLR_BIT bitfield of CFG_REG_LO CSR is '1') is enabled then in the next clock cycle, then counters of both the Timer_lo and Timer_hi are resetted to '0'. 
timer counters start incrementing and the same process repeats to generate the irq_lo_o interrupt multiple times.

**Enable Timer counter for Timer_lo:**

Timer counter for Timer_lo will be enabled and it will start its operation if any of the below conditions is satisfied:

- when ENABLE_BIT of CFG_REG_LO is '1', PRESCALER_EN_BIT of CFG_REG_LO is '0' and stoptimer_i is '0' (If prescaler of Timer_lo is disabled).
- when PRESCALER_EN_BIT and ENABLE_BIT of CFG_REG_LO is '1' and prescaler_lo_target_reached is '1' and stoptimer_i is '0' (If prescaler of Timer_lo is enabled).
- The ENABLE_BIT of CFG_REG_LO is set by the HW to '1' and enables the timer, if any of the below conditions is satisfied:
   - When the event_lo_i signal is '1' and IEM_BIT of CFG_REG_LO is '1'.
   - When TIMER_START_LO CSR is having any value other than '0'.

**Reset Timer counter for Timer_lo:**

When the Timer counter for Timer_lo is resetted, the counter is resetted to '0'. 
Timer counter for Timer_lo is resetted, if any of the below conditions is satisfied: 

- When CMP_CLR_BIT in the CFG_REG_LO CSR is '1' counter of Timer_lo reaches TIMER_CMP_LO value and the counter of Timer_hi reaches TIMER_CMP_HI value. (i.e Compare clear mode is enabled)
- When the RESET_BIT in the CFG_REG_LO CSR is '1'.
- When the RESET_LO in the TIMER_RESET_LO CSR is '1'. 

**Enable Timer counter for Timer_hi:**

Timer counter for Timer_hi will be enabled and it will start its operation if any of the below conditions is satisfied:

- when ENABLE_BIT of CFG_REG_LO is '1', counter value of Timer_lo is 0xFFFFFFFF and stoptimer_i is '0' (If prescaler_lo is disabled).
- when ENABLE_BIT of CFG_REG_LO and prescaler_lo_target_reached is '1', and counter value of Timer_lo is 0xFFFFFFFF and stoptimer_i is '0' (If prescaler_lo is enabled).


**Reset Timer counter for Timer_hi:**

When the Timer counter for Timer_hi is resetted, the counter is resetted to '0'. 
Timer counter for Timer_hi is resetted, if any of the below conditions is satisfied:
 
- When CMP_CLR_BIT in the CFG_REG_LO CSR, counter of Timer_lo reaches TIMER_CMP_LO value and the counter of Timer_hi reaches TIMER_CMP_HI value
- When the RESET_BIT in the CFG_REG_HI CSR is '1'.
- When the RESET_HI in the TIMER_RESET_HI CSR is '1'.

**Disable Timer counter for Timer_lo and Timer_hi:**

When the Timer counter for the Timer_lo and Timer_hi are disabled, it will pause its operation, that is the counters of Timer_lo and Timer_hi will not be reset to '0'.
Timer counter of both Timer_lo and Timer_hi are disabled, if any of the below conditions is satisfied: 

- When ENABLE_BIT of CFG_REG_LO CSR is '0'.
- When the input signal stoptimer_i is '1'.

Important Note:
^^^^^^^^^^^^^^^^
- For 64 bit mode, if the MODE_MTIME_BIT is '1', then issue an interrupt irq_lo_o irrespective of whether the interrupt is enabled or disabled through the IRQ_BIT.
- For 64 bit mode, busy_o pin is used to provide the status of APB Timer and it will be driven high if the 64 bit Timer is enabled.

Modes of Timer:
~~~~~~~~~~~~~~~

One shot mode:
^^^^^^^^^^^^^^^

For 32-bit timer, One shot mode can be enabled parallely for both the Timer_lo and Timer_hi.
If ONE_SHOT_BIT bitfield of CFG_REG_LO CSR is '1' then One shot mode is enabled for Timer_lo and the Timer_lo will be disabled when the Timer_lo counter reaches the TIMER_CMP_LO for the first time. 
Similarly, if ONE_SHOT_BIT bitfield of CFG_REG_HI CSR is '1' then One shot mode is enabled for Timer_hi and the Timer_hi will be disabled when the Timer_hi counter reaches the TIMER_CMP_HI for the first time.

For 64-bit timer,If ONE_SHOT_BIT bitfield of CFG_REG_LO CSR is '1' then One shot mode is enabled 
64 bit timer i.e cascaded Timer_lo and Timer_hi will be disabled when the Timer_lo counter reaches TIMER_CMP_LO and the Timer_hi counter reaches TIMER_CMP_HI for the first time.

Compare clear mode:
^^^^^^^^^^^^^^^^^^^^
For 32 bit mode, Compare clear mode can be enabled parallely for Timer_lo and Timer_hi

For 32 bit Timer_lo APB Timer, Compare clear mode is enabled If CMP_CLR_BIT bitfield of CFG_REG_LO CSR is '1', when the counter reaches the TIMER_CMP_LO, the timer is not disabled instead the counter will be reset to '0'.
As the timer is still enabled, the counter will be incremented  by '1' for every positive edge of the clock until it reaches the TIMER_CMP_LO. The same process is repeated.

For 32 bit Timer_hi APB Timer, Compare clear mode is enabled If CMP_CLR_BIT bitfield of CFG_REG_HI CSR is '1', when the counter reaches the TIMER_CMP_HI, the timer is not disabled instead the counter will be reset to '0'.
As the timer is still enabled, the  counter will be incremented  by '1' for every positive edge of the clock until it reaches the TIMER_CMP_HI. The same process is repeated.


For 64 bit Timer, When the counter of Timer_lo APB Timer reaches the TIMER_CMP_LO and the counter of Timer_hi APB Timer reaches the TIMER_CMP_HI, the timer is not disabled instead Timer_lo counter and Timer_hi counter will be reset to '0'.
As the Timer_lo and Timer_hi timers are still enabled, the Timer_lo counter will be incremented  by '1' for every positive edge of the clock until Timer_lo counter reaches the TIMER_CMP_LO and the Timer_hi counter reaches the TIMER_CMP_HI. The same process is repeated.


System Architecture:
--------------------

The figure below depicts the connections between the APB TIMER and rest of the modules in Core-V-MCU:-

.. figure:: apb_timer_soc_connections.png
   :name: APB Timer SoC Connections
   :align: center
   :alt:

   APB TIMER Core-V-MCU connections diagram

- The event_lo_i and event_hi_i input to the APB Timer is provided by APB_EVENT_GENERATOR. 
- APB Timer processes this input signals based on the various CSR configurations.
- APB Timer generates few output event signals that are further passed as interrupts to the Core complex.
- APB Timer receives the input stoptimer_i from the core complex that can stop the operations of APB TIMER.

Programmers View:
-----------------

Initial Configurations:
~~~~~~~~~~~~~~~~~~~~~~~
There are CSR bitfields in the APB timer that are required to be configured before any operations are initiated. 
As we have 2 Timer modules that can be configured individually. Each timer has to be configured with appropriate values.

-  Mode selection of 32 bit or 64 bit counters by configuring the MODE_64_BIT in CFG_REG_LO or CFG_REG_HI CSR.
-  Enable or disable the ref_clk by configuring the REF_CLK_EN_BIT in CFG_REG_LO or CFG_REG_HI CSR.
-  Enable or disable the prescaler by configuring the PRESCALER_EN_BIT in CFG_REG_LO or CFG_REG_HI CSR.
-  Prescaler compare values can be configured by using the PRESCALER_COMP in CFG_REG_LO or CFG_REG_HI CSR.
-  One shot mode can be enabled or disabled by configuring the ONE_SHOT_BIT in CFG_REG_LO or CFG_REG_HI CSR.
-  Compare clear mode can be enabled or disabled by configuring the CMP_CLR_BIT in CFG_REG_LO or CFG_REG_HI CSR.
-  event input can be enabled or disabled by configuring the IEM_BIT in CFG_REG_LO or CFG_REG_HI CSR.
-  Configure the MODE_MTIME_BIT bit so that in the 64 bit mode even if the IRQ_bit is not set an interrupt is being driven when the count == compare_value. Configure the MODE_MTIME_BIT in CFG_REG_LO or CFG_REG_HI CSR.
-  Overwriting the counter value directly via the by configuring the TIMER_VAL_LO or TIMER_VAL_HI CSR.
-  Initial counter value can be configured by using the TIMER_VAL_LO or TIMER_VAL_HI CSR.
-  Timer compare value can be configured by using the TIMER_CMP_LO or TIMER_CMP_HI CSR.
-  stoptimer_i is used to stop the counter operation of both the Timer_lo and Timer_hi directly.

Control configurations/operations:
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

There are CSR bitfields in the APB advanced timer which controls operations of each of the timer modules and its sub modules. 

- set the ENABLE_BIT in CFG_REG_LO or CFG_REG_HI CSR so that Timer_lo or Timer_hi can start counting. 
- set the START_LO or START_HI in TIMER_START_LO or TIMER_START_HI CSR respectively so that Timer_lo or Timer_hi can start counting. 
- set the RESET_BIT in CFG_REG_LO or CFG_REG_HI CSR so that Timer_lo or Timer_hi can be resetted.
- set the RESET_LO or RESET_HI in TIMER_RESET_LO or TIMER_RESET_HI CSR respectively so that Timer_lo or Timer_hi can be resetted.

Status configurations:
~~~~~~~~~~~~~~~~~~~~~~

The counter values of all the 4 Timers can be read via the following CSR bitfields in the APB advanced timer. 

- Use the TIMER_VAL_LO or TIMER_VAL_HI CSR for the current value of the Timer_lo counter or Timer_hi counter respectively.
- busy_o pin is used to provide the status of APB Timer will be driven high if anyone of the counter is enabled..

APB Timer CSRs
------------------

Refer to  `Memory Map <https://github.com/openhwgroup/core-v-mcu/blob/master/docs/doc-src/mmap.rst>`_ for the peripheral domain address of the SoC Controller.

NOTE: Several of the APB Timer CSRs are volatile, meaning that their read value may be changed by the hardware.
For example, writing the TIMER_VAL_LO CSR will set the initial counter value of Timer_lo. A subsequent read will return the latest updated counter value of Timer_lo. 
As the name suggests, the value of non-volatile CSRs is not changed by the hardware. These CSRs retain the last value written by the software.
A CSR's volatility is indicated by its "type".

Details of CSR access type are explained `here <https://docs.openhwgroup.org/projects/core-v-mcu/doc-src/mmap.html#csr-access-types>`_.

CFG_REG_LO 
~~~~~~~~~~ 

- Address Offset = 0x000
- Type: volatile

+------------------+-------+--------+---------+--------------------------------+
|     Field        | Bits  | Access | Default |         Description            |
+==================+=======+========+=========+================================+
| MODE_64_BIT      | 31:31 |   RW   |   0x0   | 1 = 64-bit mode, 0=32-bit mode |
+------------------+-------+--------+---------+--------------------------------+
| MODE_MTIME_BIT   | 30:30 |   RW   |   0x0   | 1= Generate the interrupt even |
|                  |       |        |         | if the IRQ_BIT is disabled.    |
+------------------+-------+--------+---------+--------------------------------+
| PRESCALER_COMP   | 15:8  |   RW   |   0x0   | Prescaler compare value        |
+------------------+-------+--------+---------+--------------------------------+
| REF_CLK_EN_BIT   |  7:7  |   RW   |   0x0   | 1= use Refclk for counter,     |
|                  |       |        |         | 0 = use APB bus clk for counter|
+------------------+-------+--------+---------+--------------------------------+
| PRESCALER_EN_BIT |  6:6  |   RW   |   0x0   | 1= Use prescaler               |
|                  |       |        |         | 0= no prescaler                |
+------------------+-------+--------+---------+--------------------------------+
| ONE_SHOT_BIT     |  5:5  |   RW   |   0x0   | 1= disable timer when Timer    |
|                  |       |        |         | counter == TIMER_CMP_LO value  |
+------------------+-------+--------+---------+--------------------------------+
| CMP_CLR_BIT      |  4:4  |   RW   |   0x0   | 1=counter is reset once Timer  |
|                  |       |        |         | counter == TIMER_CMP_LO,       |
|                  |       |        |         | 0 = counter is not reset       |
+------------------+-------+--------+---------+--------------------------------+
| IEM_BIT          |  3:3  |   RW   |   0x0   | 1 = event input is enabled     |
+------------------+-------+--------+---------+--------------------------------+
| IRQ_BIT          |  2:2  |   RW   |   0x0   | 1 = IRQ is enabled when Timer  |
|                  |       |        |         | counter ==TIMER_CMP_LO         |
+------------------+-------+--------+---------+--------------------------------+
| RESET_BIT        |  1:1  |   RW   |   0x0   | 1 = reset the counter          |
+------------------+-------+--------+---------+--------------------------------+
| ENABLE_BIT       |  0:0  |   RW   |   0x0   | 1 = enable the counter to count|
+------------------+-------+--------+---------+--------------------------------+

CFG_REG_HI 
~~~~~~~~~~ 

- Address Offset = 0x004
- Type: volatile

+------------------+-------+--------+---------+--------------------------------+
|     Field        | Bits  | Access | Default |         Description            |
+==================+=======+========+=========+================================+
| MODE_64_BIT      | 31:31 |   RW   |   0x0   | 1 = 64-bit mode, 0=32-bit mode |
+------------------+-------+--------+---------+--------------------------------+
| MODE_MTIME_BIT   | 30:30 |   RW   |   0x0   | 1= Generate the interrupt even |
|                  |       |        |         | if the IRQ_BIT is disabled.    |
+------------------+-------+--------+---------+--------------------------------+
| PRESCALER_COMP   | 15:8  |   RW   |   0x0   | Prescaler compare value        |
+------------------+-------+--------+---------+--------------------------------+
| REF_CLK_EN_BIT   |  7:7  |   RW   |   0x0   | 1= use Refclk for counter,     |
|                  |       |        |         | 0 = use APB bus clk for counter|
+------------------+-------+--------+---------+--------------------------------+
| PRESCALER_EN_BIT |  6:6  |   RW   |   0x0   | 1= Use prescaler               |
|                  |       |        |         | 0= no prescaler                |
+------------------+-------+--------+---------+--------------------------------+
| ONE_SHOT_BIT     |  5:5  |   RW   |   0x0   | 1= disable timer when Timer    |
|                  |       |        |         | counter == TIMER_CMP_HI value  |
+------------------+-------+--------+---------+--------------------------------+
| CMP_CLR_BIT      |  4:4  |   RW   |   0x0   | 1=counter is reset once Timer  |
|                  |       |        |         | counter == TIMER_CMP_HI,       |
|                  |       |        |         | 0=counter is not reset         |
+------------------+-------+--------+---------+--------------------------------+
| IEM_BIT          |  3:3  |   RW   |   0x0   | 1 = event input is enabled     |
+------------------+-------+--------+---------+--------------------------------+
| IRQ_BIT          |  2:2  |   RW   |   0x0   | 1 = IRQ is enabled when Timer  |
|                  |       |        |         | counter ==TIMER_CMP_HI value   |
+------------------+-------+--------+---------+--------------------------------+
| RESET_BIT        |  1:1  |   RW   |   0x0   | 1 = reset the counter          |
+------------------+-------+--------+---------+--------------------------------+
| ENABLE_BIT       |  0:0  |   RW   |   0x0   | 1 = enable the counter to count|
+------------------+-------+--------+---------+--------------------------------+

TIMER_VAL_LO 
~~~~~~~~~~~~ 

- Address Offset = 0x008
- Type: volatile

+-----------------+------+--------+---------+-----------------------------+
|     Field       | Bits | Access | Default |        Description          |
+=================+======+========+=========+=============================+
| TIMER_VAL_LO    | 31:0 |   RW   |   0x0   | 32-bit counter value        |
|                 |      |        |         |                             |
|                 |      |        |         | Low 32-bits in 64-bit mode  |
+-----------------+------+--------+---------+-----------------------------+

TIMER_VAL_HI 
~~~~~~~~~~~~ 

- Address Offset = 0x00C
- Type: volatile

+-----------------+------+--------+---------+-----------------------------+
|     Field       | Bits | Access | Default |        Description          |
+=================+======+========+=========+=============================+
| TIMER_VAL_HI    | 31:0 |   RW   |   0x0   | 32-bit counter value        |
|                 |      |        |         |                             |
|                 |      |        |         | High 32-bits in 64-bit mode |
+-----------------+------+--------+---------+-----------------------------+

TIMER_CMP_LO 
~~~~~~~~~~~~ 

- Address Offset = 0x010
- Type: non-volatile

+-----------------+------+--------+---------+-----------------------------+
|     Field       | Bits | Access | Default |        Description          |
+=================+======+========+=========+=============================+
| TIMER_CMP_LO    | 31:0 |   RW   |   0x0   | compare value for low       |
|                 |      |        |         | 32-bit counter              |
+-----------------+------+--------+---------+-----------------------------+

TIMER_CMP_HI 
~~~~~~~~~~~~ 

- Address Offset = 0x014
- Type: non-volatile

+-----------------+------+--------+---------+-----------------------------+
|     Field       | Bits | Access | Default |        Description          |
+=================+======+========+=========+=============================+
| TIMER_CMP_HI    | 31:0 |   RW   |   0x0   | compare value for high      |
|                 |      |        |         | 32-bit counter              |
+-----------------+------+--------+---------+-----------------------------+

TIMER_START_LO 
~~~~~~~~~~~~~~ 

- Address Offset = 0x018
- Type: non-volatile

+-----------------+------+--------+---------+-----------------------------+
|     Field       | Bits | Access | Default |        Description          |
+=================+======+========+=========+=============================+
| START_LO        | 31:0 |  WS    |   0x0   | Start Timer_lo APB Timer    |
|                 |      |        |         |                             |
+-----------------+------+--------+---------+-----------------------------+

TIMER_START_HI 
~~~~~~~~~~~~~~ 

- Address Offset = 0x01C
- Type: non-volatile

+-----------------+------+--------+---------+-----------------------------+
|     Field       | Bits | Access | Default |        Description          |
+=================+======+========+=========+=============================+
| START_HI        | 31:0 |  WS    |   0x0   | Start Timer_hi APB Timer    |
|                 |      |        |         |                             |
+-----------------+------+--------+---------+-----------------------------+

TIMER_RESET_LO 
~~~~~~~~~~~~~~ 

- Address Offset = 0x020
- Type: non-volatile

+-----------------+------+--------+---------+-----------------------------+
|     Field       | Bits | Access | Default |        Description          |
+=================+======+========+=========+=============================+
| RESET_LO        | 31:0 |  WS    |   0x0   | Reset Timer_lo APB Timer    |
|                 |      |        |         |                             |
+-----------------+------+--------+---------+-----------------------------+

TIMER_RESET_HI 
~~~~~~~~~~~~~~ 

- Address Offset = 0x024
- Type: non-volatile

+-----------------+------+--------+---------+-----------------------------+
|     Field       | Bits | Access | Default |        Description          |
+=================+======+========+=========+=============================+
| RESET_HI        | 31:0 |  WS    |   0x0   | Reset Timer_hi APB Timer    |
|                 |      |        |         |                             |
+-----------------+------+--------+---------+-----------------------------+

Firmware Guidelines
-------------------

Initialization:
~~~~~~~~~~~~~~~
- When the HRESETn signal is low, CSRs default to 0 and outputs are low.
- At every positive edge of the clock the CSRs are updated based on APB signals.
- FW can update the below bitfields to any custom value before the START bitfield in the REG_TIM[0-3]_CMD CSR is '1' and the timer is not active yet (which means the timer is started for the first time). Otherwise, all the config values of all sub-modules are commanded to be updated to default .


Initializing the Prescaler:
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

- By default prescalers are disabled, set PRESCALER_EN_BIT bitfield of CFG_REG_LO or CFG_REG_HI CSRs to enable the prescaler_lo or prescaler_hi respectively. 

- If the Prescalers are enabled, Write to the PRESCALER_COUNT bitfield of CFG_REG_LO or CFG_REG_HI CSRs to specify the compare value for the prescaler_lo or prescaler_hi respectively. 

Initializing the Timer counter:
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

- Write to CSRs TIMER_VAL_LO and TIMER_VAL_HI to specify the initial counter value for Timer_lo or Timer_hi respectively. 

- Write to CSRs TIMER_CMP_LO and TIMER_CMP_HI to specify the compare count for Timer_lo or Timer_hi respectively. 

- Write '1' to either ONE_SHOT_BIT bitfield or CMP_CLR_BIT bitfield of CFG_REG_LO or CFG_REG_HI CSRs to select the mode for Timer_lo or Timer_hi respectively. 

- By default interrupts are disabled, set IRQ_BIT bitfield of CFG_REG_LO or CFG_REG_HI CSRs to enable the interrupts irq_lo_o or irq_hi_o for Timer_lo or Timer_hi respectively. 

- By default input events are disabled, set IEM_BIT bitfield of CFG_REG_LO or CFG_REG_HI CSRs to enable the input events event_lo_i or event_hi_i for Timer_lo or Timer_hi respectively. 

- By default reference clock is disabled, set REF_CLK_EN_BIT bitfield of CFG_REG_LO or CFG_REG_HI CSRs to enable the reference clocks for Timer_lo or Timer_hi respectively. 

- By default 64 bit mode is disabled and 32 bit mode is enabled, set MODE_64_BIT bitfield of CFG_REG_LO  CSR to enable the 64 bit mode. 

- Set MODE_MTIME_BIT bitfield of CFG_REG_LO or CFG_REG_HI CSRs to generate interrupt irq_lo_o for 64 bit timer irrespective of whether the interrupt is enabled or disabled through the IRQ_BIT.


Start the timer:
~~~~~~~~~~~~~~~~
- Timer can be started in the following 3 ways:
   - when ENABLE_BIT of CFG_REG_LO or CFG_REG_HI is '1'.
   - when the event_lo_i or event_hi_i signal is read as '1' and IEM_BIT of CFG_REG_LO or CFG_REG_HI is '1'.
   - when TIMER_START_LO or TIMER_START_HI is any value other than '0'.
- Once the timer is started FW can observe the counter value getting incremented in the TIMER_VAL_LO or TIMER_VAL_HI CSR.

Stop the timer:
~~~~~~~~~~~~~~~~
- Timer can be stopped in the following 2 ways:
   - when ENABLE_BIT of CFG_REG_LO or CFG_REG_HI is '0'.
   - when stoptimer_i is '1'.
- Once the timer is stopped FW can observe the counter value remain the same in the TIMER_VAL_LO or TIMER_VAL_HI CSR.

Reset the timer:
~~~~~~~~~~~~~~~~
- Timer can be resetted in the following 2 ways:
   - When the RESET_BIT in the CFG_REG_LO or CFG_REG_HI CSR is '1'.
   - When the RESET_LO in the TIMER_RESET_LO or TIMER_RESET_HI CSR is '1'.
- Once the timer is stopped FW can observe the counter value resetted to '0' in the TIMER_VAL_LO or TIMER_VAL_HI CSR.


Interrupt generation:
~~~~~~~~~~~~~~~~~~~~~
- If the IRQ_BIT of  CFG_REG_LO is '1' , irq_lo_o will be asserted when the counter value of Timer_lo reaches the TIMER_CMP_LO.
- If the IRQ_BIT of  CFG_REG_HI is '1' , irq_hi_o will be asserted when the counter value of Timer_hi reaches the TIMER_CMP_HI.


Pin Diagram
-----------

The figure below represents the input and output pins for the APB Timer:-

.. figure:: apb_timer_pin_diagram.png
   :name: APB_Timer_Pin_Diagram
   :align: center
   :alt:
   
APB Timer Pin Diagram

Clock and Reset
~~~~~~~~~~~~~~~
- HCLK: System clock input
- HRESETn: Active-low reset input
- low_speed_clk_i: Reference clock input

APB Interface
~~~~~~~~~~~~~
- PADDR[11:0]: APB address bus input
- PSEL: APB peripheral select input
- PENABLE: APB enable input
- PWRITE: APB write control input (high for write, low for read)
- PWDATA[31:0]: APB write data bus input
- PREADY: APB ready output to indicate transfer completion
- PRDATA[31:0]: APB read data bus output
- PSLVERR: APB slave error

APB Event generator Interface
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
- event_lo_i: Input event for the Timer_lo
- event_hi_i: Input event for the Timer_hi

Core Complex Interface
~~~~~~~~~~~~~~~~~~~~~~
- stoptimer_i: Input signal to stop timer
- irq_lo_o: Output interrupt from Timer_lo
- irq_hi_o: Output interrupt from Timer_hi
- busy_o: Output busy signal that signifies that any one of the timer is active