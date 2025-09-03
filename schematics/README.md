# Schematics


## Redstone Counters

[Preview](https://endingcredits.github.io/litematic-viewer/?remote-url=https%3A%2F%2Fgithub.com%2FBradylus%2FMinecraft%2Fraw%2Frefs%2Fheads%2Fmain%2Fschematics%2FRedstone%2520Counters.litematic)

A set of command blocks to help debug timings in redstone circuits.

These use the scoreboard system. The Minecraft scoreboard is meant to keep track
of player scores for user defined *objectives* (for example to show the number
of deaths for all players). Since player names do not need to be actual players,
we use fake players as variables and two objectives: a *Counters" objective
that will display all variables in the sidebar, and a hidden objective for temp
variables.


### Initialization

Setup a scoreboard objective for our counters (these will be displayed in the
sidebar):

    scoreboard objectives add Counters dummy

Same for temp variables (will remain hidden):

    scoreboard objectives add tmp dummy

Finally, display the *Counters* objective in the sidebar:

    scoreboard objectives setdisplay sidebar Counters


### Measuring clock circuits

This was the original motivation for this. We'll be doing something similar to
the following lua code, but with command blocks:

```lua
Counters = {} -- the displayed counters
tmp = {}      -- temp vars

-- called on every clock cycle
function on_signal()
  Counters.ticks0 = os.time()   -- command block A in the schematic
  Counters.ticks0 -= tmp.ticks0 -- command block B
  tmp.ticks0 = os.time()        -- command block C
end
```

Note that the `on_signal` function is implemented as a single chain of command
blocks where everything happens in the same game tick. As a result, the two
calls to `time query gametime` will return the same value.

Also, the name `ticks0` is completely arbitrary, use whatever name makes sense
for thse measured signals.

Command block A (set to impulse/needs redstone):

    execute store result score ticks0 Counters run time query gametime

Command block B (set to chain/always active):

    scoreboard players operation ticks0 Counters -= ticks0 tmp

Command block C (set to chain/always active):

    execute store result score ticks0 tmp run time query gametime

Caveat: since we don't initialize `tmp.ticks0`, the first result will be wrong.
This can be worked around by wiring a "C" block (set to impulse/needs redstone)
on the clock activation signal. This might also help identify unwanted delays in
the activation signal if the first clock cycle is longer than the next ones.


### Measuring time between two redstone signals

This is useful for long-ish delays, or when pistons+observers are involved.

This time, the lua code looks like this:

```lua
Counters = {} -- the displayed counters
tmp = {}      -- temp vars

-- called on the input signal
function on_input()
   tmp.ticks0 = os.time() -- command block C in the schematic
end

-- called on the output signal
function on_output()
  Counters.ticks0 = os.time()   -- command block A
  Counters.ticks0 -= tmp.ticks0 -- command block B
end
```

Start with a command block wired to the input signal (this is block "C" in the
schematic, configured on impulse and needs redstone):

    execute store result score ticks0 tmp run time query gametime

And on the output, wire the chain of command blocks A and B (configured the same
as for clock timing):

    execute store result score ticks0 Counters run time query gametime
    scoreboard players operation ticks0 Counters -= ticks0 tmp


### Removing the counters

Deleting the *Counters* and *tmp* objectives is enough to reset everything with
a chain of two blocks:

    sscoreboard objectives remove Counters
    scoreboard objectives remove tmp


### Resetting counters

The schematic does not include dedicated command blocks for this. Just delete the
counters then initialize them again to reset all counters.

If really needed, this can be done with the following commands:

    scoreboard players reset * Conters
    scoreboard players reset * tmp

All counters will be removed. To remove a single counter, replace the "*" with
the counter name. For example, to remove a counter named "ticks0":

    scoreboard players reset ticks0 Conters
    scoreboard players reset ticks0 tmp

Just don't forget to remove the counter(s) from both objectives, `Counters` and `tmp`.


### Schematic contents

The schematic contains all these blocks preconfigures (CTRL+Middle Click to copy
them **with** their commands). I'd suggest you save them to your hotbar.

There are also some demos and another example to make an item counter + timer.

See the book on the lectern in the schematic for a quick reference and wiring
instructions.


## Sugar Cane Farm

This is a Piston/BUD based sugar cane farm (no observers needed). There are two versions:

- [Sugar Cane Farm (BUD)](https://endingcredits.github.io/litematic-viewer/?remote-url=https%3A%2F%2Fgithub.com%2FBradylus%2FMinecraft%2Fraw%2Frefs%2Fheads%2Fmain%2Fschematics%2FSugar%2520Cane%2520Farm%2520%28BUD%29.litematic),
  this is the basic model with 15 sugar cane crops. Fairly cheap, requires only
iron and redstone, your "day two" farm.

- [Sugar Cane Farm x31 (BUD)](https://endingcredits.github.io/litematic-viewer/?remote-url=https%3A%2F%2Fgithub.com%2FBradylus%2FMinecraft%2Fraw%2Frefs%2Fheads%2Fmain%2Fschematics%2FSugar%2520Cane%2520Farm%2520x31%2520%28BUD%29.litematic)
is 31 blocks long and requires a slime block. This is just a demo to show
another way to implement the block update detector. This one is actually a
bit faster. 

The glass blocks can be replaced by any solid block or top slab. The white
concrete can be replaced by any solid block.

I'd recommend building only the small variant and packing a few of them in a
single chunk with an ender pearl chunk loader.


## Universal Crafter

[Preview](https://endingcredits.github.io/litematic-viewer/?remote-url=https%3A%2F%2Fgithub.com%2FBradylus%2FMinecraft%2Fraw%2Frefs%2Fheads%2Fmain%2Fschematics%2FUniversal%2520Crafter%252012gt.litematic)

This contraption crafts items at a rate of 12 game ticks/item (that's 2/3 hopper speed).

TODO: how it's working, etc.

TODO: This requires some setup before use.

TODO: troubleshooting, and what to do on ingredient shortage.

It's theoretically possible to achieve hopper speed by using 5 sides of the crafter for input, but
I don't feel like untangling the resulting noodle soup of droppers.
