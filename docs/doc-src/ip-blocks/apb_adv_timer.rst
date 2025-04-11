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
There are few common registers that store the following configurations
  - Output event select  
  - Clock enable

There are 4 timer modules and each timer module has its own set of registers. Each of the timer module specific registers store the following configuration:
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
FW drives various configuration register and when the external input/stimulus is provided, Input stage selects and processes the appropriate inputs.
Timer controller manages all the other submodules through few control signals like active, controller reset and update.
Prescalar further process the signals from input stage and scales it up according to the configured prescaler value and passes it to the Updown counter.
Depending on FW configurations of UPDOWNSEL and mode, Updown counter generates the output event and passes necassary information to 4 comparators.
Comparators process them and generate the final output events.  

Timer Controller
^^^^^^^^^^^^^^^^
- Timer controller updates the Timer module state to active if the START bitfield is 1 in the REG_TIM0_CMD register and it is not active when the START bitfield is '0' and STOP bitfield is '1'. 
- Timer controller parses the UPDATE bitfield to Updown counter and RESET bitfield to all sub modules if START bitfield is 0 in the REG_TIM0_CMD register.
- At every positive edge of the clock, Timer controller calculates If the Timer is active and Prescaled input event is high. then only it parses the PRESC bitfield in the REG_TIM0_CFG to the Prescaler sub module and 4 TH bitfield in the REG_TIM0_CH*_TH to the 4 comparator sub modules.  

Input Stage
^^^^^^^^^^^
- Input stage uses the bitfield INSEL in REG_TIM0_CFG register and selects a signal from a set of signals in ext_sig_i.
- Input stage uses the bitfield CLKSEL in REG_TIM0_CFG register and decides whether the input will be either in sync with the rising edge of the low_speed_clk_i or in sync with the ref clock.
- At every positive edge of the selected clock and selected input signal, Input stage uses the bitfield MODE in REG_TIM0_CFG register to generate output signal according to the below information.

    ○ If MODE is 3’b000

      ■ The event is always high

    ○ If MODE is 3’b001

      ■ The event is sensitive to the negation of the signal selected

    ○ If MODE is 3’b010

      ■ The output event is sensitive to the input signal selected
    
    ○ If MODE is 3’b011

      ■ The output event is sensitive to the rising edge of the selected signal in sync with the clock.

    ○ If MODE is 3’b100

      ■ The output event is sensitive to the rising edge of the selected signal in sync with the clock.

    ○ If MODE is 3’b101

      ■ The output event is sensitive to both rising edge and falling edge of the selected signal in sync with the clock.

    ○ If MODE is 3’b110

      ■ If the timer is armed ,i,e,the register ARM is high then the event is made high for the rising edge of the selected signal and remains the same until the next rising edge of the signal.If ARM register is low,then the output event is low forever.

    ○ If MODE is 3’b111

      ■ If the timer is armed ,i,e,the register ARM is high then the event is made high for the falling edge of the selected signal and remains the same until the next falling edge of the signal.If ARM register is low,then the output event is low forever.

Prescalar
^^^^^^^^^
- The PRESC bitfield in the REG_TIM0_CFG register is parsed to Prescaler by Timer controller. 
- Prescaler module maintains a counter whose initial value is 0. At every positive edge of the clock, counter gets incremented by 1 when event input signal is 1 and Timer is active.
- When the counter value matches with the PRESC bitfield output event is set to '1' and the counter is updated to '0'. The above process continues and output events are generated.
- Whenever the lock synced events generated is equal to PRESC value then one output event is generated at positive edge of the clock(the frequency is scaled according to the PRESC register value).
- Both the counter and output event is set to 0. When either the hard reset is triggered or when Timer controller parses the RESET bitfield which is set to '1'.

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

**REG_TIM0_CH0_LUT** offset=0x01C

.. list-table::
   :widths: 10 10 10 10 50
   :header-rows: 1

   * - Field
     - Bits
     - Type
     - Access
     - Description
   * - Reserved
     - 31:18
     - 
     - 
     - 
   * - FLT
     - 23:16
     - Config
     - R/W
     - FLT
   * - LUT
     - 15:0
     - Config
     - R/W
     - LUT

..

**REG_TIM0_CH1_LUT** offset=0x020

.. list-table::
   :widths: 10 10 10 10 50
   :header-rows: 1

   * - Field
     - Bits
     - Type
     - Access
     - Description
   * - Reserved
     - 31:18
     - 
     - 
     - 
   * - FLT
     - 23:16
     - Config
     - R/W
     - FLT
   * - LUT
     - 15:0
     - Config
     - R/W
     - LUT

..


**REG_TIM0_CH2_LUT** offset=0x024

