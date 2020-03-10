# Functional Factorio

*Goal*: Implement Lean Manufacturing in Factorio

The goal of this project is to use the ideas of recursion and functional programming to create a just-in-time/lean manufacturing system in vanilla Factorio.  This will allow me to do cool things like take a 5 rocket/hour factory and tell it to fire one rocket/hour and get that exactly.  It'll also allow me to do that without mining any more ore than is absolutely necessary. This is also a design that can be easily incorporated into existing bases.  This means I need to meet the following design goals:

## Design Goals
* Pull-system: Only start new work if there is a specific request for it.
* Avoid waste:
  * unnecessary transportation
  * excess inventory/overproduction
  * waiting
  * dead-ends
  * buffers
* Modularization: a given manufacturing line should be seen as a module with inputs and outputs (in computer science terms, as a function)
* What this design is NOT:
  * A transportation system: the design doesn't care if you use belts, robots, trains, or a mix of all three.
  * An attempt to be space efficient or time efficient
  * perfect

## Design
### Demand/Kanban Network
The factory stations will fulfill orders that arrive as pulses on a demand network (red in my implementation).  These orders for ingredients propagate recursively from station to station until all ingredients needed to fulfill the original order are pulsed.  It can be useful for debugging to put an adder on the network to keep track of what was pulsed.

I haven't tried it, but it may be useful in larger builds to maintain different demand networks and even have a station pulse to a different network than it receives from.

### The Information Network
This is where stations register the products they make.  It's on the green wire in my implementation.  These are not pulses, they're constants added together via the Circuit Network's free summing.  It's also used for reseting all of the memory cells (2 per station) for every station in the information network.  This is done by pulsing "R=1".  This is very useful for debugging.  Any other virtual signals can safely be used for any other purpose on this network.

### Stations
A station has two parts: a station control unit and a sections of factory that input zero or more input ingredients and output a product.  Generally, you want to design stations to minimize transportation between stations.  This means, for example, that if your product takes copper cable, you should assemble it on site and list copper plate as your ingredient rather than transporting twice as many items.

You may want to have each station surrounded by walls and turrets both for defense and ease of telling one from the other.


**Station Control Unit**

A station has a left terminal and a right terminal: large electric poles.  The left terminal contains the station's internal networks, both red and green.  The right terminal is the station's link to the world:  both red and green networks.  The left terminal should NEVER be connected to an external circuit network.

The SCU has two primary purposes:  
1. Calculate the ingredients needed to fulfill an incoming order and pulse them to the demand network (right terminal).
1. Keep track of how many ingredients should be requested and have been delivered (left terminal).

It receives orders from other stations via pulses on the red demand network (right terminal).  It then calculates the ingredients needed using a pre-programmed recipe constant combinator.  Those ingredients are then divided by the number of supplier stations for each ingredient and pulsed back out to the demand network.  The ingredients the station needs are also stored in a decider combinator.  As ingredients enter the station, they are counted, that count is multiplied by -1 and stored.  This allows the station to calculate the number of each ingredient we still need.  For example, this can be used to set the request on a Requester Chest.  The red wire is then connected to the inserter removing items from that chest to pulse each item that's removed.  This can also be adapted easily to conveyer belts or trains without changing anything in the SCU.

The station also registers itself on the information network (right terminal green) via a constant combinator with its product as the signal.

One of the great things about Functional Factorio is that we don't care what the throughput of a given station is, so no more calculating exact ratios.  Though you should be aware that your production of a given product is limited by its slowest station.  For example, if you have a 4 GC/second station and a 1 GC/second station, they will each receive half of the total order, and we will need to wait for the slow station to finish either way to fulfill the order.  It's actually faster in this situation to just remove the slower station.

The station should always manufacture everything the ingredients allow and send the products on down the line, whether by logistics network, train, or conveyer.

It should be noted that the SCU's memory cells will integer overflow if you order enough products.  They can be reset by pulsing "R=1" on the information network.  Just keep in mind that this also voids out a station's internal knowledge of how many ingredients have been overdelivered or underdelivered.

![2 ingredient station screenshot][2-screenshot]
*Screenshot of a 2 ingredient station*

![2 ingredient station wire diagram][2-wire]
*Wire diagram of a 2 ingredient station control unit*

To program the SCU, put the product in the left constant combinator, the recipe in the right constant combinator, and modify the other combinators accordingly.  You will need to add three more arithmetic combinators per ingredient wired in parallel with the two signal isolators and per-supplier orderer.  The blueprint book in this repository has examples of 2 and 3 ingredient SCU's.

**Station Floor**

The station floor contains all of the assemblers, conveyers, and everything else necessary to make the product.  Your goal is designing the station floor should be to do everything possible to reduce waste.  By waste here I mean any ingredient that doesn't eventually leave the station as a product.  This includes ingredients left in chests, on conveyers, and in assemblers' internal buffers. In practice, due to these buffers, in a large set up you may have enough ingredients lost to waste that you can't fulfill your order completely.  In these cases, it may be a good idea to introduce a "fudge factor" combinator that adds a constant to each demand pulse signal.

If you use productivity modules, you'll have overproduction unless you account for it in your recipe, which is difficult without floating point numbers.  I would just avoid them.

In my prototype, I used barrels to transport fluids into and out of stations.  Since you can't get pulse reads on pipes, it'd be tough to get accurate information without some tanks and magical math.  I'll leave that as an exercise for the reader :-)

### Central controls
It may be useful to have a "central control center" with all of the knobs and dials you use to run your factory.  Every control center should at least have a reset pulser (green network) and an order pulser (red network).  Most others will also want a clock pulser so that you can trigger a rocket launch every hour, for example.

### Limitations
Oil Refining: The SCU will not work as designed with any station that produces more than one type of product.  This situation can come up during oil refining.  The best way around it currently is to design your refinery so that each station only produces a single output (ie. petroleum gas or plastic).

**Github Repo**: https://github.com/garrettngarcia/functional-factorio

**Factorio Prints Page**: https://factorioprints.com/view/-M21CcICFTPPvl1bHIRa

[2-screenshot]: https://github.com/garrettngarcia/functional-factorio/raw/master/2_ingredient_station.png "2 ingredient station screenshot"
[2-wire]: https://github.com/garrettngarcia/functional-factorio/raw/master/2_ingredient_station_design.png "2 ingredient station wire diagram"
