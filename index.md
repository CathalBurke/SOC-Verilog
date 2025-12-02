---
layout: home
title: FPGA VGA Driver Project
tags: fpga vga verilog
categories: demo
---

## Introduction


For our System-on-Chip assignment I built a VGA driver on a Digilent Basys3 FPGA board and then edited the design so that the monitor shows a Belgian flag (black, yellow, red) in full screen. This blog walks through:

- how the original template design works,
- how I tested it in simulation,
- how I got it running on the FPGA,
- and how I modified the colour logic to draw the flag.



---

## My Project Setup

**Hardware**

- Digilent **Basys3** FPGA board (Xilinx Artix-7, 100 MHz on-board clock)  
- External **VGA monitor** connected to the Basys3 VGA port  
- VGA resolution: **640×480 @ 60 Hz**

**Vivado project files in this repo**

- VGATop.v – top-level module for the VGA system  
- VGASync.v – generates VGA timing (hsync, vsync, row, col, vid_on)  
- VGAColourCycle.v – template colour-cycle demo  
- Testbench.v – simulation testbench  
- Basys3_Master.xdc – pin constraints (clock, reset, VGA pins)  

The XDC file:

- connects the 100 MHz board clock to my clk signal,
- maps a slide switch to rst,
- and maps hSync, vSync, and the 4-bit vgaRed, vgaGreen, vgaBlue buses to the correct Basys3 VGA pins using 3.3 V LVCMOS.

<img src="https://raw.githubusercontent.com/CathalBurke/SOC-Verilog/main/docs/assets/images/MyprojectSummary.png" alt="Vivado project overview">
*Figure 1 – Vivado project summary showing the main VGA modules and constraints file in my design.*


---

## Understanding the Template Design

Before drawing any flags I needed to understand the **template VGA driver** [1].

### VGA timing in plain English
From the Manual [2] I learned that the Basys3 monitor  wants a stream of pixels at a fixed rate. For 640×480 @ 60 Hz:

- A **25 MHz pixel clock** drives the timing.  
- Every tick, the design moves one pixel to the right (col increments).  
- At the end of a line, it jumps to the next line (row increments).  
- Extra “off-screen” pixels and lines are used for sync pulses and blanking.  

VGASync.v implements this using two counters:

- hcount – horizontal pixel position within a line  
- vcount – vertical line number within a frame  

From these, it generates:

- hsync and vsync – active-low sync pulses  
- vid_on – high only when we are actually inside the 640×480 visible area  
- row and col – exposed versions of vcount and hcount for the rest of the design

So any image generator only has to look at (row, col) and decide what colour that pixel should be.

### Modules in the template

At the top level, VGATop.v wires everything together:

- A **clock wizard** divides the 100 MHz system clock down to **25 MHz** for the VGA logic.
- VGASync produces hsync, vsync, vid_on, row and col.
- A colour module (ColourCycle) produces 4-bit red, green, blue signals.

To avoid drawing outside the visible area, the colours are gated with vid_on:

    assign vgaRed   = (vid_on) ? red   : 4'b0;
    assign vgaGreen = (vid_on) ? green : 4'b0;
    assign vgaBlue  = (vid_on) ? blue  : 4'b0;

The original ColourCycle module is a finite state machine that slowly cycles the whole screen through different colours (black, red, yellow, green, cyan, blue, magenta, white). It is driven by a wide counter so that each colour holds long enough to be clearly visible on the monitor.

<img src="https://raw.githubusercontent.com/CathalBurke/SOC-Verilog/main/docs/assets/images/elaboratedschematic.png" alt="Elaborated schematic in Vivado">
*Figure 2 – Elaborated schematic of VGATop in Vivado, including the clock wizard, VGASync and colour-generation logic.*


---

## Simulation

I started in simulation using Testbench.v. Simulating a full 640×480 frame is overkill, so the testbench **scales the timing parameters right down**:

- HDISP = 6, VDISP = 2  
- Very small front porch, sync pulse and total limits  

This means the horizontal and vertical counters wrap quickly and I can view several “frames” in a short simulation time.

The testbench:

- generates a clock with period T,  
- asserts rst high for a couple of cycles, then releases it,  
- instantiates VGASync with the reduced parameters,  
- and instantiates the colour module.

In the waveform viewer I checked that:

- hcount and vcount cycled through the expected ranges,  
- hsync and vsync pulses appeared in the right positions,  
- vid_on was only high in the display region,  
- the colour outputs changed as the FSM moved through states.

Once the timing looked correct in simulation, I was confident enough to move to hardware.

---

## Synthesis & Implementation

Next step: turn Verilog into gates and download it to the FPGA.

In Vivado I ran the usual flow:

1. **Synthesis** – RTL → gate-level netlist, basic checks for issues like unconnected signals.  
2. **Implementation** – place and route the design on the Artix-7 fabric, honouring the Basys3 pin constraints.  
3. **Generate Bitstream** – produce the .bit file for the board.

This design is tiny compared to the resources on the Basys3, so:

- LUT and flip-flop usage is low,
- timing for both 100 MHz and 25 MHz clocks is easily met,
- there are no routing or timing violations.