.. list-table::
   :widths: 10 10 10 10 50
   :header-rows: 1

   * - Field
     - Bits
     - Type
     - Access
     - Description
   * - Reserved
     - 31:18
     - 
     - 
     - 
   * - FLT
     - 23:16
     - Config
     - R/W
     - FLT
   * - LUT
     - 15:0
     - Config
     - R/W
     - LUT

..

**REG_TIM0_CH3_LUT** offset=0x028

.. list-table::
   :widths: 10 10 10 10 50
   :header-rows: 1

   * - Field
     - Bits
     - Type
     - Access
     - Description
   * - Reserved
     - 31:18
     - 
     - 
     - 
   * - FLT
     - 23:16
     - Config
     - R/W
     - FLT
   * - LUT
     - 15:0
     - Config
     - R/W
     - LUT

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

..

 **REG_TIM1_CMD** offset=0x040

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

**REG_TIM1_CFG** offset=0x044

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


**REG_TIM1_TH** offset=0x048

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

**REG_TIM1_CH0_TH** offset=0x04C

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

**REG_TIM1_CH1_TH** offset=0x050

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

**REG_TIM1_CH2_TH** offset=0x054

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

**REG_TIM1_CH3_TH** offset=0x058

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

**REG_TIM1_CH0_LUT** offset=0x05C

.. list-table::
   :widths: 10 10 10 10 50
   :header-rows: 1

   * - Field
     - Bits
     - Type
     - Access
     - Description
   * - Reserved
     - 31:18
     - 
     - 
     - 
   * - FLT
     - 23:16
     - Config
     - R/W
     - FLT
   * - LUT
     - 15:0
     - Config
     - R/W
     - LUT

..

**REG_TIM1_CH1_LUT** offset=0x060

.. list-table::
   :widths: 10 10 10 10 50
   :header-rows: 1

   * - Field
     - Bits
     - Type
     - Access
     - Description
   * - Reserved
     - 31:18
     - 
     - 
     - 
   * - FLT
     - 23:16
     - Config
     - R/W
     - FLT
   * - LUT
     - 15:0
     - Config
     - R/W
     - LUT

..


**REG_TIM1_CH2_LUT** offset=0x064

.. list-table::
   :widths: 10 10 10 10 50
   :header-rows: 1

   * - Field
     - Bits
     - Type
     - Access
     - Description
   * - Reserved
     - 31:18
     - 
     - 
     - 
   * - FLT
     - 23:16
     - Config
     - R/W
     - FLT
   * - LUT
     - 15:0
     - Config
     - R/W
     - LUT

..

**REG_TIM1_CH3_LUT** offset=0x068

.. list-table::
   :widths: 10 10 10 10 50
   :header-rows: 1

   * - Field
     - Bits
     - Type
     - Access
     - Description
   * - Reserved
     - 31:18
     - 
     - 
     - 
   * - FLT
     - 23:16
     - Config
     - R/W
     - FLT
   * - LUT
     - 15:0
     - Config
     - R/W
     - LUT

..

**REG_TIM1_COUNTER** offset=0x06C

.. list-table::
   :widths: 10 10 10 10 50
   :header-rows: 1

   * - Field
     - Bits
     - Type
     - Access
     - Description
   * - T1_COUNTER
     - 31:0
     - Status
     - R
     - ADV_TIMER0 counter register

..

 **REG_TIM2_CMD** offset=0x080

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

**REG_TIM2_CFG** offset=0x084

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


**REG_TIM2_TH** offset=0x088

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

**REG_TIM2_CH0_TH** offset=0x08C

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

**REG_TIM2_CH1_TH** offset=0x090

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

**REG_TIM2_CH2_TH** offset=0x094

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

**REG_TIM2_CH3_TH** offset=0x098

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

**REG_TIM2_CH0_LUT** offset=0x09C

.. list-table::
   :widths: 10 10 10 10 50
   :header-rows: 1

   * - Field
     - Bits
     - Type
     - Access
     - Description
   * - Reserved
     - 31:18
     - 
     - 
     - 
   * - FLT
     - 23:16
     - Config
     - R/W
     - FLT
   * - LUT
     - 15:0
     - Config
     - R/W
     - LUT

..

**REG_TIM2_CH1_LUT** offset=0x0A0

.. list-table::
   :widths: 10 10 10 10 50
   :header-rows: 1

   * - Field
     - Bits
     - Type
     - Access
     - Description
   * - Reserved
     - 31:18
     - 
     - 
     - 
   * - FLT
     - 23:16
     - Config
     - R/W
     - FLT
   * - LUT
     - 15:0
     - Config
     - R/W
     - LUT

..


**REG_TIM2_CH2_LUT** offset=0x0A4

.. list-table::
   :widths: 10 10 10 10 50
   :header-rows: 1

   * - Field
     - Bits
     - Type
     - Access
     - Description
   * - Reserved
     - 31:18
     - 
     - 
     - 
   * - FLT
     - 23:16
     - Config
     - R/W
     - FLT
   * - LUT
     - 15:0
     - Config
     - R/W
     - LUT

