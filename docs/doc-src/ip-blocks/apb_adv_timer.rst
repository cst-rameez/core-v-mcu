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
.. _apb_advanced_timer:

APB Advanced Timer
==================

The Advanced Timer supports four programmable timers called "channels", typically used for PWM generation. These four timers can be configured independently to support four unique PWM generation parallely.   

Features
--------

- Multiple trigger input sources:

  - Output signal channels of all timers
  - 32 GPIOs
  - Reference clock at 32kHz
  - FLL clock
  
- Configurable input trigger modes for each timer
- Configurable prescaler for each timer
- Configurable counting mode for each timer
- Configurable channel threshold action for each timer
- Four configurable output events
- Configurable clock gating of each timer

Architecture
-------------

The figure below is a high-level block diagram of the APB Advanced Timer module:-

.. figure:: apb_adv_timer_image1.png
   :name: APB_ADVANCED_TIMER_Block_Diagram
   :align: center
   :alt:

   APB ADVANCED TIMER Block Diagram

The APB ADVANCED TIMER IP consists of the following key components:
APB control logic, APB ADVANCED TIMER Registers and 4 Timer modules

APB control logic
~~~~~~~~~~~~~~~~~
The APB control logic interfaces with the APB bus to decode and execute commands.
It handles register reads and writes according to the APB protocol, providing a standardized interface to the system.

APB ADVANCED TIMER Registers
~~~~~~~~~~~~~~~~~~~~~~~~~~~~
There are few common registers that are store the configurations
  - Event select  
  - Clock enable
There are 4 timer modules and each timer module has its own set of registers
Each of the timer module specific registers store the following configuration:
  - Arm, Reset, Update, Stop and Start  
  - Prescalar value, Updownsel, Clksel, Input trigger mode select, Input pins select
  - Threshold high and Threshold low
  - Counter 
  - Output trigger mode 

Timer Module
~~~~~~~~~~~~
.. figure:: apb_adv_timer_image2.png
   :name: TIMER_Block_Diagram
   :align: center
   :alt:

   TIMER Block Diagram

Timer module has various submodules/components like Timer Controller, Input stage, Prescalar, Updown counter and Comparators. 

Theory of Operation
^^^^^^^^^^^^^^^^^^^
  - FW drives various configuration register and external input signals
  - Input stage selects and processes the appropriate input signal. These signal are passed to prescalar.
  - Prescaler scales up the input signals and process it further. The resultant information is passed to the updowncounter.
  - Updowncounter process it and the generates the output event. Then it sends some information to 4 comparators 
  - Comparators process them and generate the final output events.   

Input Stage
^^^^^^^^^^^
- Based on the change in the config register CLKSEL ,it is decided
  whether the input selected from the set of inputs in ext_sig_i will
  be either in sync with the rising edge of the
  low_speed_clk_i or in sync with the ref clock.

- At every positive edge of the clock,the input signal is selected from a set of signals in ext_sig_i using the config register INSEL value and how the events are generated from the signal is decided by the config register MODE.

    ○ If MODE is 3’b000

    ■ The event is always high

    ○ If MODE is 3’b001

    ■ The event is sensitive to the negation of the signal selected

    ○ If MODE is 3’b010

    ■ The output event is sensitive to the input signal selected
    
    ○ If MODE is 3’b011

    ■ The output event is sensitive to the rising edge of the selected
    signal in sync with the clock.

    ○ If MODE is 3’b100

    ■ The output event is sensitive to the rising edge of the selected
    signal in sync with the clock.

    ○ If MODE is 3’b101

    ■ The output event is sensitive to both rising edge and falling
    edge of the selected signal in sync with the clock.

    ○ If MODE is 3’b110

    ■ If the timer is armed ,i,e,the register ARM is high then the
    event is made high for the rising edge of the selected signal and
    remains the same until the
    next rising edge of the signal.If ARM register is low,then the output
    event is low forever.

    ○ If MODE is 3’b111

    ■ If the timer is armed ,i,e,the register ARM is high then the
    event is made high for the falling edge of the selected signal and
    remains the same until the next falling edge of the signal.If ARM
    register is low,then the output event is low forever.