---

## Demonstration: Template Design on Hardware

With the bitstream generated, I:

1. Connected the Basys3 VGA port to a monitor.  
2. Powered the board and selected the correct VGA input on the monitor.  
3. Programmed the FPGA from Vivado.

The monitor locked onto the 640×480 @ 60 Hz signal and showed the colour-cycling pattern from ColourCycle. This proved that:

- my timing generator worked,  
- the constraints file matched the real pins,  
- and the basic VGA pipeline was correct.

With that working, it was time to personalise the design.

---

## My VGA Design Edit: Drawing the Belgian Flag

For the edit part of the project, I replaced the colour-cycling demo with a static **Belgian flag**. The real flag is three vertical stripes:

1. Black  
2. Yellow  
3. Red  

The nice thing about VGA is that I already know the **column index** from my own  reasearch and the moodle slides for every pixel [3], from VGASync (col). So the idea is simple:

- Split the visible width into three equal sections using col.  
- Choose black, yellow or red depending on which section we are in.  

All of the timing and sync logic stays untouched. I only swap out the colour module.

### The ColourStripes module

I created a new module called ColourStripes with this interface:

    module ColourStripes (
        input  wire        clk,
        input  wire        rst,
        input  wire [10:0] col,
        input  wire [10:0] row,
        output reg  [3:0]  red,
        output reg  [3:0]  green,
        output reg  [3:0]  blue
    );

Inside, I use col to split the screen into three vertical regions. For a 640-pixel-wide display, each stripe is roughly 213 pixels wide:

    localparam LEFT_END   = 11'd213;  // end of black stripe
    localparam MIDDLE_END = 11'd426;  // end of yellow stripe

    always @* begin
        // Default to black
        red   = 4'h0;
        green = 4'h0;
        blue  = 4'h0;

        if (col < LEFT_END) begin
            // Left stripe: black (0,0,0)
            red   = 4'h0;
            green = 4'h0;
            blue  = 4'h0;
        end else if (col < MIDDLE_END) begin
            // Middle stripe: yellow (red + green)
            red   = 4'hF;
            green = 4'hF;
            blue  = 4'h0;
        end else begin
            // Right stripe: red
            red   = 4'hF;
            green = 4'h0;
            blue  = 4'h0;
        end
    end

VGATop.v then instantiates this module instead of ColourCycle:

    ColourStripes u_colour_stripes (
        .clk(clk),
        .rst(rst),
        .col(col),
        .row(row),
        .red(red),
        .green(green),
        .blue(blue)
    );

Because VGATop already gates the outputs with vid_on, the flag is only visible in the active display area; the blanking regions remain black.

### Simulation of the flag

I reused the same style of simulation as before, focusing on a single frame:

- I watched col sweep across a visible line.  
- I checked that the RGB outputs switched:
from black to yellow at col = LEFT_END,  
and from yellow to red at col = MIDDLE_END.  

This confirmed that the stripe boundaries matched what I expected before programming the board.Once the timing looked correct in simulation, I was confident enough to move to hardware.

---

## Demonstration: Belgian Flag on the Basys3

Finally, I generated a new bitstream with ColourStripes and programmed the Basys3 again. I demonstrated this live in the lab on the Basys3 board, with the flag clearly visible on an external VGA monitor.


On the monitor I now see a **stable Belgian flag**:

- a black stripe on the left,  
- a yellow stripe in the middle,  
- a red stripe on the right,  

with the monitor happily locked to the VGA timing. At this point the project requirements are met: I understand the template design, have modified it, and demonstrated my own image on the FPGA.

<img src="https://raw.githubusercontent.com/CathalBurke/SOC-Verilog/main/docs/assets/images/Belgium Flag.jpg" alt="Belgian flag output from my Basys3 board">
*Figure 3 – Belgian flag generated by my ColourStripes module and displayed on a VGA monitor using the Basys3 board.*


---

## What I Learned

From a 3rd year student point of view, this project pulled together a lot of theory and practice:

- **Digital design in context** – counters, FSMs and parameterised modules doing something visible.  
- **Timing actually matters** – not just setup/hold theory, but “does the monitor lock or not?”.  
- **Simulation first** – catching logic mistakes in the waveform viewer is much faster than guessing on the board.  
- **Reading and using constraints** – the design only works if the pins and I/O standards are correct.  
- **Linking software and hardware thinking** – instead of “drawing” pixels in a frame buffer, I’m streaming them in real time from simple logic.

If I had more time, next steps would be:

- adding a switch to toggle between different flags or patterns,  
- adding simple animation (e.g. fading between colours),  
- or generating text/graphics by decoding (row, col) more creatively.

---

## References

These are the main resources I used alongside the lab sheet:

1.  M. Lynch, .**FPGA VGA lab template**   ATU Galway, 2025
2. **Basys 3 Reference Manual** - Digilent Reference, digilent.com. https://digilent.com/reference/programmable-logic/basys-3/reference-manual?redirect=1
‌
3. M. Lynch,  **Module lecture slides**  ATU Galway, 2025.
---

