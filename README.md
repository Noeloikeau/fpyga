# Introduction
> The `fpyga` Python package simplifies workflow with Intel FPGAs. It can automatically generate verilog, control location assignments, compile, program, read/write to memory, handle multiple FPGAs connected simultaneously, and archive projects all from within Python or Jupyter notebooks. See the docs with examples at https://noeloikeau.github.io/fpyga/.


## Installation
`pip install fpyga`

## Usage

```python
import fpyga

from fpyga import *
```

## Preliminaries

Before using this tutorial, you should:

1. Install Quartus
2. Have FPGAs connected
3. Have a Quartus project open "DE10"
4. Generate a "MyPLL" Phase-Lock-Loop megafunction from the Quartus GUI
5. Similarly generate a "DE10.cdf" file in ..\output_files folder using the Programming GUI
6. Connect PIN_V11 to FPGA_CLK1_50 as input to the PLL using the Assignments GUI
7. Ensure path variables point to the Quartus scripting folder to run command-line arguments

Note: this page was previously titled `quartusfpga`, and included several utilities which have since been separated into the `sidis` package following a full re-write. `quartusfpga` is no longer maintained and has been removed.

## Project Class
We will use the `Project` class to smoothe over the process of writing location assignments, compiling, programming, reading/writing to memory, and archiving in Quartus. All the methods here are derived from functions defined in the other pages, which may be used as stand-alones if the class functionality isn't required.

```python
P=Project(projectname='DE10',
          projdir=r'C:\Users\Noeloikeau Charlot\Desktop\Research\Quartus',
          chip=DE10,
          templatefile=r'C:\Users\Noeloikeau Charlot\Desktop\Research\qdev\Repressilator_Template.txt',
         __N__=3,__T__=8,__D__=1)
```

The `templatefile` will fill in the Verilog project automatically from a base text file where we implement a Ring Oscillator, as explained below. If you're happy with Verilog and just want to see how to interface with the FPGA, feel free to skip to the following sections.

## sidis Templates
We use the `sidis` module for several utilities, including the `Template` class, which lets you write Python formatting functions inside text documents. This means you can easily replace and expand upon the native constructs of hardware programming language like Verilog. As an example, this notebook deals with a ring of three cyclic inverters implemented on two DE10 Nano Cyclone V Intel FPGAs.

```python
s=sidis.Template('Repressilator_Template.txt')
print(s.temp[11:19]) #These are the relevant lines for this example
```

    localparam N=__N__; //number of nodes
    localparam T=__T__; //number of measurements
    localparam D=__D__; //number of inverter pairs between nodes
    wire [N-1:0] node; //ring of cyclic inverters
    wire [N-1:0] init; //initial state of ring
    wire node_delayed_{0}_{1} /*synthesis keep*/; ZIP __N__, lambda i:i, lambda i:(i-1)%__N__
    DelayLine #(D) DL{0}_{1} (node[{0}], node_delayed_{0}_{1}); ZIP __N__, lambda i:(i-1)%__N__, lambda i:i
    assign node[{0}]= enable ? ~node_delayed_{1}_{0} : init[{0}]; ZIP __N__, lambda i:i, lambda i:(i-1)%__N__
    

The special characters `__N__`, `__T__` etc. will be replaced by the string interpreter in the `FillForm` function. The special flag `ZIP` is a custom function defined in this Python module which formats the text according to lambda functions mapped over an iterable. This produces Verilog assignments automatically at compilation, like so:

```python
s.fill(__N__=3,__T__=8,__D__=1)
print(s.plate)
```

    localparam N=3; //number of nodes
    localparam T=8; //number of measurements
    localparam D=1; //number of inverter pairs between nodes
    wire [N-1:0] node; //ring of cyclic inverters
    wire [N-1:0] init; //initial state of ring
    wire node_delayed_0_2 /*synthesis keep*/; 
    wire node_delayed_1_0 /*synthesis keep*/; 
    wire node_delayed_2_1 /*synthesis keep*/; 
    DelayLine #(D) DL2_0 (node[2], node_delayed_2_0); 
    DelayLine #(D) DL0_1 (node[0], node_delayed_0_1); 
    DelayLine #(D) DL1_2 (node[1], node_delayed_1_2); 
    assign node[0]= enable ? ~node_delayed_2_0 : init[0]; 
    assign node[1]= enable ? ~node_delayed_0_1 : init[1]; 
    assign node[2]= enable ? ~node_delayed_1_2 : init[2]; 
    

These methods however are much more general; see the `sidis` page https://noeloikeau.github.io/sidis/ .

## Devices and Assignments
> Now we want to assign some locations using functions from the `Scripting` page to locations defined on the `Devices` page. 

We have 3 nodes in our ring, let's see if we can place them at LAB (10,10) on the `DE10`:

```python
print(DE10(x=10,y=10),DE10(x=10,y=10,n=0))
```

    LAB LABCELL
    

Looks like we're good. We use `Project.set_loc` to set location assignments to the quartus project settings (.qsf) file:

```python
[P.set_loc(x=10,y=10,n=3*i,name=f'node[{i}]',style='a') for i in range(3)]
```

