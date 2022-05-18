# Tutorial-exercise: Embedded Memories 

## Contents
* [Embedded memories](#embedded-memories)
  * [The ready\-enable microprotocol](#the-ready-enable-microprotocol)
  * [Creating a RAM block using the Wizard] (#creating-a-ram-block-using-the-wizard)
* [Designing the Decryption Circuit](#designing-the-decryption-circuit)
  * [General implementation requirements](#general-implementation-requirements)
  * [Task 1: ARC4 state initialization](#task-1-arc4-state-initialization)


### Embedded memories

In this task, you will get started by creating a RAM using the Megafunction Wizard, creating circuitry to fill the memory, and observing the contents using the In-System Memory Content Editor.

### The ready-enable microprotocol

A ready/enable microprotocol is a handshacking mechanism that can be used to achieve the goal of two digital blocks ("caller" and "callee") synchronizing with each other when exchanging data.

The handshake has two sides: the “caller” (think: employer) and the “callee” (think: employee). Whenever the callee is ready to accept a request, it asserts its `rdy` signal. If `rdy` is asserted, the caller may assert `en` to make a “request” to the callee. The following timing diagram illustrates this:

<p align="center"><img src="figures/rdy-en.svg" title="ready-enable microprotocol" width="65%" height="65%"></p>

It is illegal for the caller to assert `en` if `rdy` is deasserted; if this happens, the behaviour of the callee is undefined.

Whenever `rdy` is asserted, it means that the callee is able to accept a request _in the same cycle_. This implies that a module that needs multiple cycles to process a request and cannot buffer more incoming requests **must** ensure `rdy` is deasserted in the cycle following the `en` call. Similarly, each cycle during which the `en` signal is asserted indicates a distinct request, so the caller must ensure `en` is deasserted in the following cycle if it only wishes to make a single request. The following timing diagram shows an example of this behaviour:

<p align="center"><img src="figures/rdy-en-singleclock.svg" title="ready-enable microprotocol" width="65%" height="65%"></p>

This microprotocol allows the callee to accept multiple requests and buffer them. You **do** need to make sure you deassert `rdy` unless you can immediately accept another request.

Finally, some requests come with arguments. In this case, the argument port must be valid **at the same time** as the corresponding `en` signal, as in this diagram:

<p align="center"><img src="figures/rdy-en-arg.svg" title="ready-enable microprotocol with an argument" width="65%" height="65%"></p>

Note: Be careful about combinational loops. For example, since `en` can derive from `rdy` through combinational logic, `rdy` cannot also derive from `en` combinationally; otherwise, the two signals will form a wire loop.


#### Creating a RAM block using the Wizard

First, create a new Quartus project. Then, create a memory component as follows:

In Quartus, select _Tools&rarr;IP Catalog_, and from the IP catalog pane that opens, choose _Basic Functions&rarr;On Chip Memory&rarr;RAM: 1-Port_.

Choose Verilog, and create an output file called `s_mem.v` in your project directory. In the next few panels, customize your Megafunction as follows:

- How wide should the _q_ output bus be? **8 bits**
- How many 8-bit words of memory? **256 words**
- What should the memory block type be? **M10K** (this is the SRAM block embedded in the Cyclone V)
- What clocking method would you like to use? **single clock**
- Which ports should be registered: **make sure the _q_ output port is unselected**
- Create one clock enable signal… : **do not select**
- Create an _aclr_ asynchronous clear… : **do not select**
- Create a _rden_ read enable… : **do not select**
- What should the _q_ output be…: **new data**
- Do you want to specify the initial contents? **no**
- Allow In-System Memory Content Editor to capture and update…: **select this**
- The _Instance ID_ of this RAM is: **S** (uppercase)
- Generate netlist: **do not select**
- Do you want to add the Quartus Prime file to the project? **yes**

When you finish this, you will find the file `s_mem.qip` in your project file list. If you expand it, you will also see `s_mem.v`. Open the Verilog file and examine it: you will find the module declaration for `s_mem`, which will look something like this:

```Verilog
module s_mem (
        address,
        clock,
        data,
        wren,
        q);

        input [7:0] address;
        input clock;
        input [7:0] data;
        input wren;
        output [7:0] q;
```

Be sure you create the memories as described, and that your declaration matches the above. This is the module you will include as a component in your design. **Do not modify this file**, or it might not do what you want during synthesis and simulation.

In the rest of the file you can see how `s_mem.v` configures and instantiates the actual embedded RAM component.

The instance ID you specified (here, “S”) will be used to identify this memory when you examine the memory while your circuit is running inside your FPGA.

In the tasks below, you will have to create additional memories, one called `pt_mem` and another called `ct_mem`, with corresponding `.v` file names. Be sure they were all generated using the same settings. 


#### Simulating Altera memories in ModelSim

To simulate with ModelSim, you will need to include the `altera_mf_ver` library (under the Libraries tab) when you start simulation. If you are using the tcl shell instead of clicking around, use the `-L` option to `vsim`, like in this example:

    vsim -L altera_mf_ver work.tb_task1

For netlist simulation, you will also need `cyclonev_ver`, `altera_ver`, and `altera_lnsim_ver`.

To make ModelSim happy about how many picoseconds each #-tick is, you will have to add

```
`timescale 1ps / 1ps
```

at the beginning of your RTL files and testbench files.

#### Examining memory contents when simulating RTL in Modelsim

You might find ModelSim's memory viewer (accessible from _View&rarr;Memory List_) helpful here; it will list all the memories in the design and allow you to examine any of them. It might be useful to change the radix to hex (right-click on the memory contents view and select _Properties_).

In your RTL testbench, you can access the memory from your testbench using the dot notation:

    dut.s.altsyncram_component.m_default.altsyncram_inst.mem_data

(assuming you named your `task1` instance `dut` inside your testbench). Note that **the dot notation may be used only in your testbench**, not in anything you wish to synthesize.

If you decide to initialize the memory in one of your testbenches in an initial block, be sure to do this **after a delay** (e.g., `#10`); otherwise your initialization will end up in a race condition with the Altera memory model and its own initial block.


#### Examining memory contents when simulating a netlist in Modelsim

In a post-synthesis netlist, your design will have been flattened into a sea of primitive FPGA components. So what happens with the memories and the lovely hierarchical path that allowed us to access the contents?

The good news is that the memories survive somewhere inside your netlist, and the primitive memory blocks are modelled as Verilog memory arrays like the RTL models. This means that we can examine them from the _Memory List_ tab and use Verilog array notation or `$readmemh` and friends to fill them (see below).

The name also survives, albeit in a horribly mangled form. Once you complete Task 1, look at the post-synthesis netlist file `task1.vo` from Task 1 and look for `cyclonev_ram_block`. You should see one instance:

```
cyclonev_ram_block \s|altsyncram_component|auto_generated|altsyncram1|ram_block3a0 (
    .portawe(!count[8]),
    .portare(vcc),
    .portaaddrstall(gnd),
    .portbwe(\s|altsyncram_component|auto_generated|mgl_prim2|enable_write~0_combout ),
    .portbre(vcc),
    ...
```

Note the space before the opening bracket: it's actually **part of the identifier syntax**, not just a meaningless space. The \ and the space delineate an escaped identifier in SystemVerilog, and you have to include the space in the middle of the hierarchical name if you want to access the array inside:

```
dut.\s|altsyncram_component|auto_generated|altsyncram1|ram_block3a0 .ram_core0.ram_core0.mem
```

The space is still there — looks weird, but that's how things work in SystemVerilog.


#### Initializing memory contents in simulation

In your testbench, you will likely want to use `$readmemh()` to initialize memories and compare them to a known reference state. You can look up how `$readmemh()` works in the _SystemVerilog 2017 Language Standard_ posted with the course documents. If you read any external files in your testbench, you will have to commit them **in the same folder** as the testbench that uses them.


#### Examining memory contents in the FPGA in Quartus

You can examine the contents of the memory while your circuit is running in the FPGA.

To do this, program your FPGA with the circuit and select _Tools&rarr;In-System Memory Content Editor_ while your circuit is active. This will open a window that shows the memory contents. In the instance manager window (left) you should see a list of memories in your design (in this Task, only the _S_ memory you created).  Right click _S_, and choose _Read Data from In-System Memory_:

<p align="center"><img src="figures/mem-editor.png" title="memory editor" width="75%" height="75%"></p>

The In-System Memory Content Editor (ISMCE) in only available for the single-ported memory configurations. The reason is simple: when you generate a memory with the in-system editing option enabled, Quartus generates circuitry to read and write your memory; that circuitry takes up one of the ports of the underlying embedded memory, leaving you with only one port.

**NOTE** when you generate memories which enable the ISMCE, or include other debugging options like SignalTap, then Quartus will add 4 extra top-level signals to your module when you generate a post-synthesis netlist. These signals are called `altera_reserved_tms`, `altera_reserved_tck`, `altera_reserved_tdi`, and `altera_reserved_tdo`; this last signal is an output, while the other three are all inputs. If you create a testbench that uses the `(.*)` notation to make connections with your DUT instance, then it will be looking for those signal names in your post-synthesis netlist testbench as well. The correct way to fix this problem is to not use the `(.*)` notation.


#### Initializing memory contents in the FPGA

In Quartus, you can initialize memories at compilation time using either a Memory Initialization File (.mif) or an Intel Hex Format file (.hex). We recommend you use the first format because it's _a lot_ easier to read, but it's up to you. Either way, Quartus includes a table-like editor for these formats; you can create a new file via _File&rarr;New&rarr;Memory Files_.


### Memory state initialization

In the `task1` folder you will find a `init.sv` and a toplevel file `task1.sv`. In `init.sv`, you will initialize the content of the memory to [0..255], which you could express in pseudo-code as:

    for i = 0 to 255:
        s[i] = i

The `init` module follows the ready/enable microprotocol [described above](#the-ready-enable-microprotocol).
You will see that this declares the component that you have just created using the Wizard.

First, generate the `s_mem` memory exactly as described above.

Next, examine the toplevel `task1` module. You will find that it already instantiates the `s_mem` RAM you generated earlier using the MF Wizard. `KEY[3]` will serve as our reset signal in `task1`. Add an instance of your `init` module and connect it to the RAM instance. Make sure that `init` is activated **exactly once** every time after reset, and that _S_ is not written to after `init` finishes. 


Remember to follow the ready-enable microprotocol we defined earlier. It is not outside the realm of possibility that we could replace either `init` or `task1` with another implementation when testing your code.

Also, be sure that you follow the instance names in the template files. Check that, starting from `task1`, s-mem memory is accessible in simulation via either

    s.altsyncram_component.m_default.altsyncram_inst.mem_data

in RTL simulation, and

    \s|altsyncram_component|auto_generated|altsyncram1|ram_block3a0 .ram_core0.ram_core0.mem

in netlist simulation.

Proceed to import your pin assignments and synthesize as usual. Examine the memory contents in RTL simulation, post-synthesis netlist simulation, and on the physical FPGA.


