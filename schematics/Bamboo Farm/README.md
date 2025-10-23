# Bamboo Farm 1.21+

This is a flying-machine based bamboo farm that should also work for any other
tall plant, even cacti.

For sustained production of large quantities of bamboo, bonemeal based farms
are not necessarily a good option because of the fairly high MSPT cost of the
bonemeal farms themselves.

## Features

The main design goal of this farm was to make it as lag efficient as possible.
A farm of 7 chunks × 7 layers × 12 blocks wide yields about 146 000 bamboo per
hour at the cost of ~1.5 MSPT on an antique AMD FX-6300, and 1.75 when
converting the bamboo to planks (31K planks/h).

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


## Optimal rates

Bamboo grows on average once every 4096 game ticks, so the maximum achievable
rate is: 72000 / 4096 = 17.58 bamboo/hour/plant

One major caveat of two-way flying machines is that the plants at the ends of
the farm are not harvested at even intervals: simplifying to the extreme, the
first plant is harvested immediately, then after a full trip back and forth,
then immediately again, etc.

With this in mind, a reasonable choice is to try to harvest the center of
the farm every 4096 game ticks. Tests showed that by doing this, we get 15.58
items/h/plant, which is about 11.3% less than the max rate. This difference is
caused by the fact that plants are not allowed to grow taller than two blocks
and because of the uneven harvesting intervals: Higher rates are possible by
shortening the delays at the cost of higher MSPT; or make the farm bigger with
an additional layer, at a lesser performance cost.

TODO: explain how the clocks work, vertical signal propagation.


### Getting the timings right

- let `l` be the length of the planted area
- let `t` be the travel time for a flying machine from its rest position to its
  rest position on the opposite side. This is equal to `(l+24 blocks)×10
  ticks/block` (the flying machines travel at speed of 0.1 block/tick).
- let `dw` be the waiting time for a flying machine before it is sent back
- let `dh` be the delay between two flying machines
- let `d0 = 263` be the time between the delay clock ticking and the actual
  departure time of the top harvester
- let `n` be the number of layers
- let `i` be the harvesting interval for the midpoint of the farm.

Although harvesting intervals at the ends of the farm are uneven, the plant
located at the middle point is harvested at even intervals. Using this point as
refernce, this gives: 

    i = t/2 + dw + d0 + t/2
    ⇔ i = t + dw + d0

Harvesting the middle point every 4096 game ticks:

    dw = 3593 - 10×l

The number of items in the delay circuit is `⌈(dw-2)/16⌉` (round up). For a 7
chunks long farm, 112 blocks, we get 154.4, rounded up gives 155, that's 2
stacks and 27 items. Rounding up allows us to simplify the maths when
calculating the other delays without any side effects. So round up!

`dh` should be as long as possible in order to minimize the number of harvesters
moving simultaneously. If it's too long however, the topmost harvester would
come back before the bottom one has left and take the next set of minecarts (and
the bottom one would never leave) or one of the harvesters would collide whith
minecarts dropping for (or from) the other. So `dh` should be short enough for
the bottommost harvester to have left its station and cleared the minecart drop
area before the top one comes back.

Departure time of the bottom harvester + 11 blocks of clearance (110 ticks travel
time). Note that `d0` for the bottom layer is d0 + 4(n-1):

    t0 = dh(n-1) + d0 + 4(n-1) + 110
    ⇔ t0 = (dh+4)(n-1) + 373

Time for the top harvester to do a full two way trip, minus 11 blocks:

    t1 = d0 + 2×t + dw + d0 - 110
    ⇔ t1 = 10×l + 4489

Solve `t0 ≤ t1` for `dh`:

    (dh+4)(n-1) + 373 ≤ 10×l + 4489
    ⇔ dh ≤ (10×l + 4116) / (n - 1) - 4

The number of items in the delay hopper clock is `⌊(dh+12)/16⌋` (round down this
time).

For higher rates without making the farm larger, let's see the timings so that
the far ends of the farm are harvested at most every 4096 game ticks:

    i = 10×(l+24) + dw + d0 + 10×l
    ⇔ dw = 3593 - 20×l

    t1 = d0 + 2×t + dw + d0 - 110 = 4489

    t0 ≤ t1 ⇔ (dh+4)(n-1) + 373 ≤ 4489
            ⇔ dh ≤ 4116 / (n-1) - 4

Interestingly, `dh` only depends on the number of layers in this case. With
these settings, the 7×7×12 farm used as a testbed runs at 2 MSPT (instead of
1.5), with rates around 152.5K items/h (vs. 146K, +4.4%), or 16.28 items/h/plant
(7.4% below the max rate).

TODO: test with an 8th layer.

### Farm size

The minimum value for `dh` is 228 game ticks (15 items in the hopper clock).
This is the time needed for the minecart dispenser to do a full run and reset.
So the maximum number of layers is

    (10×l + 4116)/(n-1) - 4 ≥ 228
    ⇔ n ≤ (10×l + 4348)/232
    
For a 7 chunks long farm, that's 23 layers (~479K bamboo/hour).

This can be increased further by making the farm longer. As for the maximum farm
length, `dw` must be greater than 0. This gives:

    0 < 3593 - 10×l
    ⇔ l ≤ 359

Just a bit over 22 chunks. A 22 chunks long field would make the total farm
length a nice 24 chunks and bring the maximum number of layers to 33, with an
expected yield of 2.17M bamboo/hour. Make it 3 slices wide for triple rates…

TODO: lower timings or add a layer? also compare with a 10 chunks farm (12 total)


## Chunk loaders

When shutting off the farm, the chunk loaders must be kept running until the
last harvester launched comes back. The delay `dc` corresponds the the worst
case (a full 2-way trip) plus 16 game ticks to compensate for rounding (when
setting up the hopper clock for `dw`, the actual delay is `dw` ± 16):

    dc = 2(d0 + t) + dw + 16
    ⇔ dc = 10×l + 4615

For the *overclocked* version where we harvest the farm ends every 4096 game
ticks, that's just 4615, regardless of the farm size.

With large farms, the delay may exceed the capacity of a single hopper delay
circuit, so there are two such circuits wired in series for the chunk loaders.
The total number of items should be `⌈(dc + 12)/16⌉` (round up), with no less
than two items per circuit (just split the items evenly).


## Notes

Like all farms of this kind, it needs to be orientated East-West so that the
hopper-carts stay in place.

Using a single "blade" saves over 30% of the MSPT caused by the flying machine
itself (not including the hopper-carts). This was also a personnal challenge ;)

The schematic is only a showcase with a 7 chunks × 7 layers × 12 blocks version.
It can be extended up/down and in length as needed (also add chunk loaders!),
and a triple width version should fit in the 3 chunks covered by the chunk
loaders (make sure to include the minecart collection and dispenser in these).

I'd like to be able to fit a quadruple version, but I'd need to redesign part
of the minecart collection and dispenser to make it fit in the same chunk as
the flying machine holding stations.


## References

TODO: link to Ilmango's sugar cane farms, and the thing to place blocks.
