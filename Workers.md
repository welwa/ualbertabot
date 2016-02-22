# Worker Jobs #

Workers in UAlbertaBot can have one of the following job assignments:

```
enum WorkerJob {Minerals, Gas, Build, Combat, Idle, Repair, Move, Scout, Default};
```

  * Minerals - Mine minerals from a given mineral patch
  * Gas - Gather gas from a given refinery
  * Build - Build a building
  * Combat - Help defend main base location
  * Idle - No current assignment
  * Repair - Repair a building
  * Move - Move to a building's construction location
  * Scout - Scouting the map
  * Default - Default constructor value

All worker jobs are stored in a `std::map` inside `WorkerData`. You can get the status of a worker by calling `WorkerData`'s `getWorkerJob()` or `setWorkerJob()` methods.

# Worker Assignment #

On each frame, `WorkerManager.update()` performs the following logic:

  * `WorkerManager.update()`
    * Re-allocate workers to new mineral patches if mined out
    * Send idle workers to gather minerals
    * Allocate gas workers until 3 in each refinery
    * Move build workers to building location if necessary

When a worker is trained, its status is initially set to `Default`. When the `update()` function is called and detects that the unit it Idle, it sets the unit to gather minerals at the closest known resource depot, up to a cap of 3 workers per mineral patch per resource depot. Similarly, when a refinery is constructed, it sends 3 mineral workers to gather gas from that refinery.

The only additional behaviour for workers is that they are able to be set to scout or combat. This is done inside `GameCommander` and `CombatCommander` respectively. These classes remove the current worker job from `WorkerManager` and control them via `ScoutManager` or `Squad` respectively.

If you wish to modify the behaviour of worker allocation, these are the locations of the code which you need to change.