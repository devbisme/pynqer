# README

The [ZYNQ devices](https://www.xilinx.com/products/silicon-devices/soc/zynq-7000.html)
from Xilinx combine an ARM processor (or 2, or 4) with a
field programmable logic array (FPGA) to create what they call an *All Programmable Systems on Chip* (APSoC).
Because the processor and FPGA are in the same package, there's a high-bandwidth
communication channel between them.
This makes it feasible to create specialized coprocessors in the FPGA
to offload compute-intensive tasks from the processor.

Unfortunately, designing something like that requires a breadth of knowledge
about embedded programming and logic design techniques as well as
experience with the various tool flows.
That's uncommon.

In order to ease the onboarding process for engineers new to ZYNQ,
Xilinx created the [PYNQ system](http://www.pynq.io/home.html)
consisting of:

* A PYNQ board containing a ZYNQ APSoC.
* A [Jupyter notebook interface](http://jupyter.org/) that runs under linux on the ZYNQ's ARM processor.
* A set of *overlays* that can be loaded using Jupyter to program the
  ZYNQ's FPGA with various coprocessors.

With these components, it's possible to quickly experiment with various
coprocessors and see how they interact with the ARM by using the Jupyter
environment to control the PYNQ board and query the results of computations.

[This site](http://xesscorp.github.io/pynqer) will record my experiences as I explore the PYNQ system.
I don't intend this as a tutorial on using Zynq or Pynq: there are
plenty of those already and I'll reference them as needed.
Rather, I'll use these posts to tamp the things I learn about PYNQ into my
own head.
As such, I'll create blog posts describing my experiments and I'll store
the design files in the Github repository.
Feel free to use them as they become available: everything is open source.


* Website: [http://xesscorp.github.io/pynqer](http://xesscorp.github.io/pynqer)
