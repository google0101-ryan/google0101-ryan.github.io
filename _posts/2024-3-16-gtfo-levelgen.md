This guide assumes you have at least some experience with Unity and GTFO level building.

Some terminology before we begin:

<b>AREA:</b> An individual section of an zone (read: room) with its own letter, A-Z.  
<b>GATE:</b> Connects two areas. Usually some kind of small rolling door.  
<b>GEOMORPH:</b> A prefab that contains a large chunk of level, usually with multiple areas contained within  
<b>PLUG:</b> Used to transition between geomorphs. Usually a door, but can be archways or holes in walls. Gates are a type of plug
<b>TILE:</b> Tiles are used to represent the placement of geomorphs on a grid. For example, a 64x64 geomorph on a grid with a tile size of 64 (most levels) takes up 1 tile  
<b>SECURITY GATE:</b> A special type of gate separating zones, usually requiring a bioscan to enter  
<b>ZONE:</b> One or more areas, separated by security gates  
<b>LAYER:</b> One of primary, secondary, or overload objectives. These are treated as almost entirely seperate levels, connected by bulkheads  
<b>BULKHEAD</b> A door requiring a bulkhead key to enter, used to seperate layers

# Job Batching

So, the first thing that needs discussing is how GTFO batches level generation steps. This happens in the `LG_Factory` class. Essentially, this class queues "Jobs" (which inherit from `LG_FactoryJob`) in batches, and then runs a job each tick when the `Update()` function is called. This is used to efficiently run jobs while also performing other tasks. There is also a semi-complicated dependency system that is mostly unused.

# Globals
In Unity, setting up globals is surprisingly difficult. So, 10CC devised a somewhat ingenious system. Each global inherits from the `GlobalManager` class, which in turn inherits from `MonoBehaviour`. Each of these then registers themselves with the `Globals.Global` class, which will call various functions as necessary.

# In The Beginning
Level setup begins in `LevelGeneration.Builder`, which is a global. The first thing it does when it wakes up is set `static LevelGeneration.Builder.Current` to `this`, so that scripts can easily reference the current `Builder`. It will then get the `LG_LevelBuilder` script instance from the `GameObject` that `Builder` is attached to. It also creates several unique random sources, hereafter referred to as `BuildSeed`, `HostIDSeed`, and `SessionSeed`. At some point, `Builder.Build()` is called. This kicks off level generation.

# Level generation actually starts
At this point, the player has started descending the elevator shaft. While this glorified loading screen is playing, `Builder.Build()` is called. It does a lot, so let's break it down into easy chunks.

1. Seed Generation  
The seeds for the expedition are fetched from the expedition datablock. This includes `BuildSeed`, `FunctionMarkerSeedOffset`, and `StandardMarkerSeedOffset`, and `LightSeedOffset`. It also gets the `HostIDSeed` and `SessionSeed` here.  
2. Create and initialize an instance of `LG_Factory`  
All jobs will live inside of `Builder`.  
3. Create the main layer.  
Here, the game fetches the level layout datablock and creates the main layer, hardcoded to build from zone 0. It then adds this layer to `LayerBuildDatas`, which is a `List` of `LG_LayerData`s.
4. Create secondary and/or overload layers.  
If other layers exist in the datablock, they get added here as well. These build off of the zone and layer specified in the rundown datablock.  
5. Setup `SerialGenerator`  
This class is used for everything from item names to the IP address used for terminal uplink objectives. This class's `Setup()` function simply initializes a premade array of serial numbers, code word prefixes, and IP addresses for fast lookup by the game. The game defaults to maximum of 899 serial numbers, 733 code words (these are used for things like reactor startup and terminal uplink), 27 prefixes (this is stuff like Y07, used for terminal uplink), and 99 IP addresses. It then uses `BuildSeed` to shuffle these arrays into a random order.

# `LG_LevelGeneration` handoff

The bulk of actual generation happens when `Builder` calls `LG_LevelGeneration.BuildFloor()`