..

**REG_TIM2_CH3_LUT** offset=0x0A8

.. list-table::
   :widths: 10 10 10 10 50
   :header-rows: 1

   * - Field
     - Bits
     - Type
     - Access
     - Description
   * - Reserved
     - 31:18
     - 
     - 
     - 
   * - FLT
     - 23:16
     - Config
     - R/W
     - FLT
   * - LUT
     - 15:0
     - Config
     - R/W
     - LUT

..

**REG_TIM2_COUNTER** offset=0x0AC

.. list-table::
   :widths: 10 10 10 10 50
   :header-rows: 1

   * - Field
     - Bits
     - Type
     - Access
     - Description
   * - T2_COUNTER
     - 31:0
     - Status
     - R
     - ADV_TIMER0 counter register

..
 **REG_TIM3_CMD** offset=0x0C0

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

**REG_TIM3_CFG** offset=0x0C4

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


**REG_TIM3_TH** offset=0x0C8

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

**REG_TIM3_CH0_TH** offset=0x0CC

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

**REG_TIM3_CH1_TH** offset=0x0D0

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

**REG_TIM3_CH2_TH** offset=0x0D4

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

**REG_TIM3_CH3_TH** offset=0x0D8

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

**REG_TIM3_CH0_LUT** offset=0x0DC

.. list-table::
   :widths: 10 10 10 10 50
   :header-rows: 1

   * - Field
     - Bits
     - Type
     - Access
     - Description
   * - Reserved
     - 31:18
     - 
     - 
     - 
   * - FLT
     - 23:16
     - Config
     - R/W
     - FLT
   * - LUT
     - 15:0
     - Config
     - R/W
     - LUT

..

**REG_TIM3_CH1_LUT** offset=0x0E0

.. list-table::
   :widths: 10 10 10 10 50
   :header-rows: 1

   * - Field
     - Bits
     - Type
     - Access
     - Description
   * - Reserved
     - 31:18
     - 
     - 
     - 
   * - FLT
     - 23:16
     - Config
     - R/W
     - FLT
   * - LUT
     - 15:0
     - Config
     - R/W
     - LUT

..


**REG_TIM3_CH2_LUT** offset=0x0E4

.. list-table::
   :widths: 10 10 10 10 50
   :header-rows: 1

   * - Field
     - Bits
     - Type
     - Access
     - Description
   * - Reserved
     - 31:18
     - 
     - 
     - 
   * - FLT
     - 23:16
     - Config
     - R/W
     - FLT
   * - LUT
     - 15:0
     - Config
     - R/W
     - LUT

..

**REG_TIM3_CH3_LUT** offset=0x0E8

.. list-table::
   :widths: 10 10 10 10 50
   :header-rows: 1

   * - Field
     - Bits
     - Type
     - Access
     - Description
   * - Reserved
     - 31:18
     - 
     - 
     - 
   * - FLT
     - 23:16
     - Config
     - R/W
     - FLT
   * - LUT
     - 15:0
     - Config
     - R/W
     - LUT

..

**REG_TIM3_COUNTER** offset=0x0EC

.. list-table::
   :widths: 10 10 10 10 50
   :header-rows: 1

   * - Field
     - Bits
     - Type
     - Access
     - Description
   * - T3_COUNTER
     - 31:0
     - Status
     - R
     - ADV_TIMER0 counter register

..

**REG_EVENT_CFG** offset=0x100

.. list-table::
   :widths: 10 10 10 10 50
   :header-rows: 1

   * - Field
     - Bits
     - Type
     - Access
     - Description
   * - Reserved
     - 31:20
     - 
     - 
     - 
   * - OUT_SEL_EVT_ENABLE
     - 19:16
     - Config
     - R/W
     - Output event select ENABLE
   * - OUT_SEL_EVT3
     - 15:12
     - Config
     - R/W
     - Output event select 3
   * - OUT_SEL_EVT2
     - 11:8
     - Config
     - R/W
     - Output event select 2
   * - OUT_SEL_EVT1
     - 7:4
     - Config
     - R/W
     - Output event select 1
   * - OUT_SEL_EVT0
     - 3:0
     - Config
     - R/W
     - Output event select 0

..

**REG_TIM3_COUNTER** offset=0x0EC

.. list-table::
   :widths: 10 10 10 10 50
   :header-rows: 1

   * - Field
     - Bits
     - Type
     - Access
     - Description
   * - Reserved
     - 31:4
     - 
     - 
     - 
   * - CLK_ENABLE
     - 3:0
     - Status
     - R/W
     - Each bit acts as clock enable for each timer. For eg: if 2nd bit is set Timer 2 clock is enabled. 

..  