Prescalar
^^^^^^^^^
- The Event signal is scaled based on the
  prescaler value(PRESC register value). At every positive edge of the
  clock the register PRESC is
  updated.The scaling happens in a way that after every time the
  number of events in sync with the external clock
  generated is equal to the PRESC register value then counter is made
  to 0 and an event is generated.Like this whenever the lock synced
  events generated is equal to PRESC value then one output event is
  generated at positive edge of the clock(the frequency is scaled
  according to the PRESC
  register value).

Updown counter
^^^^^^^^^^^^^^
- For every event the counter is incremented starting from
  the start value(TH_LO register) .Based on the register UPDOWNCLK
  representing the sawtooth mode,it is decided whether the counter
  should reset after reaching end of the counting range (TH_HI) or it
  should reverse the
  direction and go counting down to start value(TH_LO) after
  which it resets to the default values of start,stop,direction,etc .

- At every input event in sync with the clock an **output event** is
  generated and also the counter is incremented .Whenever the counter
  reaches the end of a counting range an event is generated
  representing the end of the counter and reset happens.The output port
  representing the counter is updated at every clock positive edge.

- Here, the counter value,event representing the end of the
  timer,the **output event**  are generated.


Comparator
^^^^^^^^^^
- At every positive edge of the clock,When the timer is started the
  first time or explicitly updated through the update
  command register named UPDATE, the module is updated then, the
  register values TH and MODE of the register are read in which TH
  value is the comparator threshold value and MODE is the operation
  that should be done when counter of the up down counter reaches the
  comparator threshold value.

- At every positive edge of the clock when the event coming out of
  the up down counter is high ,based on the register MODE value ,output
  is generated accordingly.

- There are two events that can happen in the comparator, 

  ○ When
  timer counter value reaches the comparator offset **(match event)**

  ○ When the UPDOWNSEL register is high and the timer reaches its end
  or when UPDOWNSEL is low and the timer counter value reaches the
  comparator threshold offset. **(event_2)**
- define OP_SET 3'b000
- define OP_TOGRST 3'b001
- define OP_SETRST 3'b010
- define OP_TOG 3'b011
- define OP_RST 3'b100
- define OP_TOGSET 3'b101
- define OP_RSTSET 3'b110
- If MODE value is OP_SET
   ○ Then the output event is high when there is a match otherwise
   remains the same .

- If MODE value is OP_TOGRST
   ○ Then if sawtooth mode is on ,then if a match happens then the
   output event is toggled else if event_2
   happens then output event is low.

   ○ If sawtooth mode is off,then if match event happens and event_2
   doesn't happen then output event is toggle and event_2 is made high
   ,else if match event happens and event_2 also happens then output
   event is made low and event_2 is also made low.

- If MODE value is OP_SETRST
   ○ Then if sawtooth mode is on ,then if a match happens then the
   output event is high else if event_2 happens then output event is
   low.

   ○ If sawtooth mode is off,then if match event happens and event_2
   doesn't happen then output event is made high and event_2 is made
   high.,else if match event happens and event_2 also happens then
   output event is made low and event_2 is also made low.

- If MODE value is OP_TOG

   ○ Then the output event is toggled when the match event occurs else
   remains the same.

- If MODE value is OP_RST
   ○ Then the output event is made low when the match event occurs
   else remains the same.

- If MODE value is OP_TOGSET
   ○ Then if sawtooth mode is on ,then if a match happens then the
   output event is toggled else if event_2
   happens then output event is high.

   ○ If sawtooth mode is off,then if match event happens and event_2
   doesn't happen then output event is toggle and event_2 is made high
   ,else if match event happens and event_2 also happens then output
   event is made high and event_2 is also made low.

- If MODE value is OP_RSTSET
    ○ Then if sawtooth mode is on ,then if a match happens then the
    output event is low else if event_2 happens then output event is
    high.

    ○ If sawtooth mode is off,then if match event happens and event_2
    doesn't happen then output event is made low and event_2 is made
    high.,else if match event happens and event_2 also happens then the
    output event is made high and event_2 is also made low.

- By default the output event remains the same (state remains same
  until further change in input) and event_2 is kept low.


APB ADVANCED CSRs
-----------------
**REG_TIM0_CMD** offset=0x000

