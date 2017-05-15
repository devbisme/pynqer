---
layout: post
title: Ripping the Lid Off 
date:  2017-05-14 22:08:27
author:
    name: Dave Vandenbout
    photo: devb-pic.jpg
    email: devb@xess.com
    description: Relax, I do this stuff for a living.
category: blog
permalink: blog/ripping-the-lid-off
mathjax: True
---

The obvious place to start exploring PYNQ is to play with its buttons and LEDs.
There's already a [notebook](https://github.com/Xilinx/PYNQ/blob/master/Pynq-Z1/notebooks/examples/board_btns_leds.ipynb)
for that so there's no need for me to replicate it.
A more interesting question is what's going on underneath?
So I'm going to rip the lid off the PYNQ software and start examining its innards.

A good starting point is the `pynq` package:

{% capture content %}{% highlight python %}
import pynq
{% endhighlight %}{% endcapture %}
{% include notebook-cell.html execution_count="[1]:" content=content type='input' %}

You can find the source here:

{% capture content %}{% highlight python %}
pynq.__path__
{% endhighlight %}{% endcapture %}
{% include notebook-cell.html execution_count="[2]:" content=content type='input' %}
{% capture content %}{% highlight text %}
['/opt/python3.6/lib/python3.6/site-packages/pynq']
{% endhighlight %}{% endcapture %}
{% include notebook-cell.html execution_count="[2]:" content=content type='output' %}

The `Overlay` class in the `pynq` package lets you specify a particular overlay that will be loaded into the Programmable Logic (PL) section of the ZYNQ chip.
Xilinx has already provided a pre-compiled overlay that interfaces to the PYNQ's buttons and LEDs (and other things):

{% capture content %}{% highlight python %}
base = pynq.Overlay('base.bit')
{% endhighlight %}{% endcapture %}
{% include notebook-cell.html execution_count="[3]:" content=content type='input' %}

This reads in the bitstream for the overlay, but doesn't yet load it into the PL.
Let's see what's in it:

{% capture content %}{% highlight python %}
dir(base)
{% endhighlight %}{% endcapture %}
{% include notebook-cell.html execution_count="[4]:" content=content type='input' %}
{% capture content %}{% highlight text %}
['__class__',
 '__delattr__',
 '__dict__',
 '__dir__',
 '__doc__',
 '__eq__',
 '__format__',
 '__ge__',
 '__getattribute__',
 '__gt__',
 '__hash__',
 '__init__',
 '__init_subclass__',
 '__le__',
 '__lt__',
 '__module__',
 '__ne__',
 '__new__',
 '__reduce__',
 '__reduce_ex__',
 '__repr__',
 '__setattr__',
 '__sizeof__',
 '__str__',
 '__subclasshook__',
 '__weakref__',
 '_bitfile_name',
 '_gpio_dict',
 '_host',
 '_interrupt_controllers',
 '_interrupt_pins',
 '_ip_dict',
 '_remote',
 '_server',
 '_timestamp',
 'bitfile_name',
 'bitstream',
 'client_request',
 'download',
 'gpio_dict',
 'interrupt_controllers',
 'interrupt_pins',
 'ip_dict',
 'is_loaded',
 'load_ip_data',
 'reset',
 'server_update',
 'setup']
{% endhighlight %}{% endcapture %}
{% include notebook-cell.html execution_count="[4]:" content=content type='output' %}

From this, the actual location of the overlay bitstream is easily found:

{% capture content %}{% highlight python %}
base.bitfile_name
{% endhighlight %}{% endcapture %}
{% include notebook-cell.html execution_count="[5]:" content=content type='input' %}
{% capture content %}{% highlight text %}
'/opt/python3.6/lib/python3.6/site-packages/pynq/bitstream/base.bit'
{% endhighlight %}{% endcapture %}
{% include notebook-cell.html execution_count="[5]:" content=content type='output' %}

If you look in that directory, you'll also see a file called `base.tcl`.
This contains *a lot* of information about the interface between the 
PL and the Processing System (PS) where the PYNQ Python code runs.
One of the things the `Overlay` class does is search through this
file, looking up all the interface information, and loading the
necessary bits into dictionaries in the object it creates.

You would *think* the `gpio_dict` contains the information about accessing
the LEDs and buttons on the PYNQ board. Let's see:

{% capture content %}{% highlight python %}
base.gpio_dict
{% endhighlight %}{% endcapture %}
{% include notebook-cell.html execution_count="[6]:" content=content type='input' %}
{% capture content %}{% highlight text %}
{'audio_path_sel': [3, None],
 'mb_1_intr_ack': [4, None],
 'mb_1_reset': [0, None],
 'mb_2_intr_ack': [5, None],
 'mb_2_reset': [1, None],
 'mb_3_intr_ack': [6, None],
 'mb_3_reset': [2, None]}
{% endhighlight %}{% endcapture %}
{% include notebook-cell.html execution_count="[6]:" content=content type='output' %}

