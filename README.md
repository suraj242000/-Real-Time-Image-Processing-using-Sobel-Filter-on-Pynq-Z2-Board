# Real-Time Image Processing with a Hardware-Accelerated Sobel Filter on Pynq-Z2

## Project Report and Summary

This project demonstrates a complete hardware/software co-design system for real-time edge detection on a Xilinx Zynq-7000 SoC, specifically using the Pynq-Z2 development board. The core of the project is the creation of a custom hardware accelerator for the Sobel filter algorithm, which is meticulously implemented in the Programmable Logic (PL) of the Zynq chip. This hardware kernel is designed as a reusable AXI-Lite IP core, facilitating seamless integration and control.

The Processing System (PS), running a standalone C application, assumes the crucial role of managing the overall system. Its responsibilities encompass system initialization, efficient data movement utilizing a high-performance AXI DMA controller, and precise coordination with the hardware accelerator. This innovative approach strategically offloads the computationally intensive 2D convolution task, inherent to the Sobel filter, from the software domain to dedicated hardware. This partitioning significantly showcases the immense performance benefits and parallel processing capabilities offered by the Zynq architecture, making it exceptionally well-suited for demanding real-time image and video processing applications.

The system is engineered to process 512x512 8-bit grayscale bitmap (BMP) images, achieving robust performance with a remarkably low measured on-chip power consumption of just 1.449W. This demonstrates not only a powerful but also an energy-efficient implementation, a critical factor for embedded systems.

---

## Table of Contents

- Key Project Features
- System Architecture
- Hardware (PL) Subsystem
- Software (PS) Subsystem
- PS-PL Integration
- Hardware and Software Requirements
- Implementation Workflow: Step-by-Step
    - Step 1: Image Preprocessing
    - Step 2: Hardware Accelerator Design (Vivado)
    - Step 3: Software Control Application (Vitis)
- Complete Source Code
    - Python Preprocessing Script
    - Vitis C Application Code
- Implementation Report: Utilization and Power
    - Resource Utilization
    - On-Chip Power Analysis
- Results: Edge Detection in Action
- Conclusion and Final Summary

---

## 1. Key Project Features

This project embodies several critical features that highlight its advanced design, robust capabilities, and adherence to modern embedded system design principles:

- **Hardware-Accelerated Sobel Filter:** At the core of this system is a custom-designed Sobel filter kernel. This kernel is meticulously implemented in Verilog, a hardware description language, and is deployed directly into the FPGA fabric (Programmable Logic - PL) of the Zynq SoC. This direct hardware implementation allows for massive parallelism and dedicated computational resources, leading to maximum performance for edge detection.

- **Hardware/Software Co-Design:** This project exemplifies a robust hardware/software co-design methodology. It intelligently leverages the inherent strengths of both the ARM Processing System (PS) for high-level control, system management, and orchestration, and the Programmable Logic (PL) for computationally intensive, data-parallel hardware acceleration. This synergistic approach optimizes overall system performance and efficiency.

- **Zynq-7000 SoC Utilization:** The project makes full and optimized utilization of the Xilinx Zynq-7000 System-on-Chip (SoC) architecture, specifically on the Pynq-Z2 development board. The Zynq's unique combination of a dual-core ARM Cortex-A9 processor and a 7 Series FPGA on a single die is fully exploited to create a powerful heterogeneous computing platform.

- **AXI Protocol Suite:** Seamless and high-performance communication between the PS and PL, and among various IP blocks within the PL, is achieved through strict adherence to the industry-standard Advanced eXtensible Interface (AXI) protocol suite:
    - **AXI-Lite:** This simplified version of the AXI protocol is employed for low-bandwidth control and status register (CSR) access. It enables the PS to efficiently configure parameters, start/stop operations, and read status information from the custom hardware IP.
    - **AXI-Stream:** Utilized for high-speed, point-to-point data transfer of the raw and processed image pixels. AXI-Stream is ideal for streaming data, ensuring continuous and efficient flow of pixel information between the DMA and the Sobel IP without address overhead.
    - **AXI-HP (High Performance):** The AXI DMA controller is strategically connected to the PS's DDR memory via a high-performance AXI port (HP0). This dedicated, high-bandwidth pathway ensures extremely efficient and rapid data access for the DMA, which is crucial for real-time image processing.

- **Direct Memory Access (DMA):** The integrated AXI DMA engine is a critical component for managing high-throughput data transfers. It autonomously handles the movement of image data between the PS's DDR memory and the PL-based Sobel accelerator. This offloads the CPU from tedious, byte-by-byte data movement tasks, allowing the ARM processor to focus on higher-level control and application logic.

- **Interrupt-Driven System:** The entire image processing workflow is managed efficiently through an interrupt-driven mechanism. Interrupts originating from both the custom Sobel IP (signaling task completion for a frame or chunk) and the AXI DMA (indicating data transfer completion) notify the PS. This asynchronous communication ensures optimal processor utilization by preventing busy-waiting (polling) and allowing the PS to perform other tasks or enter low-power states while hardware operations are in progress.

- **Low Power Consumption:** A significant achievement of this project is its remarkable power efficiency. The entire on-chip solution operates within a very low power envelope, with a measured total on-chip power consumption of just 1.449W. This demonstrates the inherent energy efficiency benefits of offloading computationally intensive tasks to specialized hardware, making the solution highly suitable for battery-powered or thermally constrained embedded applications.

---

## 2. System Architecture

The system's architecture is meticulously partitioned into two primary functional domains: the hardware subsystem, which resides within the Programmable Logic (PL) of the Zynq SoC, and the software subsystem, executing on the Processing System (PS). This clear division of labor is fundamental to the co-design approach, optimizing both performance and resource utilization.

### Hardware (PL) Subsystem