.. list-table::
   :widths: 10 10 10 10 50
   :header-rows: 1

   * - Field
     - Bits
     - Type
     - Access
     - Description
   * - Reserved
     - 31:5
     - 
     - 
     - 
   * - ARM
     - 4:4
     - Config
     - R/W
     - arm command bitfield
   * - RESET
     - 3:3
     - Config
     - R/W
     - reset command bitfield
   * - UPDATE
     - 2:2
     - Config
     - R/W
     - update command bitfield
   * - STOP
     - 1:1
     - Config
     - R/W
     - Stop command field
   * - START
     - 0:0
     - Config
     - R/W
     - Start command field
..

**REG_TIM0_CFG** offset=0x004

.. list-table::
   :widths: 10 10 10 10 50
   :header-rows: 1

   * - Field
     - Bits
     - Type
     - Access
     - Description
   * - Reserved
     - 31:24
     - 
     - 
     - 
   * - PRESC
     - 23:16
     - Config
     - R/W
     - prescaler value configuration bitfield
   * - Reserved
     - 15:13
     - 
     - 
     - 
   * - UPDOWNSEL
     - 12:12
     - Config
     - R/W
     - center-aligned mode configuration bitfield
   * -
     -
     -
     -
     - 1’b0: The counter counts up and down alternatively
   * -
     -
     -
     -
     - 1’b1: The counter counts up and resets to 0 when it reaches the threshold.
   * - CLKSEL
     - 11:11
     - Config
     - R/W
     - clock source configuration bitfield
   * -
     -
     -
     -
     - 1’b0: FLL
   * -
     -
     -
     -
     - 1’b1: Reference clock at 32kHz
   * - MODE
     - 10:8
     - Config
     - R/W
     - trigger mode configuration bitfield
   * -
     -
     -
     -
     - 3’h0: Trigger event at each clock cycle
   * -
     -
     -
     -
     - 3’h1: Trigger event if input source is 0
   * -
     -
     -
     -
     - 3’h2: Trigger event if input source is 1
   * -
     -
     -
     -
     - 3’h3: Trigger event on input source rising edge
   * -
     -
     -
     -
     - 3’h4: Trigger event on input source falling edge
   * -
     -
     -
     -
     - 3’h5: Trigger event on input source falling or rising edge
   * -
     -
     -
     -
     - 3’h6: Trigger event on input source rising edge when armed
   * -
     -
     -
     -
     - 3’h7: Trigger event on input source falling edge when armed
   * - INSEL
     - 7:0
     - Config
     - R/W
     - input source configuration bitfield
   * -
     -
     -
     -
     - 0-31: GPIO[0] to GPIO[31]
   * -
     -
     -
     -
     - 32-35: Channel 0 to 3 of ADV_TIMER0
   * -
     -
     -
     -
     - 36-39: Channel 0 to 3 of ADV_TIMER1
   * -
     -
     -
     -
     - 40-43: Channel 0 to 3 of ADV_TIMER2
   * -
     -
     -
     -
     - 44-47: Channel 0 to 3 of ADV_TIMER3
..


**REG_TIM0_TH** offset=0x008

.. list-table::
   :widths: 10 10 10 10 50
   :header-rows: 1

   * - Field
     - Bits
     - Type
     - Access
     - Description
   * - TH_HI
     - 31:16
     - Config
     - R/W
     - Threshold high part configuration bitfield
   * - TH_LO
     - 15:0
     - Config
     - R/W
     - Threshold low part configuration bitfield

..

**REG_TIM0_CH0_TH** offset=0x00C

.. list-table::
   :widths: 10 10 10 10 50
   :header-rows: 1

   * - Field
     - Bits
     - Type
     - Access
     - Description
   * - Reserved
     - 31:19
     - 
     - 
     - 
   * - MODE
     - 18:16
     - Config
     - R/W
     - Channel 0 threshold match action on channel output signal configuration bitfield
   * -
     -
     -
     -     
     - 3’h0: Set
   * -
     -
     -
     -     
     - 3’h1: Toggle then next threshold match action is clear
   * -
     -
     -
     - 
     - 3’h2: Set then next threshold match action is clear
   * -
     -
     -
     -
     - 3’h3: Toggle
   * -
     -
     -
     -
     - 3’h4: Clear
   * -
     -
     -
     -
     - 3’h5: Toggle then next threshold match action is set
   * -
     -
     -
     -
     - 3’h6: Clear then next threshold match action is set
   * - TH
     - 15:0
     - Config
     - R/W
     - Channel 0 threshold configuration bitfield