Hmm, not quite what was expected.
Looks like the reset and interrupt acknowledge pins are here, but nothing else.
So the buttons and LEDs don't have direct connections from the PL to the PS.
Let's try `ip_dict`:

{% capture content %}{% highlight python %}
base.ip_dict
{% endhighlight %}{% endcapture %}
{% include notebook-cell.html execution_count="[7]:" content=content type='input' %}
{% capture content %}{% highlight text %}
{'SEG_axi_dma_0_Reg': [2151677952, 65536, None],
 'SEG_axi_dma_0_Reg1': [2151743488, 65536, None],
 'SEG_axi_dynclk_0_reg0': [1136721920, 65536, None],
 'SEG_axi_gpio_video_Reg': [1092747264, 65536, None],
 'SEG_axi_vdma_0_Reg': [1124073472, 65536, None],
 'SEG_btns_gpio_Reg': [1092681728, 65536, None],
 'SEG_d_axi_pdm_1_S_AXI_reg': [1136656384, 65536, None],
 'SEG_hdmi_out_hpd_video_Reg': [1092812800, 65536, None],
 'SEG_mb_bram_ctrl_1_Mem0': [1073741824, 65536, None],
 'SEG_mb_bram_ctrl_2_Mem0': [1107296256, 65536, None],
 'SEG_mb_bram_ctrl_3_Mem0': [1140850688, 65536, None],
 'SEG_rgbled_gpio_Reg': [1092878336, 65536, None],
 'SEG_swsleds_gpio_Reg': [1092616192, 65536, None],
 'SEG_system_interrupts_Reg': [1098907648, 65536, None],
 'SEG_trace_cntrl_0_Reg': [2210398208, 65536, None],
 'SEG_trace_cntrl_0_Reg2': [2210463744, 65536, None],
 'SEG_v_tc_0_Reg': [1136787456, 65536, None],
 'SEG_v_tc_1_Reg': [1136852992, 65536, None]}
{% endhighlight %}{% endcapture %}
{% include notebook-cell.html execution_count="[7]:" content=content type='output' %}

This is more helpful. There are entries that look related to the buttons (`SEG_btns_gpio_Reg`), 
LEDs (`SEG_swsleds_gpio_Reg`), and RGB LEDs (`SEG_rgbled_gpio_Reg`).
But what do the entries mean?

Typing:

    help(base)

provides some information about that (along with quite a bit of other stuff):

    Each entry of the IP dictionary is a mapping:
     |  'name' -> [address, range, state]
     |  
     |  where
     |  name (str) is the key of the entry.
     |  address (int) is the base address of the IP.
     |  range (int) is the address range of the IP.
     |  state (str) is the state information about the IP.
     
This implies that reading the buttons or driving the LEDs is done using a read or write
to a location within a bank of memory addresses.
So the Python code for the buttons and LEDs must contain the instructions for what
particular address offsets and bit locations are used.

