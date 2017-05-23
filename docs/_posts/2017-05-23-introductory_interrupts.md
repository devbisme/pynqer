---
layout: post
title: Introductory Interrupts
date:  2017-05-23 14:31:28
author:
    name: Dave Vandenbout
    photo: devb-pic.jpg
    email: devb@xess.com
    description: Relax, I do this stuff for a living.
category: blog
permalink: blog/introductory-interrupts
mathjax: True
---
# Introductory Interrupts

My [previous blog post](https://xesscorp.github.io/pynqer/docs/_site/blog/ripping-the-lid-off)
showed how I used *polling* to get the state of the PYNQ's pushbuttons and display them on the LEDs.
That's a great way to burn through CPU cycles.

Anyone who's programmed embedded microcontrollers knows *interrupts* provide
a more efficient solution that only checks the buttons when they change state.
Xilinx provides
[this PYNQ demo](https://github.com/Xilinx/PYNQ/blob/master/Pynq-Z1/notebooks/examples/asyncio_buttons.ipynb)
that couples the ZYNQ interrupt hardware with Python's
[asyncio](https://pymotw.com/3/asyncio/index.html) features.

Unfortunately, I'm not very familiar with `asyncio`, and
[this post](http://lucumr.pocoo.org/2016/10/30/i-dont-understand-asyncio/)
didn't fill me with confidence.
But, eventually, I picked up enough to understand the basic concepts of an
explicit *event loop* that runs a set of *coroutines* encapsulated in *tasks*.

Then I got confused again because I couldn't see where the Xilinx example
started the event loop.
It turns out that's hidden in the `wait_for_value()` method of the
[`Switch` object](https://github.com/Xilinx/PYNQ/blob/master/python/pynq/board/switch.py):

```py
def wait_for_value(self, value):

    # Abort if the interrupt hardware isn't present in this overlay.
    if self.interrupt is None:
        raise RuntimeError('Interrupts not available in this Overlay')

    # Get the default event loop.
    loop = asyncio.get_event_loop()
    
    # Encapsulate the wait_for_value_async() coroutine in a task and run it
    # in the event loop until this button has the desired value.
    loop.run_until_complete(asyncio.ensure_future(self.wait_for_value_async(value)))
```

Now if I just call `switch[0].wait_for_value(1)`, the event loop in the following cell will run
until I flip the switch:

{% capture content %}{% highlight python %}
from pynq import Overlay
from pynq.board import Switch

# Make sure the base overlay is installed in the ZYNQ PL.
Overlay('base.bit').download()

sw0 = Switch(0)        # Create Switch object for SW0.
sw0.wait_for_value(1)  # Push SW0 up to terminate this cell.
print('SW0 is 1!')
{% endhighlight %}{% endcapture %}
{% include notebook-cell.html execution_count="[1]:" content=content type='input' %}
{% capture content %}{% highlight text %}
SW0 is 1!
{% endhighlight %}{% endcapture %}
{% include notebook-cell.html execution_count="[1]:" content=content type='output' %}

How is this any different than just polling?
The answer lies in the `wait_for_value_async()` coroutine.
It contains a loop, but it's a loop that only runs whenever the interrupt circuitry detects a change
in the switch state:

```py
@asyncio.coroutine  # Make the following function into a coroutine that runs in an event loop.
def wait_for_value_async(self, value):

    # Abort if the overlay has no interrupt circuitry (see the next method).
    if self.interrupt is None:
        raise RuntimeError('Interrupts not available in this Overlay')
        
    # Only exit this loop when this switch has the desired value.
    while self.read() != value:
    
        # Pause this coroutine until one of the switches changes state and causes an interrupt.
        yield from self.interrupt.wait()
        
        # If one of the switches caused the interrupt, then reset the interrupt flag.
        if Switch._mmio.read(0x120) & 0x1:
            Switch._mmio.write(0x120, 0x00000001)
```

The interrupt hardware is setup in the `__init__()` method of the `Switch` object:

```py
def __init__(self, index):

    # Create the MMIO object that has register addresses for reading the switch state.
    if Switch._mmio is None:
        Switch._mmio = MMIO(PL.ip_dict["SEG_swsleds_gpio_Reg"][0], 512)
        
    self.index = index  # The index for this switch (either 0 or 1 for the PYNQ).
    
    # Setup the interrupt hardware.
    self.interrupt = None  # No interrupts by default.
    try:
        # Create the interrupt object using info from the overlay.
        self.interrupt = Interrupt('swsleds_gpio/ip2intc_irpt')
        
        # Enable the interrupts using register addresses in the switch MMIO.
        Switch._mmio.write(0x11C, 0x80000000)
        Switch._mmio.write(0x128, 0x00000001)

    except ValueError as err:
        print(err)
```

There are a few mysteries in the code shown above, such as where the addresses
for querying and clearing the switch interrupts come from.
(I suspect that will be answered by diving into the HDL code for the `base` overlay.)
Also, I haven't looked into the operations of the `Interrupt` class.

But I've seen enough to replicate handling the switch interrupts.
The following code creates tasks that scan the switches whenever
an interrupt happens.
A separate task runs for a set time interval after which
the scanning stops and the CPU utilization over that interval is displayed.
(Note that in this code I've used the new `async` and `await` keywords
in place of `asyncio.coroutine` and `yield from`, respectively.)
So just run the following cell and see what happens.

{% capture content %}{% highlight python %}
import asyncio
from psutil import cpu_percent
from pynq import Overlay
from pynq.board import Switch

# Make sure the base overlay is installed in the ZYNQ PL.
Overlay('base.bit').download()

# Create objects for both slide switches.
switches = [Switch(i) for i in range(2)]

# Coroutine that waits for a switch to change state.
async def show_switch(sw):
    while True:

        # Wait for the switch to change and then print its state.
        await sw.interrupt.wait()  # Wait for the interrupt to happen.
        print('Switch[{num}] = {val}'.format(num=sw.index, val=sw.read()))

        # Clear the interrupt.
        if Switch._mmio.read(0x120) & 0x1:
            Switch._mmio.write(0x120, 0x00000001)

# Create a task for each switch using the coroutine and place them on the event loop.
tasks = [asyncio.ensure_future(show_switch(sw)) for sw in switches]
    
# Create a simple coroutine that just waits for a time interval to expire.
async def just_wait(interval):
    await asyncio.sleep(interval)

# Run the event loop until the time interval expires,
# printing the switch values as they change.
time_interval = 10  # time in seconds
loop = asyncio.get_event_loop()
wait_task = asyncio.ensure_future(just_wait(time_interval))

# Surround the event loop with functions to record CPU utilization.
cpu_percent(percpu=True)  # Initialize the CPU monitoring.
loop.run_until_complete(wait_task)
cpu_used = cpu_percent(percpu=True)

# Print the CPU utilization % for the interval.
print('CPU Utilization = {cpu_used}'.format(**locals()))

# Remove all the tasks from the event loop.
for t in tasks:
    t.cancel()
{% endhighlight %}{% endcapture %}
{% include notebook-cell.html execution_count="[2]:" content=content type='input' %}
{% capture content %}{% highlight text %}
Switch[0] = 1
Switch[1] = 0
Switch[0] = 1
Switch[1] = 0
Switch[0] = 0
Switch[1] = 0
Switch[0] = 0
Switch[1] = 0
Switch[0] = 0
Switch[1] = 1
Switch[0] = 0
Switch[1] = 1
Switch[0] = 0
Switch[1] = 0
Switch[0] = 0
Switch[1] = 1
Switch[0] = 0
Switch[1] = 1
Switch[0] = 0
Switch[1] = 0
Switch[0] = 0
Switch[1] = 0
CPU Utilization = [0.8, 0.3]
{% endhighlight %}{% endcapture %}
{% include notebook-cell.html execution_count="[2]:" content=content type='output' %}

While the code was running, I flipped each switch once.
For some reason, each transition of a switch caused the interrupt to be serviced twice.
(I still haven't figured that out.)

At the end of the prescribed time interval, the utilization of each CPU is shown to
be less than 1%.
To compare this with the use of polling, I wrote the following code that scans each switch continuously:

{% capture content %}{% highlight python %}
def scan_switch(sw):
    try:
        sw_val = sw.read()  # Get the switch state.
        
        # Print the switch state if it has changed.
        if sw.prev != sw_val:
            print('Switch[{num}] = {val}'.format(num=sw.index, val=sw_val))
            
    except AttributeError:
        # An exception occurs the 1st time thru because the switch state
        # hasn't yet been stored in the object as an attribute.
        pass
    
    # Save the current state of the switch inside the switch object.
    sw.prev = sw_val

# Compute the end time for the polling.
from time import time
end = time() + 10.0

cpu_percent(percpu=True)  # Initialize the CPU monitoring.

# Now poll the switches for the given time interval.
while time() < end:
    for sw in switches:
        scan_switch(sw)
        
# Print the CPU utilization during the polling.
cpu_used = cpu_percent(percpu=True)
print('CPU Utilization = {cpu_used}'.format(**locals()))
{% endhighlight %}{% endcapture %}
{% include notebook-cell.html execution_count="[3]:" content=content type='input' %}
{% capture content %}{% highlight text %}
Switch[0] = 1
Switch[0] = 0
Switch[1] = 1
Switch[1] = 0
CPU Utilization = [0.5, 99.8]
{% endhighlight %}{% endcapture %}
{% include notebook-cell.html execution_count="[3]:" content=content type='output' %}

Once again, I flipped each switch while the code was running.
Only now the utilization is near 100% for one of the CPUs, showing the 
interrupt-based code is much more efficient than polling.

## Pertinent Files

Here is a list of the files I examined while making this blog post:

* [`switch.py`](https://github.com/Xilinx/PYNQ/blob/master/python/pynq/board/switch.py): 
  Defines the `Switch` class for reading the state of the slide switches and handling their interrupts.
* [This Jupyter notebook](https://github.com/xesscorp/pynqer/tree/master/Notebooks/introductory_interrupts.ipynb):
  Contains the executable notebook from which this post was generated.
