# Bamboo Farm 1.21+

[Preview](https://endingcredits.github.io/litematic-viewer/?remote-url=https%3A%2F%2Fgithub.com%2FBradylus%2FMinecraft%2Fraw%2Frefs%2Fheads%2Fmain%2Fschematics%2FBamboo%2520Farm%2FBamboo%2520Farm%25207x6x1.litematic)

This is a flying-machine based bamboo farm that should also work for any other
tall plant, even cacti.

For sustained production of large quantities of bamboo, bonemeal based farms
are not necessarily a good option because of the fairly high MSPT cost of the
bonemeal farms themselves.

This farm is very much inspired by Ilmango's [Improved Jumping Hopper Minecart
Bamboo Farm] but with 12 blocks wide blades and running at a much lower MSPT for
the same production rates (see the [References section](#references)).


## Features

The main design goal was to make it as lag efficient as possible. A farm of 7
chunks × 7 layers × 12 blocks wide yields about 146 000 bamboo per hour at the
cost of ~1.5 MSPT on an antique AMD FX-6300, and 1.75 when converting the bamboo
to planks (31K planks/h).

- 2-way harvesting: bamboo grows fairly fast, as a result, a return trip
  without harvesting would cause a lot of entities lying around + cycles wasted
  on harvesting nothing.
- Flying machines only carry a single harvesting "blade": 2-way harvesters
  usually push one blade while pulling the one for the reverse direction.
  This saves about 30% on the MSPT caused by the flying machines themselves.
- Hopper minecarts are only dispensed when needed.


## Technical details

Basic operation:

- Hopper minecarts are dispensed from the top of the farm, 12 at a time
- Once the last mincart is in position, they all drop on the topmost harvester
  and a signal is sent down to get that harvester moving (only the topmost
  harvester receives this signal)
- Wait for a set delay (short enough to allow several harvesters to be moving
  simultaneously)
- Repeat until all harvesters have left
- On the other side of the farm, wait long enough to harvest only every 4096
  game ticks, then restart the whole process.

The mincart dispenser is based on a simple item counter that dispenses exactly
12 minecarts. For a wider farm, make sure to change the number of items in the
dropper (# of minecarts to be dispensed minus 2).

The farm is controlled by a delay circuit (pulse extender) and a clock, both
replicated on each side of the farm, behind the minecart dispensers. The delay
circuit controls how long harvesters will wait before moving in the other
direction, and the clock the interval between each harvester (actual timings are
discussed in the [Optimal Rates](#optimal-rates) section).

When a harvester arrives at an end station, there's a line of slime/honey
blocks waiting for it and that will be pulled just in time to grab the trapdoors
holding the minecarts (this happens after the minecarts have already been
dropped), forming the blade to be pushed forward for the way back. The harvester
still pushes its now trapdoorless blade, but this will be grabbed as it leaves
the station (just watch it in action, this will probably make more sense).

The signal to start harvesters is sent from the clock at the top, propagating
downwards until it reaches a layer with a harvester present, stopping the signal
propagation. Signal redirection is done by moving a solid block over the desired
signal path.


## Optimal rates

Bamboo grows on average once every 4096 game ticks, so the maximum achievable
rate is: 72000 / 4096 = 17.58 bamboo/hour/plant.

One major caveat of two-way flying machines is that the plants at the ends of
the farm are harvested at uneven intervals: simplifying to the extreme, the
first plant is harvested immediately, then after a full trip back and forth,
then immediately again, etc.

With this in mind, a reasonable choice is to try to harvest the center of
the farm every 4096 game ticks. Tests showed that by doing this, we get 15.58
items/h/plant, which is about 11.3% less than the max rate. This difference is
caused by the fact that plants are not allowed to grow taller than two blocks
and because of the uneven harvesting intervals. Higher rates are possible by
shortening the delays at the cost of higher MSPT; or make the farm bigger with
an additional layer, at a lesser performance cost.


### Getting the timings right

- let `l` be the length of the planted area
- let `t` be the travel time for a flying machine from its rest position to its
  rest position on the opposite side. This is equal to `(l+24 blocks)×10
  ticks/block` (the flying machines travel at speed of 0.1 block/tick).
- let `w` be the waiting time for a flying machine before it is sent back
- let `h` be the delay between two flying machines
- let `d0` be the time between the delay clock ticking and the actual
  departure time of the top harvester. On a 12 blocks wide farm, this is 274
  game ticks.
- let `n` be the number of layers
- let `i` be the harvesting interval for the midpoint of the farm.

Although harvesting intervals at the ends of the farm are uneven, the plant
located at the middle point is harvested at even intervals. Using this point as
refernce, this gives: 

    i = t/2 + w + d0 + t/2
    ⇔ w = i - t - d0

Harvesting the middle point every 4096 game ticks:

    w = i - 240 - d0 - 10×l 
    w = 3582 - 10×l

The number of items in the delay circuit is `⌈(w-2)/16⌉` (round up). For a 7
chunks long farm, 112 blocks, we get 153.75, rounded up gives 154, that's 2
stacks and 26 items. Rounding up allows us to simplify the maths when
calculating the other delays without any side effects.

`h` should be as long as possible in order to minimize the number of harvesters
moving simultaneously. If it's too long however, the topmost harvester would
come back before the bottom one has left and take the next set of minecarts (and
the bottom one would never leave) or one of the harvesters would collide whith
minecarts dropping for (or from) the other. So `h` should be short enough for
the bottommost harvester to have left its station and cleared the minecart drop
area before the top one enters that same area.

Departure time of the bottom harvester + 12 blocks of clearance (120 ticks travel
time). Note that `d0` for the bottom layer is d0 + 4(n-1):

    t0 = h(n-1) + d0 + 4(n-1) + 120
    ⇔ t0 = (h+4)(n-1) + d0 + 120

Time for the top harvester to do a full two way trip, minus 12 blocks:

    t1 = d0 + 2×t + w + d0 - 120
    ⇔ t1 = 20×l + 2×d0 + 480 - 120 + i - 240 - d0 - 10×l
    ⇔ t1 = 10×l + d0 + 120 + i

Solve `t0 ≤ t1` for `h`:

    (h+4)(n-1) + d0 + 120 ≤ 10×l + d0 + 120 + i
    ⇔ (h+4)(n-1) ≤ 10×l + i
    ⇔ h ≤ (10×l+4096)/(n-1) - 4

The number of items in the hopper clock is `⌊(h+12)/16⌋` (round down this time).

<details>
<summary>Sacrificing MSPT for higher rates…</summary>
For higher rates without making the farm larger, let's see the timings so that
the far ends of the farm are harvested at most every 4096 game ticks:

    i = 10×(l+24) + w + d0 + 10×l
    ⇔ w = 3582 - 20×l

    t1 = d0 + 2×t + w + d0 - 120 = 4490

    t0 ≤ t1 ⇔ (h+4)(n-1) + 394 ≤ 4490
            ⇔ h ≤ 4096 / (n-1) - 4

Interestingly, `h` only depends on the number of layers in this case. With
these settings, the 7×7×12 farm used as a testbed runs at 2 MSPT (instead of
1.5), with rates around 152.5K items/h (vs. 146K, +4.4%), or 16.28 items/h/plant
(7.4% below the max rate).

With that same farm, adding an 8th layer instead of lowering the delays would
linearly increase the rates to 166.9K with only +0.2 MSPT.

In conclusion, I wouldn't recommend shortening the delays unless you've reached
the maximum number of layers already and need to squeeze a few more thouthands
items out of the farm.
</details>


### Farm size

When speaking of farm length, I only consider the planted area, the harvester
stations take one additional chunk on each end.

For the maximum farm length, `w` must be greater than 0, so `l ≤ 359` which is
just a bit over 22 chunks. However, hopper minecarts can only carry 320 items.
With `i` set to 4096 game ticks, the average amount of bamboo collected per
run would be ~377, which does not fit. Also if the bamboo is somehow allowed to
grow without harvesting, on the first run you'd end up with 12×12×16 bamboo not
collected per layer, and as many entities lying around for the next 10 minutes,
causing a serious lag spike.

When planning your farm size, I'd recommend making it as long as the simulation
distance so that a player can keep the farm loaded while standing at either end
should the chunk loaders fail, but no longer than 10 chunks.

A 10 chunks long farm (12 chunks total), would do nicely in single player where
the default render distance is 12 chunks. In multiplayer, with most servers
configured for an 8 chunks simulation distance, I'd recommend a 7 chunks long
farm (+2 for the stations); just make sure that the chunk loaders are running
while putting the final touches to your farm.

While the rates (and MSPT) increase linearly with each layer, extending the
length of the farm will increase the rates with diminishing returns.

Here are the rates for some configurations I have tested (all are only 12 blocks
wide):

| Field length | Total length | Layers | Rates | items/h/plant | MSPT |
| -----------: | -----------: | -----: | ----: | ------------: | ---: |
|          112 |          144 |      6 | 125.0K | 15.55        |  1.4 |
|          112 |          144 |      7 | 146.0K | 15.55        |  1.5 |
|          112 |          144 |      8 | 166.8K | 15.55        |  1.7 |
|          112 |          144 |     10 | 208.5K | 15.55        |  2.0 |
|          160 |          196 |      7 | 207.3K | 15.42        |  2.3 |

The last two have the same total surface area, however the 7 chunks long version
has higher rates for a lower MSPT.

The minimum value for `h` is 212 game ticks (14 items in the hopper clock).
This is the time needed for the minecart dispenser to do a full run and reset.
So the maximum number of layers is:

    (10×l + 4096)/(n-1) - 4 ≥ 212
    ⇔ n ≤ (10×l + 4312)/216

For a 7 chunks long farm, that's 25 layers (~521K bamboo/hour) and 27 for a
10 chunks farm (800K bamboo/hour).


## Chunk loaders

When shutting off the farm, the chunk loaders must be kept running until the
last harvester launched comes back. The delay `c` corresponds the the worst
case (a full 2-way trip) plus 200 game ticks (10s) to collect the minecarts:

    c = 2(d0 + t) + w + 200
    ⇔ c = 10×l + 4810

For the *overclocked* version where we harvest the farm ends every 4096 game
ticks, that's just 4810, regardless of the farm size.

The pulse extender for the chunk loaders is a double delay version of the usual
hopper clock pulse extender. The total number of items should be `⌈(c + 12)/32⌉`
(round up).


## Notes

Like all farms of this kind, it needs to be orientated East-West so that the
hopper-carts stay in place.

The schematic is only a demo with a 7 chunks × 6 layers × 12 blocks version. It
is more than enough to feed a medium furnace array (up to 115 furnaces) with
bamboo planks. I intend to include a 10 chunks × 27 layers × 36 at some point to
show how to do it.

I'd like to be able to fit a quadruple version, but I'd need to redesign part
of the minecart collection and dispenser to make it fit in the same chunk as
the flying machine holding stations.

The bottom piston of the double extender that swaps the trapdoors needs to stay
extended so that the flying machine of the next layer down does not stick to it.
It's working flawlessly, yet it could probably be made simpler (like having both
pistons fully extended?).

Minecart dispensing happens at 10gt/minecart. Couldn't get it to work at hopper
speed. At that speed, the observers/traps beneath the pistons pushing the rails
could be replaced by a pair of torches if it did not conflict with the item
counter's wiring. Making it work at hopper speed with a better item counter
could help shorten the minimum delay between harvesters and increase the maximum
number of layers.

When setting up the delay circuits, make sure to prevent them from triggering
unexpectedly by powering the non-sticky piston with a redstone block.

When building the item counter, place the items in the dropper only once the
full circuit is built (the comparator should not output any signal).

I did not test with more than 10 layers, but I'm pretty sure that the current
delay circuit (the 2 comparators wired to the last minecart dropping) is not
enough for much more layers. This needs to be tested and `d0` updated
accordingly. 

The item collection system at the bottom is unfiltered because the input is
irregular and even for a farm producing only 146K/h, filtering would require
enough filters to handle ~21 stacks of items at once (~58 filters). Even without
filtering, there's a system to push item entities over the hoppers every 8 gt
in order to limit the number of hoppers required. Filtering the output from the
crafters is much less of an problem (only 16200 bamboo blocks/h), and should
some unwanted item get into the crafters, this is a only a minor annoyance and
an easy fix.

To fully filter the output, A 500K bamboo/hour farm, requires at least 28 double
speed filters, and to accomodate for the irregular input rate, one solution
would be to buffer the entites and use an entity separation system to send ~64
items over the hoppers every 8gt.

Speaking of unwanted items in the farm, I'd recommend building it over a deep
ocean, at least 32 blocks away from the ocean floor and any land so that no
endermen can get into the farm and steal farmland…

To place the dirt and glass, Glowsquid's [Automatic Floor Builder] should come
in handy.


## References

As mentionned in the intro, this farm is heavily based on tall plant farms by
Ilmango:
- [Mega Base Friendly Sugar Cane Farm]: reused the minecart distribution system
  from this farm as it made the stackable farm cheaper to build and the minecart
  handling easier.
- [Improved Jumping Hopper Minecart Bamboo Farm]: for the idea to have the
  flying machine move a single blade.
- [10 Million Bamboo Per Hour]: make the farm stackable.

Ilmango went for a solution with narrower blades and trapdoors to hold the
hopper minecarts on both sides of the blade. I really wanted to find a way to
keep the 12 blocks blade and flip the trapdoors around. A couple of years ago,
I made a working design based on the idea of looping over two layers (see
[10 Million Bamboo Per Hour]), but it ended up not being easily stackable, and
was a nightmare to build. So I went back to the drawing board and came up with
this design. With careful adjustment of the timings, it has similar rates to a
looping version and is stackable every 5 blocks with a lower overall MSPT.

[Mega Base Friendly Sugar Cane Farm]: https://www.youtube.com/watch?v=-mzdU-mk7JA
[Improved Jumping Hopper Minecart Bamboo Farm]: https://www.youtube.com/watch?v=yzwRwlwWlz0
[10 Million Bamboo Per Hour]: https://www.youtube.com/watch?v=xlEnKWqXOdI
[Automatic Floor Builder]: https://www.youtube.com/watch?v=aA4V4Ws8_ig