Which writes 

`set_location_assignment LABCELL_X10_Y10_N0 -to "node[0]"`

`set_location_assignment LABCELL_X10_Y10_N3 -to "node[1]"`

`set_location_assignment LABCELL_X10_Y10_N6 -to "node[2]"`

to a new file `qsf_source.qsf`, and adds the argument `source qsf_source.qsf` to our project `DE10.qsf`, thereby assigning these locations to the `node` logic elements comprising our ring of inverters to the `(10,10)` LAB on the FPGA at the LABCELL coordinates `N=0,3,6`. 

We now compile, program, and archive our project with `Project.Bootup`:

```python
P.compile()
P.program_all()
P.archive()
```

The compilation was successful, the `DE10`'s were each programmed with the same `DE10.sof` file, and a new file `DE10-2020-09-12.qar` was generated, which contains everything needed for someone to boot up our entire project using the Quartus GUI under Tools->Archive. 

## Memory Interface
Let's take a look at our FPGAs and how to interface with their memory instances, as explained in the `Memory` page:

```python
P.get_insts()
```




    [['DE-SoC [USB-1]',
      '@2: 5CSEBA6(.|ES)/5CSEMA6/.. (0x02D020DD)',
      [['0', '1', '1', 'RW', 'ROM/RAM', 'BIT'],
       ['1', '8', '3', 'RW', 'ROM/RAM', 'READ'],
       ['2', '2', '3', 'RW', 'ROM/RAM', 'WRIT']]],
     ['DE-SoC [USB-2]',
      '@2: 5CSEBA6(.|ES)/5CSEMA6/.. (0x02D020DD)',
      [['0', '1', '1', 'RW', 'ROM/RAM', 'BIT'],
       ['1', '8', '3', 'RW', 'ROM/RAM', 'READ'],
       ['2', '2', '3', 'RW', 'ROM/RAM', 'WRIT']]]]



The two FPGA's are connected on ports `USB-1` and `USB-2`. Their names are given by the second string, `5CSEBA6...`. In fact, there are also `SoC`s (system-on-a-chip) connected to these ports, which we could see using `GetPorts`. But we automatically filtered the `SOC`s out, as for our purposes we want to program the FPGAs directly. 

The last arguments in the FPGA lists `[0,1,1,RW,ROM/RAM,BIT]...` are memory instances. They are formatted like:

0. Instance index, used in read/write calls
1. Number of words at that index
2. Number of bits per word
3. Read/write functionality
4. Type of memory (here ROM/RAM)
5. Custom name string defined in Verilog module

Here, instance `0` is an `enable` bit which we use to begin the dynamics of the 3-bit ring. When the enable bit is low, the ring is set to its initial state, specified by instance `2`, which can hold two initial configurations of the 3 node network. When the enable bit is high, the ring begins to invert itself, and the state of each node state is recorded in instance `1` at approximately `1ns` intervals a total of `8` times.

Since we haven't flipped the enable bit or set any initial conditions, the ring should be all 0s on both FPGAs. We check this using Project.ReadAllData:

```python
P.read_all(inst=1)
```




    array([[[0, 0, 0],
            [0, 0, 0],
            [0, 0, 0],
            [0, 0, 0],
            [0, 0, 0],
            [0, 0, 0],
            [0, 0, 0],
            [0, 0, 0]],
    
           [[0, 0, 0],
            [0, 0, 0],
            [0, 0, 0],
            [0, 0, 0],
            [0, 0, 0],
            [0, 0, 0],
            [0, 0, 0],
            [0, 0, 0]]], dtype=uint8)



The first `8x3` column is the state of the 3-node network over 8ns on FPGA `DE-SoC [USB-1]`, the second on `DE-SoC [USB-2]`. This shows us that both of our FPGA rings are held fixed at zero. Now, let's set the initial conditions to `001` on instance `2` using `Project.WriteAllData`:

```python
P.write_all(inst=2,data=[[0,0,1],[0,0,1]]) #We have two initializations, but only the 0th is used
```

Now let's make sure it wrote correctly:

```python
P.read_all(inst=2)
```




    array([[[0, 0, 1],
            [0, 0, 1]],
    
           [[0, 0, 1],
            [0, 0, 1]]], dtype=uint8)



Good. Now, we flip the enable bit to 1:

```python
P.write_all(inst=0,data=[[1]])
```

and read the state of each network over time:

```python
P.read_all(inst=1)
```




    array([[[0, 0, 0],
            [1, 1, 0],
            [1, 0, 1],
            [0, 0, 1],
            [1, 1, 0],
            [1, 0, 1],
            [0, 1, 1],
            [1, 0, 1]],
    
           [[0, 0, 0],
            [0, 1, 0],
            [1, 0, 0],
            [0, 0, 1],
            [0, 1, 0],
            [1, 0, 0],
            [0, 0, 1],
            [0, 0, 0]]], dtype=uint8)



So we managed to create and measure dynamical systems from multiple FPGAs at nanosecond intervals all from Python! This concludes the tutorial, but there are many other possibilities using this package, check out some of the other pages.