..

**REG_TIM0_CH1_TH** offset=0x010

.. list-table::
   :widths: 10 10 10 10 50
   :header-rows: 1

   * - Field
     - Bits
     - Type
     - Access
     - Description
   * - Reserved
     - 31:19
     - 
     - 
     - 
   * - MODE
     - 18:16
     - Config
     - R/W
     - Channel 1 threshold match action on channel output signal configuration bitfield
   * -
     -
     -
     -     
     - 3’h0: Set
   * -
     -
     -
     -
     - 3’h1: Toggle then next threshold match action is clear
   * -
     -
     -
     -
     - 3’h2: Set then next threshold match action is clear
   * -
     -
     -
     -
     - 3’h3: Toggle
   * -
     -
     -
     -
     - 3’h4: Clear
   * -
     -
     -
     -
     - 3’h5: Toggle then next threshold match action is set
   * -
     -
     -
     -
     - 3’h6: Clear then next threshold match action is set
   * - TH
     - 15:0
     - Config
     - R/W
     - Channel 1 threshold configuration bitfield

..

**REG_TIM0_CH2_TH** offset=0x014

.. list-table::
   :widths: 10 10 10 10 50
   :header-rows: 1

   * - Field
     - Bits
     - Type
     - Access
     - Description
   * - Reserved
     - 31:19
     - 
     - 
     - 
   * - MODE
     - 18:16
     - Config
     - R/W
     - Channel 2 threshold match action on channel output signal configuration bitfield
   * -
     -
     -
     -     
     - 3’h0: Set
   * -
     -
     -
     -     
     - 3’h1: Toggle then next threshold match action is clear
   * -
     -
     -
     -     
     - 3’h2: Set then next threshold match action is clear
   * -
     -
     -
     -     
     - 3’h3: Toggle
   * -
     -
     -
     -    
     - 3’h4: Clear
   * -
     -
     -
     -     
     - 3’h5: Toggle then next threshold match action is set
   * -
     -
     -
     -    
     - 3’h6: Clear then next threshold match action is set
   * - TH
     - 15:0
     - Config
     - R/W
     - Channel 2 threshold configuration bitfield

..

**REG_TIM0_CH3_TH** offset=0x018

.. list-table::
   :widths: 10 10 10 10 50
   :header-rows: 1

   * - Field
     - Bits
     - Type
     - Access
     - Description
   * - Reserved
     - 31:19
     - 
     - 
     - 
   * - MODE
     - 18:16
     - Config
     - R/W
     - Channel 3 threshold match action on channel output signal configuration bitfield
   * -
     -
     -
     -
     - 3’h0: Set
   * -
     -
     -
     -
     - 3’h1: Toggle then next threshold match action is clear
   * -
     -
     -
     -     
     - 3’h2: Set then next threshold match action is clear
   * -
     -
     -
     -     
     - 3’h3: Toggle
   * -
     -
     -
     -     
     - 3’h4: Clear
   * -
     -
     -
     -    
     - 3’h5: Toggle then next threshold match action is set
   * -
     -
     -
     -     
     - 3’h6: Clear then next threshold match action is set
   * - TH
     - 15:0
     - Config
     - R/W
     - Channel 3 threshold configuration bitfield

..

**REG_TIM0_COUNTER** offset=0x02C

.. list-table::
   :widths: 10 10 10 10 50
   :header-rows: 1

   * - Field
     - Bits
     - Type
     - Access
     - Description
   * - T0_COUNTER
     - 31:0
     - Status
     - R
     - ADV_TIMER0 counter register

Input Ports
+++++++++++

1. **HCLK**: External clock for synchronization
2. **HRESETn**: Reset pin to reset the timer
3. **APB bus pins**
4. **dft_cg_enable_i**
5. **low_speed_clk_i**
6. **ext_sig_i**: A set of 32 GPIOs

Output Ports
++++++++++++

1. **events_o[3:0]**
2. **ch0_o[3:0]**
3. **ch1_o[3:0]**
4. **ch2_o[3:0]**
5. **ch3_o[3:0]**
