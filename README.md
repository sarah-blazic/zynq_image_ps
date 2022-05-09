# Creating a Zynq Image Processing Platform 

#### Introduction
The purpose of this project is to use the Zybo Z710 as an Image Processing Platform to change the resolution of the image on the display. To perform this task, I used a Zybo Z710, a Nikon D3400 camera, a mini HDMI to HDMI cable, an HDMI cable, and a monitor with an HDMI input to display the processed image. For this project the Nikon D3400 camera can be replaced by any camera that has a mini HDMI or regular HDMI port. 

#### Procedure and Results
To begin, the Diligent Vivado Library must be imported to the project in Vivado. Next, I began the block design by adding a ZYNQ7 Processing System. Afterwards, I ran an automatic configuration to create the DDR and FIXED_IO external ports. The ZYNQ processing system then had to be configured to enable the HP Slave AXI interface, HP0, on the PS-PL Configuration tab. This enables high performance on the slave interface 0. On the Peripheral I/O Pins, the I2C 0 was added to the default settings. This pin will be used to create an external pin, hdmi_out_ddc, which is the output’s Data Display Channel (DDC). Within the Clock Configuration tab, the PL Fabric clocks 0 through 2 must be enabled with the following settings displayed in the picture below.

![FCLK](https://i.imgur.com/Zl2MLFF.png)

Multiple fabric clocks must be configured due to the differing clocking requirements of the DVI to RGB Video Decoder, the AXI Video DMA with its stream converting blocks, and the AXI GPIO. The last configuration on the ZYNQ processing system is to enable the IRQ_F2P[15:0] PL-PS Interrupt port under the Interrupts tab. 

The following IP blocks were added after the processor was set: two AXI4-Stream Subset Converters (one for input and one for output), two Video Timing Controllers (one for input and one for output), a Dynamic Clock Generator, an AXI4-Stream to Video Out, an AXI GPIO, a Constant (used to control the reset of the AXI4-Stream Subset Converters), a Video In to AXI4-Stream, and a Concat (to control interrupt port of the ZYNQ processor with numerous interrupt sources) with corresponding AXI Interconnects and Processor System Resets. A RGB to DVI Video Encoder (Source) was added by dragging and dropping the HDMI out configuration from the Board and the DVI to RGB Video Decoder (Sink) was similarly added with the HDMI in configuration. 
The AXI GPIO was enabled to export the hdmi_in_hpd, therefore it was configured to be a custom board interface with the IP configurations shown in the following photo.  This output is used to detect if there is a connected HDMI Source. 

![GPIO](https://i.imgur.com/KfQpbOz.png)

Then, the ip2intc_irpt was connected to a pin on the 5 port Concat IP block with its output going to the S_AXI_HP0 block of the ZYNQ Processing System. 

The Video In to AXI4-Stream’s clock mode was changed to Independent and the FF0 depth was changed to 4096. This is so that the Video In to AXI4-Stream clock will be controlled by the slowest sync clock. The Video Timing Controller for the inputs configuration can be found in the photo below. 

![Video in to Stream](https://i.imgur.com/BfAnyvm.png)

As for the output Video Timing Controller, the only difference from the default settings is that detection is disabled. The irq output of both Video Timing Controller blocks will be inputs to the Concat IP controlling the ZYNQ interrupts.  

For the AXI Video Direct Memory Access IP block, the write channel moves the AXI Stream video into a AXI Memory mapped form for storage in the PS DDR memory. While the read channel accesses the PS DDR and converts the AXI Memory Mapped format into a AXI stream for output. Both directions must be enabled. The last two inputs to the ZYNQ interrupts are the mm2s_introut and ss2mm_introut, respectively of the AXI Video Direct Memory Access IP block. The complete configuration for this block is pictured below.  

![VDMA](https://i.imgur.com/JtH062D.png)

The RGB to DVI Video Encoder was configured by disabling the reset active high and disabling the Generate SerialClk internally from pixel clock option. This is so that the SerialClk and the aRst_n can be controlled by the Dynamic Clock Generator. 

The AXI4-Stream to Video out'ss configuration can be viewed in the photo below. Essentially, the FIFO Depth was changed to 4096, the clock mode was switched to independent, and the timing mode changed to Master. 

![Stream to Video out](https://i.imgur.com/qU6kQF0.png)

Both AXI4-Stream Subset Converters are configured to have independent clocks that the pixel clock and AXI stream clocks are different. The full configuration of both in and out subset converters are pictured below.

![Stream Subset Converters](https://i.imgur.com/0DzmMqM.png)

The rest of the connections, including the different AXI Interconnects and Processor System Resets can be viewed in the full block design photographed below. 

![BD](https://i.imgur.com/UrizG4X.png)

After the block design has been completed and validated, an HDL wrapper was created for the block design source and a synthesis was run. When this was completed, the ports corresponding to the receiving and transmitting HDMI functions were mapped to the Zybo Z710. With these ports mapped to the ports represented in the following tables, a bitstream was generated and the hardware was exported via XSA file with the aforementioned bitstream.

**HDMI TX** 
| Pin / Signal | Description | FPGA Pin |
|-----|-----|-----|
| D[2]_P, D[2]_N | Data O/P | B19, A20 |
| D[1]_P, D[1]_N | Data O/P | C20, B20 | 
| D[1]_P, D[1]_N | Data O/p | D19, D20 |
| CLK_P, CLK_N | CLK O/P | H16, H17 |
|HPD/HPA| Hot-plug detect I/P| G17, G18 |
| SCL, SDA | DDC bidirectional | E18 | 


**HDMI RX** 
| Pin / Signal | Description | FPGA Pin |
|-----|-----|-----|
| D[2]_P, D[2]_N | Data I/P | N20, P20 |
| D[1]_P, D[1]_N | Data I/P | T20, U20 | 
| D[1]_P, D[1]_N | Data I/p | V19, W20 |
| CLK_P, CLK_N | CLK I/P | U18, U19 |
|HPD/HPA| Hot-plug detect O/P| W18, Y19 |
| SCL, SDA | DDC bidirectional | W19 | 

The software was implemented via the Vitis IDE. An application project was created with the following attributes: based off of the hardware XSA file, standalone OS platform, ps7_cortexa9_0, and an empty C application. A variety of files were then imported to help initialize the interrupts, video timers, and dynamic clock. The main.c file focuses on two functions: DemoInitialize() and DemoRun(). The DemoInitialize() function initializes an array of pointers to the 3 frame buffers, a timer for a simple delay, a VDMA driver, the display controller, the interrupt controller, and the video capture device. Meanwhile, the DemoRun() function flushes the UART and takes a user input that either exits the program or changes the screen resolution. Within DemoRun(), there is a CRMenu() which displays the screen resolution options the user may pick and a corresponding case statement that implements the users’ choice based on their input. 

This project was then run by building and debugging it with a Single Application Debug (GDB). Within the Vitis terminal under the debug setting of the Vitis IDE, the user has two options: input a 1 to change the screen resolution or input a q to quit the program. If any other character was input, a “Invalid Input,” message will be displayed and the menu will be repeated to guide the user. If a 1 was chosen, a new menu containing the various screen resolution options will be shown. If the user does not input any of the options, the error message will be displayed once more and the menu will repeat. Once a screen resolution is picked, the monitor will display the image within the confines of the new resolution. A demo of this project may be viewed in the link below.

https://youtu.be/UByttYzg4iA
	
#### Conclusion
This lab demonstrates how the Zybo Z710 can receive input from an HDMI camera, process the image, and export it to a monitor via HDMI cable. A main portion of this project is understanding the connection between the IP blocks within the Vivado block design. It showed a pathway, aided by timing controllers and buffers, that took a video input from the HDMI Rx port, converted it into streaming form in order to transfer the data across the Video Direct Memory Access or (VDMA), and then converted it back into a video output form for the HDMI Tx port. This kind of data processing can be used in almost any video or strict image processing application. 



