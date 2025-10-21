# Bamboo Farm 1.21+

This is a flying-machine based bamboo farm that should also work for any other
tall plant, even cacti.

For sustained production of large quantities of bamboo, bonemeal based farms
are not necessarily a good option because of the fairly high MSPT cost of the
bonemeal farms themselves.

## Features

The main design goal of this farm was to make it as lag efficient as possible.
A farm of 7 layers × 7 chunks yields about 140 000 bamboo per hour at the cost
of ~1.5 MSPT on an antique AMD FX-6300, and 1.75 when converting the bamboo to
planks (31K planks/h).

- 2-way harvesting: bamboo grows fairly fast, as a result, a return trip
  without harvesting would cause a lot of entities lying around.
- Flying machines only carry a single harvesting "blade": 2-way harvesters
  usually push one blade while pulling the one for the reverse direction.
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

Rince and repeat on each side of the farm. The major issue is that the bottom
most harvester must have left its station before the top one comes back,
otherwise, the top one would pick up the next set of minecarts (or just run
into them as they drop, depending on the timings) and the bottom one would never
leave.

The solution is to wait for the bottom harvester to arrive on one side of the
farm before starting to dispense minecarts on that same side. Neat and simple,
the only issue being that while this bottom harvester is coming back, no others
are moving (we'll see why this matters later). This can be worked around by
allowing the topmost harvester to move when the second to last one arrives, or
third to last and so on, which one to pick depending on the farm lenght, number
of layers and delay between harvesters. Which brings us to the subject of…

**TODO:** may be a word about the signal propagation system for lauching the
harvesters. I have a feeling it could be simpler.


## Optimal rates

With a two way flying machines, we cannot have it harvest plants at the ends
of the farm at even intervals: if we simplify to the extreme, the first plant
will be harvested immediately, then after a full trip back and forth, then
immediately again, etc. The plant at the center of the farm will however be
harvested at fixed intervals, so we'll use that one as a target to adjust the
timings.

Bamboo grows on average once every 4096 game ticks, so the maximum achievable
rate is: 72000 / 4096 = 17.58 bamboo/hour/plant

A reasonable choice is to try to pass over the center point of the farm every
4096 game ticks. Tests showed that by doing this, we get 15.55 items/h/plant,
which is about 11.5% less than the max rate. This difference is caused by the
fact that plants are not allowed to grow taller than two blocks and of the
uneven harvesting intervals. Higher rates are possible by shortening the delays
at the cost of a much higher MSPT; or make the farm bigger, at a lesser
performance cost.

TODO: move this to last: A farm made of a single row of 12 blocks × 7 chunks × 16 blocks × 7 layers has
9408 plants (-6 per layer because of redstone but let's ignore this), so the
maximum achievable rate should be about 164 600. The actual rates we get are 145700, so we're just 11.5% below the max.

Getting the timings right:

- let `L` be the length of the planted area
- let `T` be the travel time for a flying machine from its rest position to its
  rest position on the opposite side. This is equal to `(L+24 blocks)×10
  ticks/block`.
- let `D` be the delay between two flying machines.
- let `N` be the number of layers
- let `B` be the number of bottom layers. i.e. when the harvester at layer
  N-B arrives, the topmost harvester is launched again. `1 ≤ B < N`
- let `D0 = 275 + 3(N-B)` be the time between the bottom flying machine arriving
  and the top one departing. We'll however ignore the `3(N-B)` part since it's
  really insignificant; this will also simplify some of the math.

The harvesting interval `I` for the midpoint of the farm is `T + D(N-B) + D0`.
Solving for D:

    I = T + D(N-B) + D0
    => D = (I-D0-T)/(N-B)

While the number of layers `N` and field length `L` depend entirely on the
desired yield, we want to adjust `B` so that we have as few harvesters moving
simultaneously as possible. The lower limit for `B` is 1 (safe) but the upper
limit is where the bottom-most harvester has left its station before the
returning topmost harvester enters the minecart drop area.

Travel time for the top harvester to the opposite side and back to the drop
area (use 16 blocks as a safe margin):

    t1 = 2T - 160 + D(N-B) + D0

Departure time of harvester at layer `N`:

    t0 = D(N-1) + D0

Solve `t0 < t1`:

    t0 < t1 <=> D(N-1) + D0 < 2T + D(N-B) + D0 - 160
    ⇔ DN - D < 2T + DN - DB - 160
    ⇔ DB < 2T - 160 + D
    ⇔ B < (2T-160)/D + 1

Subtitute `D`:

    B < (2T-160)/D + 1
    ⇔ B < (2T-160)×(N-B)/(I-D0-T) + 1

And solve for `B`:

    ⇔ B < (N-1)(2T-160) / (I+T-D0-160) + 1
    ⇔ B < (N-1)(2T-160) / (3661+T) + 1

### TL;DR
    
Here we are, chose `B` such that it is the integer part of `(N-1)(2T-160)/(3661+T) + 1`
then calculating `D` with the formula `D = (3821-T)/(N-B)` should give the
delay for optimal rates vs. lag efficiency.

The number of items in the top hopper clock on both sides of the farm should be `(D+12)/16`.

Since the calculate value of `D` will not likely be a nice round integer, you'll
have to round this down (not up! This is just a simple rule of thumb to avoid
edge cases).


### Maximum farm size

Also, the minimum value for `D` should be around 180 gt (12 items in the hopper
clocks), so that harvesters have fully cleared the station before the next one
starts. If you get delays that low, `L` or `N` are very likely too high anyways.

The maximum length of a farm is such that:

    T + D(N-B) + D0 ≤ I
    ⇔ (L+24)×10 + D(N-B) + D0 ≤ I
    ⇔ L ≤ (I - 275 - 240)/10 - D(N-B)/10
    ⇔ L ≤ 358.1 - D(N-B)/10

For a single layer farm where `D` and `D(N-B)` are 0, the maximum length is 358
blocks (> 22 chunks). For more layers,

TODO: harder to work out... for two layers, with D = 180, we get... half that?
L = 179, T = 2030, B = ... uh... the math is surprising. A spreadsheet maybe?


## Notes

Like all farms of this kind, it needs to be orientated East-West so that the
hopper-carts stay in place.

Using a single "blade" saves over 30% of the MSPT caused by the flying machine
itself (not including the hopper-carts). This was also a personnal challenge ;)

The schematic is only a showcase with a 7 layers × 7 chunks version. It can be
extended up/down and in length as needed (also add chunk loaders!), and a triple
width version should fit in the 3 chunks covered by the chunk loaders (make sure
to include the minecart collection and dispenser in these).

I'd like to be able to fit a quadruple version, but I'd need to redisign part
of the minecart collection and dispenser to make it fit in the same chunk as
the flying machine holding stations.

The rates of the 7×7 are suboptimal. I need to fine tune the timings and work
out the best strategy for expansion. Lenght-wise, I'd prefer to keep it no
more than 9 chunks long so that should the chunk loaders fail, the player can
still keep the whole thing loaded with the prety standard 8 chunks simulation
distance.


## References

TODO: link to Ilmango's sugar cane farms, and the thing to place blocks.