The [Python code for the buttons](https://github.com/Xilinx/PYNQ/blob/master/python/pynq/board/button.py) is stored in:

{% capture content %}{% highlight python %}
import pynq.board.button
pynq.board.button.__file__
{% endhighlight %}{% endcapture %}
{% include notebook-cell.html execution_count="[8]:" content=content type='input' %}
{% capture content %}{% highlight text %}
'/opt/python3.6/lib/python3.6/site-packages/pynq/board/button.py'
{% endhighlight %}{% endcapture %}
{% include notebook-cell.html execution_count="[8]:" content=content type='output' %}

There are two important imports in there:

    from pynq import MMIO
    from pynq import PL
    
The `MMIO` class instantiates objects for reading and writing to a segment of memory, and
the `PL` is a singleton object that provides access to the dictionaries of whatever overlay is currently
loaded into the PL of the ZYNQ.
The `__init__` method of a `Button` object uses
the `ip_dict` of the `Overlay` object to initialize an `MMIO` object with the starting
address and size of the address range for the buttons:

    def __init__(self, index):
            if Button._mmio is None:
                Button._mmio = MMIO(PL.ip_dict["SEG_btns_gpio_Reg"][0], 512)
            self.index = index  # This is the bit position of a button in the memory word.
            ...

Then the `Button` object's `read` method will return a
particular button's current state by reading the memory word and masking-off the associated bit:

    def read(self):
        curr_val = Button._mmio.read()  # Read the 1st word of the memory range.
        return (curr_val & (1 << self.index)) >> self.index  # Mask off the bit for this button.

The [Python code for the LED](https://github.com/Xilinx/PYNQ/blob/master/python/pynq/board/led.py)
is very similar except that it writes a bit to the memory address to turn an LED on or off.
(Except that the address for writing the LED values is offset by `0x8` for some unknown reason
that may become apparent later.)

So, if I understand this correctly, I should be able to explicitly use `MMIO` and `PL` to write my own
code for reading the state of a buttons and turning the LEDs on or off.
The code in the following cell can be run and then (for a 10-second interval)
the LED above each button on the PYNQ-Z1 will come on as long as that button is pushed.

{% capture content %}{% highlight python %}
from pynq import Overlay, PL, MMIO

base = Overlay('base.bit')
base.download()             # Load the PL of the ZYNQ with the bitstream for buttons & LEDs.

# Create MMIO objects for reading the buttons and turning the LEDs on and off.
button_addr  = base.ip_dict['SEG_btns_gpio_Reg'][0]
button_range = base.ip_dict['SEG_btns_gpio_Reg'][1]
button_mmio  = MMIO(button_addr, button_range)
led_addr     = base.ip_dict['SEG_swsleds_gpio_Reg'][0]
led_range    = base.ip_dict['SEG_swsleds_gpio_Reg'][1]
led_mmio     = MMIO(led_addr, led_range)

# For a ten-second interval, read the values of all four buttons and
# display it on all four of the LEDs.
from time import time
end = time() + 10.0
while time() < end:
    buttons = button_mmio.read(0)  # Read memory word containing all four button values.
    led_mmio.write(0x8, buttons)   # Write button values to memory word driving all four LEDs.

{% endhighlight %}{% endcapture %}
{% include notebook-cell.html execution_count="[9]:" content=content type='input' %}

The same thing can be done using the higher-level PYNQ software:

{% capture content %}{% highlight python %}
from pynq import Overlay
from pynq.board.button import Button
from pynq.board.led import LED

# Create lists of the buttons and LEDs.
buttons = [Button(i) for i in range(4)]
leds = [LED(i) for i in range(4)]

# For a ten-second interval, execute a loop to read the values of each button and
# display it on the associated LED.
from time import time
end = time() + 10.0
while time() < end:
    for i in range(4):
        leds[i].write( buttons[i].read() )
        
{% endhighlight %}{% endcapture %}
{% include notebook-cell.html execution_count="[10]:" content=content type='input' %}

Obviously, using the higher-level constructs makes the intent of the code clearer, so
why bother with the low-level, explicit approach?
*Because it shows how the magic is done!*
And I'll need to recreate that magic when I build my own PL overlays and interface them to the PS.
An understanding of the underlying code is necessary for that.

You'll notice in `button.py` that it includes some code for handling interrupts.
I'll start tinkering with those next.

## Pertinent Files

Here is a list of the files I examined while making this blog post:

* [`base.bit`](https://github.com/Xilinx/PYNQ/blob/master/Pynq-Z1/bitstream/base.bit): 
  A bitstream that program the ZYNQ's PL with a set of interfaces to the hardware on the PYNQ-Z1 board.
* [`base.tcl`](https://github.com/Xilinx/PYNQ/blob/master/Pynq-Z1/bitstream/base.tcl): 
  Contains a set of definitions for register/memory addresses for the interfaces in the `base` overlay.
* [`pl.py`](https://github.com/Xilinx/PYNQ/blob/master/python/pynq/pl.py): 
  Defines the classes for PL overlays (`PL_Meta`, `PL`, `Bitstream` and `Overlay`). 
* [`mmio.py`](https://github.com/Xilinx/PYNQ/blob/master/python/pynq/mmio.py): 
  Defines the `MMIO` class for reading/writing a segment of memory.
* [`button.py`](https://github.com/Xilinx/PYNQ/blob/master/python/pynq/board/button.py): 
  Defines the `Button` class for reading the state of the pushbuttons on the PYNQ-Z1.
* [`switch.py`](https://github.com/Xilinx/PYNQ/blob/master/python/pynq/board/switch.py): 
  Defines the `Switch` class for reading the state of the slide switches on the PYNQ-Z1.
  (I didn't use these in this post.)
* [`led.py`](https://github.com/Xilinx/PYNQ/blob/master/python/pynq/board/led.py): 
  Defines the `LED` class for changing the state of ON-OFF LEDs on the PYNQ-Z1.
* [`rgbled.py`](https://github.com/Xilinx/PYNQ/blob/master/python/pynq/board/rgbled.py): 
  Defines the `RGBLED` class for changing the state of RGB LEDs on the PYNQ-Z1.
  (I didn't use these in this post.)
* [This Jupyter notebook](https://github.com/xesscorp/pynqer/tree/master/Notebooks/ripping_the_lid_off.ipynb):
  Contains the executable notebook from which this post was generated.


