# Amazon EC2 F1 Tutorial: The Shell-to-CL AXI-Lite interface used in the Hello World Example

The “Hello World” example exercises the [OCL](#ocl) Shell-to-CL AXI-Lite interface, the Virtual LED outputs and the Virtual DIP switch inputs. This blog post will walk through the custom logic RTL (Verilog), explain the AXI-lite slave logic, and highlight the PCIe APIs that can be used for accessing the registers behind the AXI-lite interface.


## Table of Contents

1. [Overview](#overview)
2. [Function Description](#functional)
3. Hello World Example Metadata

<a name="overview"></a>
## Overview:

This simple *hello_world* example builds a Custom Logic (CL) that will enable the instance to "peek" and "poke" registers in the Custom Logic (CL).
These registers will be in the memory space behind AppPF BAR0, which is the ocl\_cl\_ AXI-lite bus on the Shell to CL interface.

This example demonstrate a basic use-case of the Virtual LED and Virtual DIP switches.

All of the unused interfaces between AWS Shell and the CL are tied to fixed values, and it is recommended that the developer use similar values for every unused interface in the developer's CL.


Now let’s start with the top-level Verilog module of the example, `aws-fpga/hdk/cl/examples/cl_hello_world/design/cl_hello_world.sv`:

```verilog
module cl_hello_world 

(
   `include "cl_ports.vh" // Fixed port definition

);

`include "cl_common_defines.vh"      // CL Defines for all examples
`include "cl_id_defines.vh"          // Defines for ID0 and ID1 (PCI ID's)
`include "cl_hello_world_defines.vh" // CL Defines for cl_hello_world

logic rst_main_n_sync;


//--------------------------------------------0
// Start with Tie-Off of Unused Interfaces
//---------------------------------------------
// the developer should use the next set of `include
// to properly tie-off any unused interface
// The list is put in the top of the module
// to avoid cases where developer may forget to
// remove it from the end of the file

`include "unused_flr_template.inc"
`include "unused_ddr_a_b_d_template.inc"
`include "unused_ddr_c_template.inc"
`include "unused_pcim_template.inc"
`include "unused_dma_pcis_template.inc"
`include "unused_cl_sda_template.inc"
`include "unused_sh_bar1_template.inc"
`include "unused_apppf_irq_template.inc"
```

First you will see the above module declaration uses a `include "cl_ports.vh"` statement to declare all the interface ports that are fixed for the CL. The cl_ports.vh file is located at
aws-fpga/hdk/common/shell_v04261818/design/interfaces/cl_ports.vh.

Users should tie-off all unused interfaces, by including a list of AWS-provided Verilog files located at
aws-fpga/hdk/common/shell_v04261818/design/interfaces/unused_*_template.inc

```verilog
//-------------------------------------------------
// ID Values (cl_hello_world_defines.vh)
//-------------------------------------------------
  assign cl_sh_id0[31:0] = `CL_SH_ID0;
  assign cl_sh_id1[31:0] = `CL_SH_ID1;

//-------------------------------------------------
// Reset Synchronization
//-------------------------------------------------
logic pre_sync_rst_n;

always_ff @(negedge rst_main_n or posedge clk_main_a0)
   if (!rst_main_n)
   begin
      pre_sync_rst_n  <= 0;
      rst_main_n_sync <= 0;
   end
   else
   begin
      pre_sync_rst_n  <= 1;
      rst_main_n_sync <= pre_sync_rst_n;
   end
```

Then you will see the standard PCIe and PCIe subsystem ID settings, and an always block synchronizing the input reset signal, rst_main_n.

```verilog
//-------------------------------------------------
// PCIe OCL AXI-L (SH to CL) Timing Flops
//-------------------------------------------------

  axi_register_slice_light AXIL_OCL_REG_SLC (
   .aclk          (clk_main_a0),
   .aresetn       (rst_main_n_sync),

   // CL's top level interface signals connecting to Shell.
   .s_axi_awaddr  (sh_ocl_awaddr),
   .s_axi_awprot   (2'h0),
   .s_axi_awvalid (sh_ocl_awvalid),
   .s_axi_awready (ocl_sh_awready),
   .s_axi_wdata   (sh_ocl_wdata),
   .s_axi_wstrb   (sh_ocl_wstrb),
   .s_axi_wvalid  (sh_ocl_wvalid),
   .s_axi_wready  (ocl_sh_wready),
   .s_axi_bresp   (ocl_sh_bresp),
   .s_axi_bvalid  (ocl_sh_bvalid),
   .s_axi_bready  (sh_ocl_bready),
   .s_axi_araddr  (sh_ocl_araddr),
   .s_axi_arvalid (sh_ocl_arvalid),
   .s_axi_arready (ocl_sh_arready),
   .s_axi_rdata   (ocl_sh_rdata),
   .s_axi_rresp   (ocl_sh_rresp),
   .s_axi_rvalid  (ocl_sh_rvalid),
   .s_axi_rready  (sh_ocl_rready),

   // Local signals connecting to internal CL implementation.
   .m_axi_awaddr  (sh_ocl_awaddr_q),
   .m_axi_awprot  (),
   .m_axi_awvalid (sh_ocl_awvalid_q),
   .m_axi_awready (ocl_sh_awready_q),
   .m_axi_wdata   (sh_ocl_wdata_q),
   .m_axi_wstrb   (sh_ocl_wstrb_q),
   .m_axi_wvalid  (sh_ocl_wvalid_q),
   .m_axi_wready  (ocl_sh_wready_q),
   .m_axi_bresp   (ocl_sh_bresp_q),
   .m_axi_bvalid  (ocl_sh_bvalid_q),
   .m_axi_bready  (sh_ocl_bready_q),
   .m_axi_araddr  (sh_ocl_araddr_q),
   .m_axi_arvalid (sh_ocl_arvalid_q),
   .m_axi_arready (ocl_sh_arready_q),
   .m_axi_rdata   (ocl_sh_rdata_q),
   .m_axi_rresp   (ocl_sh_rresp_q),
   .m_axi_rvalid  (ocl_sh_rvalid_q),
   .m_axi_rready  (sh_ocl_rready_q)
  );


```


Following that there is the logic for “PCIe OCL AXI-L (SH to CL) Timing Flops”. It uses an “AXI register slice” core (axi_register_slice_light) to connect all top level OCL Shell-to-CL interface ports to a set of corresponding “local” signals through pipeline registers. This is for the timing purpose. The slave access implementation of the PCIe OCL AXI-L interface will interact with this set of “local” signals. Now let’s take a look at the AXI-lite slave logic.

```verilog
logic        wr_active;
logic [31:0] wr_addr;
 
always_ff @(posedge clk_main_a0)
  if (!rst_main_n_sync) begin
     wr_active <= 0;
     wr_addr   <= 0;
  end
  else begin
     wr_active <=  wr_active && bvalid  && bready ? 1'b0     :
                  ~wr_active && awvalid           ? 1'b1     :
                                                    wr_active;
     wr_addr <= a// Write Request
wvalid && ~wr_active ? awaddr : wr_addr     ;
  end
 
assign awready = ~wr_active;
assign wready  =  wr_active && wvalid;
 
// Write Response
always_ff @(posedge clk_main_a0)
  if (!rst_main_n_sync) 
    bvalid <= 0;
  else
    bvalid <=  bvalid && bready ? 1'b0  : 
              ~bvalid && wready ? 1'b1  :
                                  bvalid;
assign bresp = 0;
```


On the write side, the wr_active register represents whether the write operation is active. It toggles from 0 to 1 on the assertion of the input write address valid signal (awvalid) (line 12); and toggles from 1 to 0 if the write response handshaking signals — output valid (bvalid) and input ready (bready), are high (line 11).
The wr_addr register storing the write address updates its value on the assertion of write address valid (awvalid) if there is no currently active write operation (~wr_active) (line 14)
The write address ready output (awready) is high if and only if there is no active write operation (line 17).
The write data ready (wready) output is asserted when write operation is active and write data valid input is asserted (line 18).
The write response output register (bvalid) toggles from 0 to 1 after wready is asserted (i.e., after write data valid input is received) (line 26); and toggles from 1 to 0 after the Shell master asserts bready signal (line 25).


# Appendinx

## AXI-Lite interfaces for register access -- (SDA, OCL, BAR1)

There are three AXI-L master interfaces (Shell is master) that can be used for register access interfaces. Each interface is sourced from a different PCIe PF/BAR. Breaking this into multiple interfaces allows for different software entities to have a control interface into the CL:

* SDA AXI-L: Associated with MgmtPF, BAR4. If the developer is using AWS OpenCL runtime Lib (as in SDAccel case), this interface will be used for performance monitors etc.
* OCL AXI-L: Associated with AppPF, BAR0. If the developer is using AWS OpenCL runtime lib(as in SDAccel case), this interface will be used for openCL Kernel access
* BAR1 AXI-L: Associated with AppPF, BAR1.

Please refer to PCI Address map for a more detailed view of the address map.