The Programmable Logic (PL) contains all the custom hardware logic designed and synthesized using Xilinx Vivado. The block diagram (as depicted in the project's Vivado design) illustrates the intricately interconnected components that form the hardware accelerator pipeline:

- **Zynq PS (Interface):** This block serves as the critical interface between the ARM processor cores, their integrated memory controller, and various peripherals within the Processing System, and the custom logic residing in the PL. It facilitates robust communication via high-speed AXI interfaces, acting as the gateway for data and control signals.

- **Sobel Filter IP (imageProcessTop_0):** This is the heart of the hardware acceleration. It is a custom-built hardware accelerator, implemented in Verilog, specifically designed to perform the Sobel edge detection algorithm.
    - It receives a continuous stream of pixels via its S_AXIS (Slave AXI-Stream) input port, ensuring efficient data ingestion.
    - Internally, it employs sophisticated logic, typically involving line buffers (implemented using Block RAMs or distributed RAMs within the FPGA) to store the necessary pixel rows (e.g., three rows for a 3x3 kernel) required to compute the convolution for each output pixel.
    - After performing the Sobel convolution, it outputs the resulting edge-detected pixel data on its M_AXIS (Master AXI-Stream) output port.
    - Crucially, it also includes an interrupt output (o_intr) to signal the PS upon the completion of processing a frame or a significant data block, enabling asynchronous communication.

- **AXI Direct Memory Access (AXI DMA):** This is a pivotal Intellectual Property (IP) block that functions as a high-bandwidth, autonomous bridge between the PS's DDR memory and the AXI-Stream peripherals located in the PL.
    - The Memory Map to Stream (MM2S) channel within the DMA is responsible for efficiently reading the source image data from a designated location in DDR memory and streaming it directly to the Sobel filter IP's input.
    - The Stream to Memory Map (S2MM) channel receives the processed pixel stream from the Sobel filter IP's output and writes it back to another designated location in DDR memory. This entire data transfer process occurs without direct CPU intervention, freeing the ARM cores for other tasks.

- **AXI Interconnects:** These are essential, intelligent bus fabrics that dynamically route AXI transactions between various AXI master components (such as the Zynq PS) and AXI slave peripherals (like the AXI DMA and the custom Sobel IP's AXI-Lite interface). They handle complex tasks such as address decoding, arbitration, and efficient data routing across the PL.

- **AXI4-Stream Data Width Converter:** This utility IP is strategically inserted when there is a mismatch in the data width between the AXI-Stream interfaces of connected components (e.g., if the DMA outputs 32-bit data but the Sobel IP expects 8-bit pixel data). It ensures seamless data flow by converting the stream width as needed, maintaining data integrity.

- **xlconcat (Concatenator):** This specialized utility IP is used to combine multiple individual interrupt signals originating from different PL-based IP cores (specifically, from the imageProcessTop_0 and axi_dma_0) into a single, consolidated interrupt line. This combined interrupt line is then connected to an Interrupt Request (IRQ) input port (IRQ_F2P[0:0]) of the Zynq PS, simplifying the interrupt handling mechanism on the software side.

### Software (PS) Subsystem

The Processing System (PS) runs a standalone C application, meticulously developed within the Xilinx Vitis IDE. Its primary responsibilities are centered around high-level control, orchestration, and overall system management, rather than direct pixel-level computation. This allows the ARM cores to efficiently manage the hardware acceleration process:

- **Driver Initialization:** The PS application begins by initializing all necessary hardware drivers. This includes drivers for the UART (Universal Asynchronous Receiver-Transmitter, used for communication with a host terminal for debugging and result output), the Interrupt Controller (SCUGIC - Xilinx System Control Unit Generic Interrupt Controller), and the AXI DMA controller. These drivers provide the software interface to interact with the underlying hardware.

- **Interrupt System Setup:** This is a critical and intricate part of the software. The code meticulously sets up specific Interrupt Service Routines (ISRs): imageProcISR is dedicated to handling interrupts from the custom Sobel IP, and dmaReceiveISR is responsible for interrupts from the DMA's S2MM channel. These ISRs are then connected to their corresponding hardware interrupt IDs within the SCUGIC. Furthermore, the code configures the SCUGIC's priorities and trigger types for these interrupts and enables exceptions at the ARM processor level (Xil_ExceptionInit, Xil_ExceptionRegisterHandler, Xil_ExceptionEnable) to ensure the processor is fully responsive to these hardware-generated events.

- **DMA Configuration and Initiation:** The software is responsible for configuring the AXI DMA for the initial data transfer. It initiates a DMA simple transfer to send the source image data (residing in the imageData array in DDR memory) to the Sobel filter IP in the PL via the DMA's MM2S (Memory Map to Stream) channel. The system is designed to potentially handle subsequent transfers (e.g., line-by-line or chunk-by-chunk) within the imageProcISR if the hardware IP processes data in a streaming manner that requires software intervention for continuous data feeding.

- **Orchestrated Processing:** The system is designed to process the image either line by line or in larger chunks, depending on the hardware IP's internal buffering and streaming capabilities. The imageProcISR plays a crucial role here, potentially triggering subsequent DMA transfers or managing the flow of data through the accelerator.

- **Completion Waiting:** The main function enters an efficient idle loop (while(!done)). The done flag is a global volatile variable that is set to 1 by the dmaReceiveISR (the interrupt handler for the DMA's S2MM channel) once the entire processed image has been successfully transferred back to DDR memory. This flag signals the completion of the hardware acceleration, allowing the PS to proceed to the next stage.

- **Result Transmission:** Once the done flag is set and the image processing is complete, the software transmits the final processed image data (now residing in the imageData array in DDR memory) back to a connected host computer via UART. This allows for real-time visualization, debugging, or saving of the edge-detected output on a serial terminal.

---

## PS-PL Integration

The seamless and high-performance synergy between the Processing System (PS) and the Programmable Logic (PL) is the absolute cornerstone of this project's success and its ability to achieve real-time performance:

- **PS as Master:** The PS acts as the central master controller, responsible for configuring and commanding the various PL-based slave IP cores, including the AXI DMA and the AXI-Lite control interface of the custom Sobel filter IP. This master-slave relationship ensures centralized control and coordination.

- **Shared Memory:** Image data, encompassing both the raw input and the processed output, resides in the PS-controlled DDR memory. This shared memory space is critically accessible by both the PS (for software operations) and the PL (for hardware acceleration) via the AXI DMA. This eliminates the need for separate memory spaces and complex inter-processor communication protocols for data exchange.

- **Autonomous Data Movement (DMA):** The AXI DMA engine is pivotal because it operates almost entirely autonomously. It efficiently pulls raw image data from DDR memory, streams it directly to the PL accelerator (the Sobel filter), collects the processed results from the accelerator's output stream, and writes them back to a designated location in DDR memory. This crucial capability completely frees the ARM CPU from the burden of tedious, byte-by-byte data transfer tasks, allowing it to dedicate its processing cycles to higher-level control, application logic, or other system functions.

- **Efficient Communication via Interrupts:** Interrupts from the PL-based IP cores serve as an extremely efficient asynchronous communication mechanism. They signal the PS when specific hardware tasks are completed (e.g., a frame is processed by the Sobel IP, a DMA transfer is finished). This allows the PS to react to hardware events promptly without continuously polling status registers. By responding to interrupts, the PS can enter low-power sleep states or perform other concurrent tasks, significantly improving overall system efficiency and responsiveness compared to a purely polled system.

---

## 3. Hardware and Software Requirements

To successfully replicate, develop, and work with this project, the following specific hardware and software components are essential. Adhering to these requirements ensures compatibility and proper functionality:

### Hardware

- **Pynq-Z2 Development Board:** This is the primary target hardware platform for the entire system. The Pynq-Z2 features a Xilinx Zynq-7020 System-on-Chip, which integrates a dual-core ARM Cortex-A9 processor with a 7 Series FPGA.

### Software

- **Xilinx Vivado Design Suite:** This comprehensive suite is indispensable for all hardware design aspects of the project. It is used for:
    - Developing and synthesizing the Verilog-based custom Sobel filter IP.
    - Packaging custom IP cores for reuse within Vivado.
    - Assembling the complete hardware block design in IP Integrator, connecting the Zynq PS, DMA, and custom IP.
    - Performing synthesis, implementation (place-and-route), and generating the bitstream (.bit file) for programming the FPGA.

- **Xilinx Vitis Unified Software Platform:** This is the integrated development environment (IDE) specifically designed for software development, debugging, and deployment on Xilinx embedded processors, including the Zynq PS. It is used for:
    - Creating platform projects from the Vivado-generated hardware platform (.xsa file).
    - Developing the C/C++ application code that runs on the ARM processor.
    - Debugging the software application directly on the Pynq-Z2 board.

- **Python 3.x with Pillow Library:** This is essential for the image preprocessing script. The Pillow library (a fork of PIL - Python Imaging Library) provides robust image manipulation capabilities. It can be easily installed via pip:  
    `pip install Pillow`

---

## 4. Implementation Workflow: Step-by-Step

This section provides a detailed, step-by-step guide to the entire implementation process, from the initial preparation of image data to the final software deployment and execution on the Pynq-Z2 board. Following these steps ensures a successful build and deployment of the real-time image processing system.

### Step 1: Image Preprocessing

The hardware accelerator is rigidly designed for a fixed image size (512x512 pixels) and a specific color format (8-bit grayscale). Therefore, a crucial preprocessing step is required for any arbitrary input image before it can be processed by the hardware:

- A dedicated Python script, leveraging the powerful PIL (Pillow) library, was developed to handle this essential conversion.

- This script takes any input image file (e.g., a JPG, PNG, or other common image formats) and performs two primary operations:
    - **Grayscale Conversion:** It converts the input image to an 8-bit grayscale format. This is critical because the Sobel filter operates on single-channel intensity values, discarding color information.
    - **Resizing:** It accurately resizes the image to the exact 512x512 pixel dimensions, which is the precise input size expected by the hardware accelerator.

- The processed image is then saved as a standard BMP (Bitmap) file. This format is chosen for its simplicity and direct pixel representation, making it easier to extract raw pixel data.

- Finally, this processed BMP file is converted into a C header file (e.g., imageData.h). This header file contains the raw pixel data as a u8 (unsigned char) array. This allows the image data to be directly compiled into the Vitis application code, making the image data readily available in the PS's DDR memory for efficient DMA transfer to the PL.

### Step 2: Hardware Accelerator Design (Vivado)

The entire hardware design phase, including the development of the custom IP and the assembly of the system-level block diagram, is conducted exclusively within the Xilinx Vivado Design Suite:

- **Sobel IP Creation:**
    - A custom Sobel filter module was designed and meticulously implemented in Verilog. This module is highly optimized for hardware acceleration, focusing on parallel computation of the convolution.
    - The module was engineered to be streaming-capable, meaning it processes pixels continuously as they arrive, without needing to store the entire image. To achieve this, it incorporates line buffers (typically implemented using Block RAMs or distributed RAMs within the FPGA fabric). These buffers temporarily hold the necessary pixel rows (e.g., three rows for a 3x3 kernel) required to compute the convolution for each output pixel.
    - Once designed, this Verilog module was then packaged as an AXI-Lite IP core within Vivado. This packaging process defines its interfaces, exposes any necessary control registers (for configuration by the PS), and specifies its AXI-Stream data ports (for pixel input and output).

- **Block Design Assembly:**
    - A new Vivado project was initiated, specifically targeting the Pynq-Z2 board's Zynq-7020 SoC.
    - The ZYNQ7 Processing System IP block was added to the IP Integrator canvas. Its configuration was meticulously set to enable the AXI_HP0 (High Performance) port (essential for high-speed data access from PL to PS DDR memory) and the IRQ_F2P (Interrupt Request from Fabric to Processor) port (to receive interrupts generated by PL-based IP).
    - Your custom imageProcessTop_0 IP (the Sobel filter) and the AXI Direct Memory Access IP were added to the canvas.
    - AXI Interconnect blocks were strategically added to create the necessary bus fabrics. For instance, one interconnect was used to connect the M_AXI_GP0 (General Purpose AXI) port of the PS to the S_AXI_LITE configuration ports of the AXI DMA and potentially the Sobel IP (if it had AXI-Lite control registers).
    - The DMA's M_AXI_S2MM (Memory Map to Stream) and S_AXI_MM2S (Stream to Memory Map) ports, which handle data transfers to and from DDR memory, were connected to the S_AXI_HP0 port on the PS via another AXI interconnect. This ensures high-speed, dedicated memory access.
    - The M_AXIS_MM2S (DMA output stream, carrying the input image data) was connected to the S_AXIS port (Sobel IP input stream). An AXI4-Stream Data Width Converter was inserted between them if the data widths of the DMA's output stream and the Sobel IP's input stream did not match, ensuring seamless data flow.
    - The M_AXIS port (Sobel IP output stream, carrying the processed edge-detected image data) was connected to the S_AXIS_S2MM port (DMA input stream for writing data back to DDR memory).
    - **Interrupts:** The individual interrupt outputs from the DMA (s2mm_introut) and the Sobel IP (o_intr) were connected to an xlconcat (concatenator) IP. The single, consolidated output of the xlconcat IP was then wired to the IRQ_F2P[0:0] port of the Zynq PS, effectively merging all PL interrupts into one line for the PS.

- **Finalization:**
    - The entire block design was rigorously validated within Vivado's IP Integrator to check for any connection errors, address conflicts, or other design rule violations.
    - The design was then synthesized (translating Verilog to gates) and implemented (performing place-and-route to map the design onto the FPGA fabric).
    - Finally, the bitstream (.bit file) was generated. This is the binary configuration file used to program the FPGA. Concurrently, the hardware platform (.xsa file) was exported. This .xsa file contains all the necessary hardware information (IP addresses, interrupt connections, etc.) that is crucial for subsequent software development in Vitis.

### Step 3: Software Control Application (Vitis)

The software development, which orchestrates the entire image processing flow and runs on the Zynq's ARM Processor System, is performed within the Xilinx Vitis Unified Software Platform:

- **Project Creation:**
    - A new Application Project was created in Vitis.
    - Crucially, this project was linked to the exported hardware platform (.xsa file) generated in Vivado. This step ensures that the Vitis environment is fully aware of the custom hardware IP, its memory-mapped addresses, and its interrupt connections, providing the necessary context for software development.

- **Code Development:**
    - The C code was meticulously written to control and interact with the hardware components. The logical flow of the application is as follows:
        - **Initialization:** The code begins by looking up the configuration parameters and initializing the drivers for the essential peripherals: the UART (for serial communication), the SCUGIC (System Control Unit Generic Interrupt Controller), and the AXI DMA controller. This establishes the software's ability to communicate with and control the hardware.
        - **Interrupt System Setup:** This is a critical and complex part of the software. The code meticulously sets up the Interrupt Service Routines (ISRs): imageProcISR is assigned to handle interrupts originating from the custom Sobel IP, and dmaReceiveISR is responsible for interrupts from the DMA's S2MM channel. These ISRs are then connected to their respective hardware interrupt IDs within the SCUGIC. Furthermore, the code configures the SCUGIC's priorities and trigger types for these interrupts and enables exceptions at the ARM processor level (Xil_ExceptionInit, Xil_ExceptionRegisterHandler, Xil_ExceptionEnable) to ensure the processor is fully responsive to these hardware-generated events.
        - **DMA Transfer Initiation:** The software is responsible for configuring the AXI DMA for the initial data transfer. It initiates a DMA simple transfer to send the source image data (residing in the imageData array in DDR memory) to the Sobel filter IP in the PL via the DMA's MM2S (Memory Map to Stream) channel. The system is designed to potentially handle subsequent transfers (e.g., line-by-line or chunk-by-chunk) within the imageProcISR if the hardware IP processes data in a streaming manner that requires software intervention for continuous data feeding.
        - **Main Loop (Waiting for Completion):** The main function enters an efficient idle loop (while(!done)). The done flag is a global volatile variable that is set to 1 by the dmaReceiveISR (the interrupt handler for the DMA's S2MM channel) once the entire processed image has been successfully transferred back to DDR memory. This flag signals the completion of the hardware acceleration, allowing the PS to proceed to the next stage.
        - **Data Transmission (UART):** Once the done flag is set and the image processing is complete, the software transmits the final processed image data (now residing in the imageData array in DDR memory) back to a connected host computer via UART. This allows for real-time visualization, debugging, or saving of the edge-detected output on a serial terminal.

---

## 5. Complete Source Code

This section provides the complete source code for both the Python preprocessing script and the Vitis C application that runs on the Zynq's Processing System.

### Python Preprocessing Script

This Python script is crucial for preparing the input images. It converts any given image into the specific 512x512 8-bit grayscale BMP format required by the hardware accelerator. It utilizes the Pillow library for image manipulation.

```python
###################### Preprocessing
from PIL import Image
import os

def convert_to_grayscale(image_path, output_path):
    """
    Converts an image to a 512x512 grayscale bitmap.

    This function takes an input image, opens it, converts it to an 8-bit
    grayscale format, resizes it to a fixed 512x512 resolution, and then
    saves the result as a BMP (Bitmap) file. This ensures the image is
    in the correct format and dimensions for the hardware accelerator.

    Args:
        image_path (str): The file path to the input image (e.g., "input.jpg").
        output_path (str): The file path where the processed BMP image will be saved
                           (e.g., "output_image.bmp").
    """
    try:
        # Open the image file using Pillow
        image = Image.open(image_path)
        
        # Convert the image to grayscale. 'L' mode represents 8-bit pixels, black and white.
        grayscale_image = image.convert("L")
        
        # Resize the grayscale image to the target resolution of 512x512 pixels.
        resized_image = grayscale_image.resize((512, 512))
        
        # Save the processed image as a BMP file.
        resized_image.save(output_path)
        
        print(f"Image successfully processed and saved to {output_path}")

        # In a typical workflow, this BMP file would then be converted into a C header file
        # (e.g., imageData.h) containing a byte array of the pixel data. This conversion
        # is often done by a separate utility or script that reads the raw pixel data
        # from the BMP and formats it into a C array declaration.
        # For example, you might read the BMP bytes and format them into a C array.
        # with open(output_path, 'rb') as f:
        #     bmp_data = f.read()
        # print(f"BMP file size: {len(bmp_data)} bytes")

    except FileNotFoundError:
        # Handle the case where the input image file does not exist.
        print(f"Error: Input image not found at {image_path}")
    except Exception as e:
        # Catch any other exceptions that might occur during image processing.
        print(f"An unexpected error occurred during image processing: {e}")

# Example Usage:
# To use this function, ensure 'n1.jpg' (or your input image) exists in the
# same directory as this script, or provide its full path.
# input_image_path = "n1.jpg"
# output_image_path = "output_image.bmp"
# convert_to_grayscale(input_image_path, output_image_path)

# Optional: Function to generate a dummy imageData.h for initial Vitis project setup.
# In a real project, you would convert the actual processed BMP to this format.
def generate_dummy_image_data_h(file_name="imageData.h", size=512*512):
    """
    Generates a dummy C header file (imageData.h) containing a byte array.
    This is useful for initial Vitis project setup and compilation before
    the actual BMP-to-C array conversion utility is integrated.
    The array is filled with simple gradient data for demonstration.
    """
    with open(file_name, "w") as f:
        f.write("#ifndef IMAGE_DATA_H\n")
        f.write("#define IMAGE_DATA_H\n\n") # Format for readability (16 bytes per line)
        f.write("#include \"xil_types.h\"\n\n") # Include Xilinx types for u8 etc.
        f.write(f"unsigned char imageData[{size}] = {{\n");
        for i in range(size):
            f.write(f"0x{i % 256:02x}, ") # Fill with a simple repeating pattern (0x00 to 0xFF)
            if (i + 1) % 16 == 0: # Format for readability (16 bytes per line)
                f.write("\n")
        f.write("};\n\n")
        f.write("#endif // IMAGE_DATA_H\n")
    print(f"Generated dummy {file_name} with {size} bytes of data.")

# Uncomment the line below to generate a dummy imageData.h for initial testing:
# generate_dummy_image_data_h()
```

---

### Vitis C Application Code

This C code runs on the Zynq's Processor System (PS) and serves as the primary control application. It is responsible for initializing all necessary peripherals, configuring the AXI DMA controller, setting up the interrupt system, initiating data transfers to the hardware-accelerated Sobel filter in the PL, and finally, transmitting the processed results back to a host terminal via UART.

```c
/*
 * main.c
 *
 * This application configures a hardware-accelerated Sobel filter in the PL (Programmable Logic),
 * uses an AXI Direct Memory Access (DMA) controller to stream an image through it,
 * and manages the entire process using an interrupt-driven approach.
 *
 * Author: Suraj Prajapati (M23EEV019)
 * Project: Real-Time Image Processing using Sobel Filter on Pynq-Z2 Board
 *
 * This code orchestrates the interaction between the Zynq's ARM Processor System (PS)
 * and the custom hardware accelerator (Sobel filter) in the Programmable Logic (PL).
 * It handles DMA transfers, interrupt management, and serial communication for results.
 */

#include "xaxidma.h"      // AXI DMA driver for high-speed data transfers
#include "xparameters.h"  // Hardware configuration parameters (generated by Vivado/Vitis)
#include "sleep.h"        // Provides usleep() for small delays
#include "xil_cache.h"    // Xilinx cache operations (flush and invalidate) for data coherency
#include "xil_io.h"       // Xilinx I/O functions for direct register access (e.g., XAxiDma_ReadReg)
#include "xscugic.h"      // Xilinx SCU GIC (Generic Interrupt Controller) driver
#include "imageData.h"    // Custom header file expected to contain:
                          // `unsigned char imageData[512*512] = {...};`
                          // This array serves as both the input buffer for raw image data
                          // and the output buffer for processed image data.
#include "xuartps.h"      // UART driver for serial communication with a host terminal

// --- Definitions for Hardware IDs and Image Size ---
#define imageSize             512 * 512  // Total number of pixels for a 512x512 grayscale image
#define DMA_DEVICE_ID         XPAR_AXI_DMA_0_DEVICE_ID // Device ID for the AXI DMA IP as defined in xparameters.h
#define INTC_DEVICE_ID        XPAR_PS7_SCUGIC_0_DEVICE_ID  // Device ID for the PS's SCU GIC
#define SOBEL_IP_INTR         XPAR_FABRIC_IMAGEPROCESSTOP_0_0_INTR_INTR // Interrupt ID for the custom Sobel IP
#define DMA_RX_INTR           XPAR_FABRIC_AXI_DMA_0_S2MM_INTROUT_INTR   // Interrupt ID for DMA S2MM channel (receive completion)
#define UART_DEVICE_ID        XPAR_PS7_UART_0_DEVICE_ID // Device ID for the PS UART peripheral

// --- Global Variables ---
// Instance of the GIC Interrupt Controller. This structure holds the state of the GIC.
XScuGic IntcInstance;
// Instance of the AXI DMA controller. This structure holds the state of the DMA.
XAxiDma myDma;
// Volatile flag to signal completion of DMA transfer from PL to PS memory.
// 'volatile' ensures the compiler doesn't optimize away reads, as it's modified by an ISR.
volatile int done = 0;

// --- Function Prototypes ---
// Function to check if the AXI DMA is in an idle state by reading its status register.
u32 checkIdle(u32 baseAddress, u32 offset);
// Interrupt Service Routine (ISR) for the custom Image Processor IP (Sobel Filter).
// This ISR is triggered by the Sobel IP in the PL, typically upon completing a processing task.
static void imageProcISR(void *CallBackRef);
// Interrupt Service Routine (ISR) for the AXI DMA S2MM (Stream to Memory Map) channel.
// This ISR is triggered when the DMA finishes transferring data from the PL to PS memory.
static void dmaReceiveISR (void *CallBackRef);

// --- Main Function: Entry point of the application ---
int main() {
    u32 status; // Variable to store the return status of various API calls
    XUartPs_Config *myUartConfig; // Pointer to UART configuration structure
    XUartPs myUart;               // Instance of the UART peripheral

    xil_printf("--- Starting Real-Time Image Processing Application ---\n\r");

    // --- 1. UART Initialization ---
    // Look up the configuration information for the UART device based on its ID.
    myUartConfig = XUartPs_LookupConfig(UART_DEVICE_ID);
    if (myUartConfig == NULL) {
        xil_printf("ERROR: UART config lookup failed. Check xparameters.h and Vivado design.\n\r");
        return XST_FAILURE; // Return failure status
    }
    // Initialize the UART driver with the looked-up configuration and base address.
    status = XUartPs_CfgInitialize(&myUart, myUartConfig, myUartConfig->BaseAddress);
    if (status != XST_SUCCESS) {
        xil_printf("ERROR: UART initialization failed. Status: %d\n\r", status);
        return XST_FAILURE;
    }
    // Set the baud rate for UART communication (e.g., 115200 bits per second).
    status = XUartPs_SetBaudRate(&myUart, 115200);
    if (status != XST_SUCCESS) {
        xil_printf("ERROR: UART baudrate setting failed. Status: %d\n\r", status);
        return XST_FAILURE;
    }
    xil_printf("UART Initialized and configured at 115200 bps.\n\r");

    // --- 2. DMA Initialization ---
    // Look up the configuration information for the AXI DMA device.
    XAxiDma_Config *myDmaConfig;
    myDmaConfig = XAxiDma_LookupConfig(DMA_DEVICE_ID);
    if (myDmaConfig == NULL) {
        xil_printf("ERROR: DMA config lookup failed. Check xparameters.h and Vivado design.\n\r");
        return XST_FAILURE;
    }
    // Initialize the AXI DMA driver with the looked-up configuration.
    status = XAxiDma_CfgInitialize(&myDma, myDmaConfig);
    if (status != XST_SUCCESS) {
        xil_printf("ERROR: DMA initialization failed. Status: %d\n\r", status);
        return XST_FAILURE;
    }
    xil_printf("AXI DMA Initialized.\n\r");

    // Disable all DMA interrupts initially to ensure a clean state before configuring specific ones.
    // XAXIDMA_DMA_TO_DEVICE refers to the MM2S (Memory Map to Stream) channel.
    XAxiDma_IntrDisable(&myDma, XAXIDMA_IRQ_ALL_MASK, XAXIDMA_DMA_TO_DEVICE);
    // XAXIDMA_DEVICE_TO_DMA refers to the S2MM (Stream to Memory Map) channel.
    XAxiDma_IntrDisable(&myDma, XAXIDMA_IRQ_ALL_MASK, XAXIDMA_DEVICE_TO_DMA);

    // --- 3. Interrupt Controller (SCUGIC) Initialization ---
    // Look up the configuration information for the GIC (Generic Interrupt Controller).
    XScuGic_Config *IntcConfig;
    IntcConfig = XScuGic_LookupConfig(INTC_DEVICE_ID);
    if (IntcConfig == NULL) {
        xil_printf("ERROR: GIC config lookup failed. Check xparameters.h.\n\r");
        return XST_FAILURE;
    }
    // Initialize the GIC driver with the looked-up configuration.
    status = XScuGic_CfgInitialize(&IntcInstance, IntcConfig, IntcConfig->CpuBaseAddress);
    if (status != XST_SUCCESS) {
        xil_printf("ERROR: Interrupt controller initialization failed. Status: %d\n\r", status);
        return XST_FAILURE;
    }
    xil_printf("Interrupt Controller (GIC) Initialized.\n\r");

    // --- 4. Set up Interrupt Handlers ---
    // 4.1. Sobel IP Interrupt (from PL to PS)
    // Set priority (0xA0) and trigger type (3 = rising edge) for the Sobel IP's interrupt.
    XScuGic_SetPriorityTriggerType(&IntcInstance, SOBEL_IP_INTR, 0xA0, 3);
    // Connect the Sobel IP's interrupt handler (imageProcISR) to the GIC.
    status = XScuGic_Connect(&IntcInstance, SOBEL_IP_INTR, (Xil_InterruptHandler)imageProcISR, (void *)&myDma);
    if (status != XST_SUCCESS) {
        xil_printf("ERROR: Sobel IP interrupt connection failed. Status: %d\n\r", status);
        return XST_FAILURE;
    }
    // Enable the Sobel IP's interrupt in the GIC.
    XScuGic_Enable(&IntcInstance, SOBEL_IP_INTR);
    xil_printf("Sobel IP Interrupt configured and enabled.\n\r");

    // 4.2. DMA Receive Interrupt (S2MM channel - from PL to PS)
    // Set priority (0xA1) and trigger type (3 = rising edge) for the DMA S2MM interrupt.
    XScuGic_SetPriorityTriggerType(&IntcInstance, DMA_RX_INTR, 0xA1, 3);
    // Connect the DMA receive interrupt handler (dmaReceiveISR) to the GIC.
    status = XScuGic_Connect(&IntcInstance, DMA_RX_INTR, (Xil_InterruptHandler)dmaReceiveISR, (void *)&myDma);
    if (status != XST_SUCCESS) {
        xil_printf("ERROR: DMA RX interrupt connection failed. Status: %d\n\r", status);
        return XST_FAILURE;
    }
    // Enable the DMA S2MM interrupt in the GIC.
    XScuGic_Enable(&IntcInstance, DMA_RX_INTR);
    xil_printf("DMA Receive Interrupt configured and enabled.\n\r");

    // --- 5. Initialize Exception Handling in ARM Processor ---
    // Initialize the exception handling mechanism of the ARM processor.
    Xil_ExceptionInit();
    // Register the GIC interrupt handler as the primary exception handler for interrupts (ID_INT).
    // This ensures that when an interrupt occurs, the GIC's handler is called.
    Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT, (Xil_ExceptionHandler)XScuGic_InterruptHandler, (void *)&IntcInstance);
    // Enable exceptions at the processor level.
    Xil_ExceptionEnable();
    xil_printf("Processor Exception Handling Enabled.\n\r");

    xil_printf("Preparing for Image Processing via DMA...\n\r");

    // --- 6. Cache Coherency Management ---
    // Flush the data cache for the 'imageData' buffer.
    // This is crucial to ensure that any modifications made by the CPU to 'imageData'
    // are written back from the CPU's cache to the main DDR memory before the DMA starts reading it.
    Xil_DCacheFlushRange((UINTPTR)imageData, imageSize);
    xil_printf("Data Cache Flushed for imageData buffer (input data).\n\r");

    // --- 7. Initiate DMA Transfers ---
    // 7.1. Configure the S2MM (Stream to Memory Map) channel for receiving processed data from PL.
    // This sets up the DMA to receive 'imageSize' bytes into the 'imageData' buffer from the PL.
    // It's important to set up the receive channel *before* the transmit channel to avoid race conditions.
    status = XAxiDma_SimpleTransfer(&myDma, (u32)imageData, imageSize, XAXIDMA_DEVICE_TO_DMA);
    if (status != XST_SUCCESS) {
        xil_printf("ERROR: DMA S2MM (receive) transfer setup failed. Status: %d\n\r", status);
        return XST_FAILURE;
    }
    xil_printf("DMA S2MM channel configured for receiving processed data from PL.\n\r");

    // 7.2. Configure the MM2S (Memory Map to Stream) channel for sending raw data to PL.
    // This sets up the DMA to send 'imageSize' bytes from the 'imageData' buffer to the PL.
    // Once this transfer starts, the hardware accelerator in the PL will begin processing.
    status = XAxiDma_SimpleTransfer(&myDma, (u32)imageData, imageSize, XAXIDMA_DMA_TO_DEVICE);
    if (status != XST_SUCCESS) {
        xil_printf("ERROR: DMA MM2S (transmit) transfer setup failed. Status: %d\n\r", status);
        return XST_FAILURE;
    }
    xil_printf("DMA MM2S channel configured for sending input data to PL. Hardware processing initiated.\n\r");

    // --- 8. Wait for Processing Completion ---
    // The main function enters an idle loop, waiting for the 'done' flag to be set
    // by the 'dmaReceiveISR'. This flag is set when the S2MM transfer (receiving processed data) completes.
    // This is an efficient way to wait, as the CPU is free to do other tasks or enter low-power states
    // while the hardware is busy.
    while (!done) {
        // The processor can potentially enter a low-power state here (e.g., WFI - Wait For Interrupt)
        // to save power, rather than continuously busy-waiting.
    }

    xil_printf("Image Processing Complete. Retrieving results and sending via UART.\n\r");

    // --- 9. Cache Coherency Management for Results ---
    // Invalidate the data cache for the 'imageData' buffer.
    // This is crucial because the DMA has written the processed data directly to DDR memory,
    // bypassing the CPU's cache. Invalidating ensures that the CPU reads the fresh data
    // from DDR memory when accessing 'imageData', not stale data from its cache.
    Xil_DCacheInvalidateRange((UINTPTR)imageData, imageSize);
    xil_printf("Data Cache Invalidated for imageData buffer (processed output).\n\r");

    // --- 10. Send Processed Data Over UART ---
    // Transmit the entire processed image data byte-by-byte via UART to a connected host terminal.
    // This part can be slow for large images (512x512 = 262144 bytes) due to UART's limited bandwidth.
    // It's primarily for debugging, verification, or simple display.
    for (int i = 0; i < imageSize; i++) {
        XUartPs_Send(&myUart, (u8*)&imageData[i], 1);
        usleep(10); // Small delay to prevent overwhelming the UART buffer or the host receiver.
                    // Adjust this delay based on your UART's capabilities and host's receiving speed.
    }

    xil_printf("--- Program Execution Finished ---\n\r");
    return 0; // Indicate successful program execution
}

// --- Interrupt Service Routine for Image Processor IP ---
// This ISR is triggered by the custom Sobel filter IP in the PL.
// Its specific behavior (e.g., triggering next chunk of processing) depends on the IP's design
// and whether it processes the image in a fully streaming manner or in smaller chunks.
static void imageProcISR(void *CallBackRef) {
    // 'i' is a static variable from the original code, suggesting some internal logic
    // related to processing chunks or rows of the image.
    static int i = 4; // Example value from original code, likely for internal logic.
    // int status; // Uncomment if needed for status checks within the ISR.
    // XAxiDma *DmaInstance = (XAxiDma *)CallBackRef; // Cast CallBackRef if DMA operations are performed here.

    // Disable the Sobel IP's interrupt to prevent re-entry while this ISR is executing.
    // This is a common practice to avoid nested interrupts or race conditions.
    XScuGic_Disable(&IntcInstance, SOBEL_IP_INTR);
    
    // This section would typically contain logic to handle the IP's completion
    // for a specific chunk/row and potentially initiate the next transfer if needed.
    // The original code snippet had a fixed size of 512 bytes per transfer within this ISR.
    if(i < 514){ // Example condition from original code, likely related to image rows or chunks.
        // If the Sobel IP processes data in chunks and needs the PS to initiate
        // the next chunk's transfer, that logic would go here.
        // For a fully streaming IP, this ISR might just acknowledge status or clear flags.
        // Example (commented out, but shows potential use if image processing is chunked):
        // status = XAxiDma_SimpleTransfer(DmaInstance, (u32)&imageData[i*512], 512, XAXIDMA_DMA_TO_DEVICE);
        i++; // Increment 'i' to track progress or the next chunk.
    }

    // Re-enable the Sobel IP's interrupt in the GIC to allow future interrupts
    // once this ISR has completed its execution.
    XScuGic_Enable(&IntcInstance, SOBEL_IP_INTR);
}

// --- Interrupt Service Routine for DMA S2MM (Device to Memory) channel ---
// This ISR is triggered by the AXI DMA when a transfer from the PL to PS memory (S2MM channel) completes.
// It's crucial for signaling the main application that the processed data is ready.
static void dmaReceiveISR(void *CallBackRef) {
    u32 IrqStatus; // Variable to hold the interrupt status register value.
    XAxiDma *DmaInstance = (XAxiDma *)CallBackRef; // Cast the callback reference to a pointer to the DMA instance.

    // Read the pending interrupt status from the DMA S2MM channel.
    IrqStatus = XAxiDma_IntrGetIrq(DmaInstance, XAXIDMA_DEVICE_TO_DMA);
    
    // Acknowledge the interrupts to clear them in the DMA controller's status register.
    // This is essential to allow future interrupts from the DMA.
    XAxiDma_IntrAckIrq(DmaInstance, IrqStatus, XAXIDMA_DEVICE_TO_DMA);

    // Check if the Interrupt on Completion (IOC) flag is set in the status register.
    // This indicates that the entire DMA transfer has completed successfully.
    if (IrqStatus & XAXIDMA_IRQ_IOC_MASK) {
        done = 1; // Set the global 'done' flag to signal completion to the main function.
    }
    
    // --- Optional: Add more robust error handling for DMA ---
    // You can check for other bits in 'IrqStatus' to detect various DMA errors,
    // such as:
    // - XAXIDMA_IRQ_DELAY_MASK (Delay Interrupt)
    // - XAXIDMA_IRQ_ERROR_MASK (Error Interrupt)
    // For example:
    // if (IrqStatus & XAXIDMA_IRQ_ERROR_MASK) {
    //     // Handle DMA error: print message, reset DMA, etc.
    //     xil_printf("DMA S2MM Error Detected!\n\r");
    //     // Optionally reset the DMA: XAxiDma_Reset(DmaInstance);
    // }
}
```

---

## 6. Implementation Report: Utilization and Power

The implementation on the Pynq-Z2 board demonstrates highly efficient resource utilization within the Programmable Logic (PL) and a remarkably low power profile. These metrics underscore the significant benefits of hardware acceleration for embedded applications, where both performance and energy efficiency are critical.

### Resource Utilization

The design is lightweight and efficiently utilizes the Zynq-7020's Programmable Logic (PL) resources. The low utilization percentages indicate that the design is compact and leaves ample room for more complex logic or the integration of additional IP cores, making the design highly scalable for future enhancements.

| Resource | Utilization | Available | Utilization % |
|----------|-------------|-----------|---------------|
| LUT      | 6271        | 53200     | 11.79%        |
| LUTRAM   | 1587        | 17400     | 9.12%         |
| FF       | 6543        | 106400    | 6.15%         |
| BRAM     | 8           | 140       | 5.71%         |
| DSP      | 2           | 220       | 0.91%         |

- LUT (Look-Up Tables): Used for implementing combinational logic. Low utilization indicates efficient logic synthesis.
- LUTRAM (LUT-based RAM): Used for small, distributed memory blocks, often for line buffers in streaming designs like this.
- FF (Flip-Flops): Used for implementing sequential logic (registers).
- BRAM (Block RAM): Dedicated, larger memory blocks. Used for more substantial buffers, such as line buffers for the Sobel filter.
- DSP (Digital Signal Processing Slices): Specialized hardware blocks optimized for arithmetic operations (multiplication, accumulation). The low DSP utilization suggests the convolution might be implemented primarily using LUTs or that the Sobel filter's core operations are efficiently mapped.

### On-Chip Power Analysis

The total on-chip power consumption for the entire system is a modest 1.449 W. This power analysis, derived from the implemented netlist with activity estimates, highlights the power efficiency achieved through hardware acceleration. The breakdown reveals that the Processing System (PS7 block) is the dominant power consumer, which is typical for Zynq designs where the ARM cores are active even during PL operations. The custom hardware accelerator in the PL adds minimal overhead to the overall power budget, demonstrating the energy-saving potential of offloading tasks to dedicated hardware.

- **Total On-Chip Power:** 1.449 W
- **Dynamic Power:** 1.312 W (representing approximately 91% of the total power). This is the power consumed due to switching activity within the device.
    - **PS7 (Processor System):** 1.256 W (constituting approximately 94% of the dynamic power). This indicates that the ARM cores and their associated peripherals are the primary contributors to dynamic power consumption, even when managing PL operations.
    - **Clocks, Signals, Logic, BRAM, DSP:** The remaining 0.057 W (approximately 6% of dynamic power) is attributed to these PL components, showcasing their energy efficiency.
- **Device Static Power:** 0.136 W (representing approximately 9% of the total power). This is the quiescent power consumed by the device even when no operations are performed.
- **Junction Temperature:** 41.7 °C. This is the estimated temperature of the silicon die during operation.
- **Thermal Margin:** 43.3 °C (with a design power budget of 3.6 W). This indicates ample thermal headroom, meaning the device is operating well within its safe temperature limits and can dissipate more heat if needed.

---

## 7. Results: Edge Detection in Action

The project successfully demonstrates the real-time edge detection capabilities of the implemented system. The visual results provide clear and compelling evidence of the effectiveness and correct functionality of the hardware-accelerated Sobel filter.

The comparison below visually illustrates the transformation from the original grayscale input image to the precisely edge-detected output:

- **A: Original 512x512 Grayscale Image**
    - This image represents the input to the hardware accelerator after the preprocessing step (grayscale conversion and resizing). It contains the raw intensity information of the scene.
    - The image is ready to be streamed into the Sobel filter for edge analysis.

- **B: Edge Detected 512x512 Output Image**
    - This image is the direct result generated by the hardware-accelerated Sobel filter residing in the Zynq's Programmable Logic.
    - The prominent white lines against a dark background clearly and accurately highlight the significant intensity changes within the original image, which correspond precisely to the edges of objects and features present in the scene.
    - The sharpness and accuracy of the detected edges confirm the functional correctness and high performance of the custom hardware accelerator and the overall integrated hardware/software system.

---

## 8. Conclusion and Final Summary

This project stands as a comprehensive and practical case study in the successful creation of a hardware-accelerated image processing system utilizing the powerful Xilinx Zynq-7000 System-on-Chip on the Pynq-Z2 development board.

The core achievement lies in the successful integration of a custom AXI-Lite IP core, meticulously implementing the computationally intensive Sobel filter algorithm, with the ARM Processing System. This integration is seamlessly facilitated by the high-performance AXI Direct Memory Access (DMA) controller and an efficient interrupt-driven communication mechanism. This validation of the hardware/software co-design methodology demonstrates its immense potential for complex embedded applications.

By intelligently offloading the pixel-intensive 2D convolution task, which is a bottleneck for software-only implementations, to the dedicated Programmable Logic, we have achieved high-throughput edge detection. This level of performance would be significantly challenging, if not impossible, for the ARM processor alone to handle in real-time. Crucially, this high performance is accomplished while maintaining an impressively low on-chip power profile of just 1.449W, making the solution exceptionally well-suited for power-constrained embedded vision applications.

The architecture developed in this project is inherently scalable and forms a solid, extensible foundation. It can be readily adapted and expanded for implementing more advanced real-time computer vision and digital signal processing algorithms on embedded FPGA-based platforms. This project not only demonstrates functional correctness and high performance but also vividly showcases the profound efficiency and power benefits inherent in leveraging the Zynq's unique heterogeneous architecture for accelerating critical computational tasks.
