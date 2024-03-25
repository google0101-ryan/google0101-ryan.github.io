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

# `LG_LevelBuilder` handoff

The bulk of actual generation happens when `Builder` calls `LG_LevelBuilder.BuildFloor()`. The first thing this function does is call `SetupStartRotation()`, which picks a random number between one and four using the `BuildSeed`. It maps like this:  
| Number| Direction	|
|---	|---------	|
| 0 	| Forward 	|
| 1 	| Back    	|
| 2 	| Right   	|
| 3 	| Left    	|  

EDIT: However, it seems that the `switch` is hardcoded to 0, which means it always defaults to Forward.

It also builds a rotation lookup table, using the following block of code:

```c#
protected void BuildRotationLookup()
{
	RotationLookup = new Quaternion[5];
	RotationLookup[0] = Quaternion.AngleAxis(-90f, Vector3.up);
	RotationLookup[1] = Quaternion.AngleAxis(0f, Vector3.up);
	RotationLookup[2] = Quaternion.AngleAxis(90f, Vector3.up);
	RotationLookup[3] = Quaternion.AngleAxis(180f, Vector3.up);
	RotationLookup[4] = Quaternion.AngleAxis(0f, Vector3.up);
}
```

Now, it calls `static LG_Floor.Create()`, passing (0, 0, 0) as an argument representing the floor's origin in the world.

# `LG_Floor.Create(Vector3 pos)`

This function begins by instantiating an empty GameObject called "StaticGameObject" and sets its name to either 1st floor, 2nd floor, 3rd floor, or `(i - 1) + "th floor"`. It then attaches an LG_Floor component to the object, and sets `Builder.Current.m_currentFloor` to the newly created object. The function then returns the floor to the calling function.

When `Create()` returns, `Builder.ComplexResourceBlock` gets set to a code representation of the JSON `ComplexResource` datablock, which describes all plug, door, ladder, elevator, and room prefabs. We then inject two factory jobs, which, for the sake of this, we'll assume runs immediately. The first is `LG_LoadComplexDataSetResourcesJob`, which organizes all the complex resources into lists of "shards", i.e. by type. The next job is much more interesting, and is called `LG_SetupFloor`. Let's go there now.

# `LG_SetupFloor.Build()`
Once we get here, things really get interesting. The first things this function does is set the current subcomplex, then use that to select an elevator tile from the previously mentioned asset shards. The `ComplexResourceSetDatablock` uses `BuildSeedRandom` to pick a number in the range 0 and the number of elevators, and then returns the prefab at that index in the datablock. It then spawns this tile. However, as it has not setup any of the prefabs or markers, there is still nothing there. The game then gets the `LG_FloorTransition` component off of the elevator. Now it calls `SetupAreas` of the new floor transition. This runs through two lists: `LG_RandomAreaSelector` and `LG_Area`. Let's go through them one by one.

# `LG_RandomAreaSelector`
For each instance of this script, there is a list of areas attached. Using `BuildSeedRandom`, only one of these areas with a matching index will be enabled, and the rest will be disabled.

# `LG_Area`
This one is a bit more complicated. Each zone has a list of potential areas attached to it. After these have been enabled or disabled based on `LG_RandomAreaSelector`, we are left with only a handful of areas. For each area, it calls `Setup()`.

# `LG_Area.Setup()`

