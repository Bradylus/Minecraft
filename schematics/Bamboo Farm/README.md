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

TODO

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
