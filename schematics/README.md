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


## Sugar Cane Farm (lite version)

This is a Piston/BUD based sugar cane farm that requires only overworld resources.

The TL;DR is: if you have enough gold to build a batch of powered rails, or have
tons of iron, build either of the [Sugar Cane Farm 2x16 (BUD, no slime)](https://endingcredits.github.io/litematic-viewer/?remote-url=https%3A%2F%2Fgithub.com%2FBradylus%2FMinecraft%2Fraw%2Frefs%2Fheads%2Fmain%2Fschematics%2FSugar%2520Cane%2520Farm%25202x16%2520%28BUD%2C%2520no%2520slime%29.litematic)
and [Sugar Cane Farm 2x16 (BUD, 1 sticky piston)](https://endingcredits.github.io/litematic-viewer/?remote-url=https%3A%2F%2Fgithub.com%2FBradylus%2FMinecraft%2Fraw%2Frefs%2Fheads%2Fmain%2Fschematics%2FSugar%2520Cane%2520Farm%25202x16%2520%28BUD%2C%25201%2520sticky%2520piston%29.litematic)
variants, depending on your luck in getting some slime. On both of these, the hoppers can be
replaced by a hopper minecart (in which case the mud can also be replaced by dirt).

If you're totally broke, go for the [Sugar Cane Farm 2x16 (BUD, no slime, 2 hoppers)](https://endingcredits.github.io/litematic-viewer/?remote-url=https%3A%2F%2Fgithub.com%2FBradylus%2FMinecraft%2Fraw%2Frefs%2Fheads%2Fmain%2Fschematics%2FSugar%2520Cane%2520Farm%25202x16%2520%28BUD%2C%2520no%2520slime%2C%25202%2520hoppers%29.litematic)
variant. It requires only 2 hoppers but loses some items. Upgrading to one of the other two is
pretty straightforward once you have the resources.

The glass blocks can be replaced by top slabs. The white concrete can be
replaced by any solid block. On the no-slime variants, if you ever find that the
pistons get stuck in the extended position, break the topmost repeater (the one
facing the long line of redstone on top), then place it back and set it to a 3
redstone ticks delay (right click twice).

All variants work on the same principle:

- power the pistons without them noticing, thus creating a [block update detector](https://minecraft.wiki/w/Tutorial:Block_update_detector) (also known as a BUD)
- when a sugar cane plant grows before one of them, it gets updated and propagates
  the update to all the others (noticing at that point that they are powered)
- finally, all pistons extend, breaking the sugar cane
- the blob of redstone at the end of the farm resets the BUD

This farm can be chunk-loaded with an ender pearl chunk loader. Just make sure
that all of the sugar cane fits in the same chunk as the chunk loader. Some of
the redstone will be just outside of the central chunk, but this will work just
fine.


## Universal Crafter

[Preview](https://endingcredits.github.io/litematic-viewer/?remote-url=https%3A%2F%2Fgithub.com%2FBradylus%2FMinecraft%2Fraw%2Frefs%2Fheads%2Fmain%2Fschematics%2FUniversal%2520Crafter%252012gt.litematic)

This machine crafts items at a rate of 12 game ticks/item (2/3 hopper speed).
This is intended for the occasional crafting of a few stacks of items.
I personally use two of them, one feeding the other when crafting things like
dispensers (I just rebuild the hopper line between the two when needed). For
bulk crafting, check the Storage Tech or TMC Catalogue Discord servers, you'll
find machines that can craft millions of items per hour.

With this kind of design, it's theoretically possible to achieve hopper speed
by using 5 sides of the crafter for input, but I don't feel like untangling the
resulting noodle soup of droppers. Parallel crafting is a much saner way to
achieve higher speeds.


### How this works

Items are inserted one row at a time every 4 game ticks, which gives one crafted
item every 12 game ticks.

Items are inserted in a crafter from left to right, top to bottom. So the top
left slot is 1st, the top center one is 2nd, and the middle right slot is 6th.

For technical reasons, the item order of the input chests is top to bottom, then
left to right: items in the 3rd chest will end up in the bottom left slot (slot
7), and items in the 8th chest will be inserted in the middle right slot (slot
6). The slot number (and a crude slot map) for each chest is indicated on the
signs above them:

    Chest #:        1    2    3    4    5    6    7    8    9
    
    Crafter Slot#:  1    4    7    2    5    8    3    6    9
                   ■□□  □□□  □□□  □■□  □□□  □□□  □□■  □□□  □□□
                   □□□  ■□□  □□□  □□□  □■□  □□□  □□□  □□■  □□□
                   □□□  □□□  ■□□  □□□  □□□  □■□  □□□  □□□  □□■

Put another way, the first row of items in the crafter comes from chests
1, 4 and 7, the second row from chests 2, 5 and 8, and the last row from
chests 3, 6 and 9.


### Building the machine

This build is not directional, it can be mirrored and rotated as needed. As
usual, concrete blocks can be replaced by any solid full block and glass by
slabs (with some obvious exceptions).

The top 3rd, 6th and 9th droppers (those checked by the comparators) must
contain a single stackable item (signal strength no greater than 1).

The dropper facing a composter below the top dropper line should contain a
single non-compostable item (this is to prevent clicking sounds, although
I'll admit that it's probably useless in this specific case).

For the slot map on the signs that indicate the destination slots for the input
chests, I used the unicode characters ■ (U+25A0) and □ (U+25A1). If you don't
know how to enter these characters, just copy/paste from the example above.

The dropper on the item output from the crafter can be replaced by a barrel.


### Setting up the machine

Before crafting a new recipe:

- Make sure that the machine is in its initial state: the top 3rd, 6th and
  9th redstone lamps must be lit. If not, press Manual Advance (note block on
  the top left) until the proper lamps are lit
- configure the crafter (just beneath the input chests); only disable slots as
  needed, no need to prefill the crafter
- for each disabled slot in the crafter, flick the corresponding lever at the
  back of the machine to extend the piston (this is to have the auto shut-off
  system ignore this slot)
- insert the ingredients in the input chests
- start the machine (note block on the bottom left)

If the machine does not start, check the pistons at the back of the machines:
with all ingredients inserted and disabled slots properly configured, all
pistons should be extended.

Once a crafting batch is complete, the machine will stop in an invalid state.
You will need to press Manual Advance twice to put it back in its initial
state (see below).


### Troubleshooting

The auto-shut off system is too slow to prevent insertion of a partial row of
items. If all ingredients get consumed, the machine will just need to be reset
manually to its initial state before the next batch of items. However, if
it runs out of an ingredient while crafting, the current item recipe will be
invalid. To resume crafting in such a situation:

- Flip the lever below the crafter to the "Purge position"
- Press "Manual Advance" until the machine is in its initial state (3rd, 6th
  and 9th lamps lit)
- Flip the lever below the crafter back to the "Normal Operation" position
- Fill in the missing ingredients (the pistons at the back of the machine will
  be retracted for slots with missing ingredients)
- Press start

In the top row of redstone lamps, the ones that are lit indicate which slots
should have been filled last. If the machine stops because of missing
ingredients, you could just correct the corresponding row in the crafter, add
the missing ingredients, advance manually a couple of times to check that
everything works as expected before starting the machine again. The method
descibed above is however less error prone.

If you mess up while refilling, you may end up with multiple items stuck in the
dropper lines, and the machine will need to be purged to work again:

- remove all input items from the chests, hoppers, and the droppers that are fed
  by the hoppers)
- flick the lever to the purge position
- flick the lever at the back machine to disable the auto-shut off
- start the machine and let it run until no more items come in the purge barrel
- reset both levers to their initial positions
- manually reset the machine to its initial state (3rd, 6th and 9th lamp lit) 

