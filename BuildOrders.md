# Build Order Overview #

All unit production in UAlbertaBot is handled by [ProductionManager](https://code.google.com/p/ualbertabot/source/browse/trunk/UAlbertaBot/Source/base/ProductionManager.h).

`ProductionManager.cpp`'s constructor sets an initial build order as follows:

  * initially set build order to 'opening book' build order from strategy manager
    * these build orders are represented by a series of integers corresponding to unit types, which can be found in `StarcraftData.hpp`
    * these opening book build orders will occur in entirety until an event happens which triggers a new build order to be searched
Here is the control flow for `ProductionManager.update()` which is called every frame:

  * if event occurs which triggers a new build order (worker dies, building dies, or we run out of things to build in the current build order)
    * clear current build order
    * get new build order goal from `StrategyManager`
    * pass goal into `StarcraftBuildOrderSearch`
    * set build order to the returned result
  * if current build order contains items
    * if the highest priority item can be built
      * build the highest priority item

The events under which a new build order are triggered are listed below

# Implementation #

Build orders in UAlbertaBot are handled by the [BuildOrderQueue](https://code.google.com/p/ualbertabot/source/browse/trunk/UAlbertaBot/Source/base/BuildOrderQueue.h), which stores a priority-sorted queue of `BuildOrderItem` objects.

```
class BuildOrderItem 
{
   MetaType   metaType; 
   int        priority;
   bool       blocking; 
};
```

A `BuildOrderItem` is essentially just a container which holds
  * [MetaType](http://code.google.com/p/ualbertabot/source/browse/trunk/UAlbertaBot/Source/base/MetaType.h) - A custom container for a BWAPI buildable types
  * `int` - The priority of the item in the queue
  * `bool` - Whether the object 'blocks' the next item in the build order. If an item is blocking, items with lower priority in the queue can be built until it is made. If an item is not blocking, the next item in the queue will be built if its resources are available before the that item. To ensure a build order will be executed in exact order, set all items blocking to true.

A [MetaType](http://code.google.com/p/ualbertabot/source/browse/trunk/UAlbertaBot/Source/base/MetaType.h) object is a custom class which is able to store either of the three BWAPI buildable/trainable types. It holds either a `BWAPI::UnitType`, `BWAPI::TechType` or `BWAPI::UpgradeType`. It was created so that the `BuildOrderQueue` only needs to store a single object type. To create a `MetaType` simply call its constructor with one of the 3 listed types:

```
MetaType type(BWAPI::UnitTypes::Marine);
MetaType type2(BWAPI::UpgradeTypes::Leg_Enhancements);
```

`BuildOrderItem`s are then inserted into the `BuildOrderQueue` by calling one of the following:

```
void BuildOrderQueue::queueItem(BuildOrderItem<PRIORITY_TYPE> b);
void BuildOrderQueue::queueAsHighestPriority(MetaType m, bool blocking);
void BuildOrderQueue::queueAsLowestPriority(MetaType m, bool blocking);
```

If `queueItem` is called, its priority level must be set manually by the user. It is recommended to only use the highest / lowest priority insert methods.

## Example Build Order ##

As an example, let's say we want to build the following items in order:
  1. Protoss Probe
  1. Protoss Probe
  1. Protoss Pylon
  1. Protoss Probe
  1. Protoss Gateway
  1. Protoss Zealot

You could insert each of these items into the `BuildOrderQueue` yourself from the `ProductionManager`, however a function already exists to do this job for you:

```
void ProductionManager::setBuildOrder(const std::vector<MetaType> & buildOrder);
```

This method takes an ordered vector of `MetaType` objects, clears the current `BuildOrderQueue` and inserts each type to be built in order. So  the safest and easiest way to set a custom build order is to call the following code somewhere within `ProductionManager`:

```
std::vector<MetaType> buildOrder;

buildOrder.push_back(MetaType(BWAPI::UnitTypes::Protoss_Probe));
buildOrder.push_back(MetaType(BWAPI::UnitTypes::Protoss_Probe));
buildOrder.push_back(MetaType(BWAPI::UnitTypes::Protoss_Pylon));
buildOrder.push_back(MetaType(BWAPI::UnitTypes::Protoss_Probe));
buildOrder.push_back(MetaType(BWAPI::UnitTypes::Protoss_Gateway));
buildOrder.push_back(MetaType(BWAPI::UnitTypes::Protoss_Zealot));

setBuildOrder(buildOrder);
```

## Build Order Goals & Planning ##

UAlbertaBot uses a heuristic search algorithm to determine build orders, which is located in the `StarcraftBuildOrderSearch` project. The details of this project are out of the scope of this wiki. However, all interaction to the algorithms in this project are handled in UAlbertaBot via the [StarcraftBuildOrderSearchManager](http://code.google.com/p/ualbertabot/source/browse/trunk/UAlbertaBot/Source/base/StarcraftBuildOrderSearchManager.h) class, which is called from `ProductionManager.performBuildOrderSearch()`. For now, let's treat that system as a black box which when given a build order goal creates an ordered vector of `MetaType` objects similar to the example above.

`ProductionManager` will trigger a new build order to be planned based on one of the following events occurring:

  * The first build order of the game is obtained via an 'opening book' of build orders in the constructor of `ProductionManager`

The remaining events are detected in `ProductionManager.update()` which is called each frame:

  * If the current `BuildOrderQueue` is empty
  * If a supply deadlock is detected (we have too little supply to build a unit)
  * If a critical unit (building or worker) has died
  * If an enemy cloaked unit is detected

For each of these events, a new build order goal is obtained via
```
const MetaPairVector StrategyManager::getBuildOrderGoal()
```

If you wish to alter the build order goal strategy of UAlbertaBot, you simply need to modify that function. However if you want to alter UAlbertaBot to not use its automatic planning system, this must be done by rewriting your own planning system inside

```
void ProductionManager::testBuildOrderSearch(const std::vector< std::pair<MetaType, UnitCountType> > & goal)
```


## Overriding Build Order Search ##

As mentioned above, by default UAlbertaBot uses a heuristic search algorithm to determine build orders from a goal vector of units. This system can be overridden (useful for Zerg since it doesn't work for them very well) by constructing your own build orders. Simply remove any code inside `ProductionManager` which calls the function `performBuildOrderSearch()` which sets the build order based on the search algorithm and replace this with your own custom build order code. You can then set the build order in `ProductionManager` using the method described above in the "Example Build Order" section.