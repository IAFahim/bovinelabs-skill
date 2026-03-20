> **Audience:** Developers working on Unity projects utilizing Entities (DOTS), Burst, Jobs, and the `BovineLabs.Core` framework.  
> **Philosophy:** Zero cyclomatic complexity systems, strict zero-allocation hot paths, maximum Burst compliance, and leveraging `BovineLabs` code-generation and extensions to eliminate boilerplate.

## 1. Assembly Definition Architecture

The foundation of our ECS architecture relies on strict compilation boundaries. If code can reach where it shouldn't, architecture degrades.

### 1.1 `autoReferenced: false` is Non-Negotiable
Every assembly must explicitly declare its dependencies. `autoReferenced: true` is banned.
*   **Why?** Editor-only types will leak into player builds, build times will skyrocket, and circular dependencies will form.

### 1.2 The Six-Layer Assembly Pattern
Use the `BovineLabs Assembly Builder` (`BovineLabs -> Tools -> Assembly Builder`) to generate compliant layers.

| Assembly            | Purpose                                      | Constraint / Attributes                             | Key Rule                               |
|---------------------|----------------------------------------------|-----------------------------------------------------|----------------------------------------|
| `Feature.Data`      | `IComponentData`, `IBufferElementData`, Tags | None (Ships everywhere)                             | **No logic. Pure data structs only.**  |
| `Feature`           | `ISystem`, Jobs, ViewModels                  | None                                                | No MonoBehaviours. No Editor types.    |
| `Feature.Authoring` | Bakers, MonoBehaviours                       | `UNITY_EDITOR` <br> `[DisableAutoTypeRegistration]` | **Never** compile into player builds.  |
| `Feature.Debug`     | Debug panels, diagnostic tools               | `UNITY_EDITOR \|\| BL_DEBUG`                        | Ships only in dev builds.              |
| `Feature.Editor`    | Custom inspectors (`ElementEditor`)          | `includePlatforms: [Editor]`                        | Tools for designers.                   |
| `Feature.Tests`     | Unit & Integration tests                     | `TestAssemblies` <br> `[DisableAutoCreation]`       | Test systems never run in live worlds. |

### 1.3 `InternalsVisibleTo` Routing
We use `internal` heavily to hide system implementations and raw data from other domains.
```csharp
// Feature.Data/AssemblyInfo.cs
[assembly: InternalsVisibleTo("Feature")]
[assembly: InternalsVisibleTo("Feature.Authoring")]
[assembly: InternalsVisibleTo("Feature.Tests")]
```
> **Rule:** `Feature.Data` exposes internals to its consumers, but *never* receives `InternalsVisibleTo` back from runtime. No circular trust.


## 2. World & Bootstrap Architecture

Standard Unity DOTS pushes everything into the `DefaultWorld`. We use `BovineLabsBootstrap` to cleanly separate logic into distinct worlds based on their lifecycle and network role.

### 2.1 The `BovineLabsBootstrap`
Do not use standard Unity bootstraps. Inherit from `BovineLabsBootstrap` to automatically generate our core worlds:
*   **GameWorld:** Main simulation world.
*   **ServiceWorld:** Runs background services, UI synchronization, and systems that persist across scene loads.
*   **MenuWorld:** Exists for main menus without spinning up heavy physics/gameplay systems.

```csharp
// ✅ GOOD - Custom bootstrap initializing the BovineLabs ecosystem
[Configurable]
public class GameBootstrap : BovineLabsBootstrap
{
    protected override void Initialize()
    {
        // 1. Automatically creates ServiceWorld
        base.Initialize();
        
        // 2. Initialize our specialized worlds
        CreateMenuWorld();
        
        // Use CreateGameWorld() or CreateClientServerWorlds(isLocal: true) for NetCode
    }
}
```

### 2.2 System World Filtering
Always tag systems with their target world using `[WorldSystemFilter]`. If you don't, Unity puts it everywhere.

```csharp
using BovineLabs.Core; // Access to predefined Worlds flags

// ✅ System only runs in the simulation world (Client/Server/Local)
[WorldSystemFilter(Worlds.Simulation)]
public partial struct MovementSystem : ISystem { }

// ✅ System only runs in the persistent Service world[WorldSystemFilter(Worlds.Service)]
public partial struct UserProfileServiceSystem : ISystem { }

// ✅ System runs in Simulation AND Editor (for edit-time baking/rendering)
[WorldSystemFilter(Worlds.SimulationEditor)]
public partial struct PhysicsStateSystem : ISystem { }
```


## 3. Core Utilities: Logging, Math, and Assertions

We do not use standard `UnityEngine.Debug` or `UnityEngine.Assertions` in hot paths or ECS code. The `BovineLabs.Core` utilities are Burst-friendly, SIMD-optimized, and context-aware.

### 3.1 Assertions: `Check.Assume`
Standard asserts cause branching which degrades Burst's auto-vectorization. We use `BovineLabs.Core.Assertions.Check`.

```csharp
// ❌ BAD: Debug.Assert creates branches and compiles out poorly
Debug.Assert(health > 0, "Health must be positive");

// ✅ GOOD: Check.Assume tells Burst compiler to optimize under this assumption
// Only evaluates if ENABLE_UNITY_COLLECTIONS_CHECKS or UNITY_DOTS_DEBUG is defined
Check.Assume(health > 0, "Health must be positive");
Check.Assume(index < buffer.Length);
```

### 3.2 Global & World-Aware Logging
Never use `UnityEngine.Debug.Log` in ECS. Use `BLGlobalLogger` for static contexts and the `BLLogger` singleton for world/frame-aware logging.

Log levels are controlled dynamically via the `debug.loglevel` ConfigVar.

```csharp
// 1. Static Contexts (e.g., Bootstrap, Static methods)
BLGlobalLogger.LogInfo("Application Starting...");
BLGlobalLogger.LogWarning("Network disconnected.");

// 2. In Systems / Jobs (World and Frame aware)
[BurstCompile]
public partial struct DamageSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Get the logger singleton. It automatically prefixes the world name and frame count!
        var logger = SystemAPI.GetSingleton<BLLogger>();
        
        // FixedString allocation is Burst compatible
        logger.LogDebug($"Processing {count} damage events");
        logger.LogError512("Critical damage overflow!");
    }
}
```
*Output Format:* `[Level] | [Frame] | [World] | [Message]` -> `D | 1204 | Game | Processing 5 events`

### 3.3 SIMD Math: `mathex`
When processing arrays, always use `BovineLabs.Core.Utility.mathex`. It leverages `float4` and `int4` block processing for massive speedups.

```csharp
// ❌ BAD: Iterating an array manually
float max = float.MinValue;
for(int i = 0; i < array.Length; i++) max = math.max(max, array[i]);

// ✅ GOOD: Use mathex for SIMD chunking
float max = mathex.max(array);
float sum = mathex.sum(array);
mathex.add(outputArray, inputArray, 5); // SIMD vector addition
```

### 3.4 Thread-Safe Random: `GlobalRandom`
`Unity.Mathematics.Random` cannot be shared across threads. Creating new seeds per entity is slow. We use `GlobalRandom` and `ThreadRandom`, which maintain pre-allocated states per worker thread.

```csharp
// ✅ GOOD: Safe to use in IJobEntity, IJobChunk, IJobFor, etc.
[BurstCompile]
partial struct RandomSpawnJob : IJobEntity
{
    private void Execute(ref LocalTransform transform)
    {
        // Automatically fetches the Random instance for the current executing worker thread.
        // Zero false-sharing, zero locks, maximum performance.
        transform.Position = GlobalRandom.NextFloat3Direction() * 10f;
        transform.Rotation = GlobalRandom.NextQuaternionRotation();
    }
}
```


## 4. Naming Conventions & Layout

### 4.1 Namespaces
*   **Format:** `[Company].[Project].[Feature]` (e.g., `BovineLabs.Combat`)
*   **Authoring:** `[Company].[Project].[Feature].Authoring`

### 4.2 Component Naming
Components describe WHAT an entity IS or HAS, not what it DOES.
*   **Tags:** Adjectives or States (e.g., `Dead`, `Stunned`, `Invulnerable`). *Never* suffix with "Tag".
*   **Data:** Nouns (e.g., `Health`, `Velocity`). *Never* suffix with "Component" or "Data".
*   **Buffers:** Singular Noun (e.g., `InventoryItem`, `Waypoint`).

### 4.3 System & Job Naming
*   **Systems:** `[Verb][Subject]System` (e.g., `ApplyDamageSystem`, `ProcessInputSystem`).
*   **Jobs:** `[Verb][Subject]Job`.
    *   If using `IJobChunk`, suffix with `ChunkJob`.
    *   If using `IJobEntity`, just suffix `Job`.

### 4.4 File Layout Standard
Group by feature, not by ECS type.
```text
Feature.Data/
  Components/
    Health.cs            <-- Pure data, NO LOGIC
  Facets/
    HealthFacet.cs       <-- Replaces Aspects
  Buffers/
    StatusEffect.cs

Feature/
  Systems/
    ApplyDamageSystem.cs <-- ISystem, [BurstCompile]
  Jobs/
    ProcessDamageJob.cs

Feature.Authoring/
  HealthAuthoring.cs     <-- Contains MonoBehaviour AND Baker class
```


## 5. Standard Component Design Rules

**RULE: Components are pure memory layouts.**
```csharp
// ❌ BAD: Contains methods, properties, or logic.
public struct Health : IComponentData
{
    public float Current;
    public float Max;
    public bool IsDead => Current <= 0; // VIOLATION
    public void TakeDamage(float amt) { ... } // VIOLATION
}

// ✅ GOOD: Blittable, pure memory.
public struct Health : IComponentData
{
    public float Current;
    public float Max;
}
```

**RULE: Hide internal buffers behind Facets/Commands.**
Never directly modify complex `DynamicBuffer` structures from random systems.


## 1. The `IEnableableComponent` vs. Structural Changes

Structural changes (adding/removing components or destroying entities) stall the main thread, invalidate caches, and prevent parallel processing.

### 1.1 Structural Change Rules
*   **RULE:** Never use `EntityCommandBuffer.AddComponent<T>` or `RemoveComponent<T>` for state toggles that happen frequently (e.g., `IsStunned`, `IsSelected`).
*   **RULE:** Always use `IEnableableComponent` for transient states.

```csharp
// ❌ BAD: Causes a chunk move and memory allocation every time a unit gets stunned
public struct Stunned : IComponentData { } 
// Usage: ecb.AddComponent<Stunned>(entity);

// ✅ GOOD: Zero allocation, zero chunk moves. Memory is pre-allocated.
public struct Stunned : IComponentData, IEnableableComponent { }
// Usage: SystemAPI.SetComponentEnabled<Stunned>(entity, true);
```

### 1.2 Profiling Change Filters
If a system uses `[WithChangeFilter(typeof(MyComponent))]`, but that component is written to every frame, the filter is useless.
*   **RULE:** Add `[ChangeFilterTracking]` to highly-queried components.
*   Use the `BovineLabs -> Tools -> Change Filter` window to ensure your component isn't thrashing (changing >85% of frames).


## 2. The `KSettings` System (Burst-Safe String IDs)

Enums are rigid and require recompilation to change. Hardcoding integer IDs is prone to conflicts. Comparing strings inside Burst jobs is heavily restricted and slow.

**The Solution:** We use `KSettings<T, TV>`. It allows designers to define human-readable strings in a ScriptableObject, which are deterministically converted to unmanaged values (`byte`, `int`, `short`) that are 100% Burst-compatible.

### 2.1 Defining a K-Setting
Create a class inheriting from `KSettingsBase<T, TV>` or `KSettings<T, TV>`.

```csharp
using BovineLabs.Core.Keys;

// ✅ Defines a mapping of string -> byte for Character States
public class CharacterStates : KSettings<CharacterStates, byte>
{
    // Optionally provide hardcoded defaults, designers can add more in the Editor
    protected override IEnumerable<NameValue<byte>> SetReset()
    {
        yield return new NameValue<byte>("idle", 0);
        yield return new NameValue<byte>("running", 1);
        yield return new NameValue<byte>("attacking", 2);
    }
}
```

### 2.2 Authoring with `[K]`
In your Authoring MonoBehaviours, use the `[K]` attribute. The inspector will automatically render a dropdown of the available string names, but serialize the underlying unmanaged value.

```csharp
public class SpawnAuthoring : MonoBehaviour
{
    // The inspector shows "idle", "running", but stores the byte value
    [K(nameof(CharacterStates))] 
    public byte InitialState;

    // Use flags: true for bitmasks!
    [K(nameof(PhysicsLayers), flags: true)] 
    public int CollisionMask;
}
```

### 2.3 Resolving inside Burst
Inside a `[BurstCompile]` job, you can safely look up the key by its string name without allocating memory.

```csharp
[BurstCompile]
public void Execute(ref Character character)
{
    // 100% Burst compatible. Converts "attacking" to its byte ID at runtime.
    if (character.State == CharacterStates.NameToKey("attacking"))
    {
        // Do attack logic
    }
}
```


## 3. Global vs. World-Specific Settings

We heavily rely on `ScriptableObjects` for design data, but we must cleanly bridge them into the ECS world.

### 3.1 `SettingsSingleton<T>` (Global / Pre-World Data)
Use `SettingsSingleton<T>` for data that must exist *before* ECS worlds spin up (e.g., Boot configurations, UI themes, Master Audio volumes).
*   **RULE:** They are automatically collected and included in the build via `CoreBuildSetup`. Do not put them in a `Resources` folder.
*   **Access:** `MyGlobalSettings.I` (Available instantly, everywhere).

### 3.2 `SettingsBase` (World-Baked Data)
For ECS gameplay data, inherit from `SettingsBase`.
*   Use `[SettingsGroup("Category")]` for the Editor Window.
*   Use `[SettingsWorld("Client", "Server")]` to target specific worlds (crucial for Netcode).

```csharp
[SettingsGroup("Combat")][SettingsWorld("Server", "Local")] // Strip from ThinClients
public class CombatSettings : SettingsBase
{
    public float GlobalDamageMultiplier = 1.2f;

    public override void Bake(Baker<SettingsAuthoring> baker)
    {
        var entity = baker.GetEntity(TransformUsageFlags.None);
        baker.AddComponent(entity, new CombatConfig { Multiplier = GlobalDamageMultiplier });
    }
}
```
> *Workflow:* Add a `SettingsAuthoring` component to a GameObject in a SubScene. Opening the `BovineLabs -> Settings` window automatically assigns all `SettingsBase` assets to the correct Authoring components based on their `[SettingsWorld]` attributes.


## 4. `[Singleton]` Buffers (The Many-to-One Pattern)

When making modular games, multiple subscenes or mods might contribute to a single list of data (e.g., a master list of all craftable items). We use `BovineLabs.Core.Settings.SingletonAttribute` to automatically merge them.

### 4.1 Defining and Baking
**CRITICAL:** Ensure you use `BovineLabs.Core.Settings.SingletonAttribute`, *not* the Facet `[Singleton]` attribute.

```csharp
using BovineLabs.Core.Settings;

// 1. Mark the buffer with [Singleton]
[Singleton]
public struct CraftableItemElement : IBufferElementData
{
    public int ItemId;
}

// 2. Multiple different bakers can add this buffer to their own entities
public class ModABaker : Baker<ModAAuthoring>
{
    public override void Bake(ModAAuthoring auth)
    {
        var buffer = AddBuffer<CraftableItemElement>(GetEntity(TransformUsageFlags.None));
        buffer.Add(new CraftableItemElement { ItemId = 100 });
    }
}
```

### 4.2 Runtime Merging & The Initialization Signal
At runtime, `SingletonSystem` automatically finds *all* entities with `CraftableItemElement`, copies their contents into **one master singleton entity**, and removes the buffer from the source entities.

It also enables a one-frame `SingletonInitialize` component.

```csharp
// ✅ GOOD: Rebuild your internal caches only when the singleton buffer changes
[UpdateInGroup(typeof(SingletonInitializeSystemGroup))]
public partial struct BuildItemCacheSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // This system ONLY runs on frames where a [Singleton] buffer was merged/updated
        var masterBuffer = SystemAPI.GetSingletonBuffer<CraftableItemElement>(true);
        // ... rebuild hash maps ...
    }
}
```


## 5. Object Management: `IUID` and `ObjectDefinition`

Traditional ECS requires passing `Entity` prefab references. This breaks down when:
1. You need to reference a prefab across a network (Entities have different IDs on Client vs Server).
2. You need to save/load a reference to disk.
3. You need a designer to pick a prefab in an Inspector for a component that hasn't been baked yet.

**The Solution:** We use `ObjectId` (a deterministic `int`) backed by `ObjectDefinition` assets.

### 5.1 Creating Object Definitions
1.  Implement `IUID` on a ScriptableObject, or use the built-in `ObjectDefinition`.
2.  Use the `[AutoRef]` attribute so it automatically registers itself to a Manager asset.
3.  The ID is automatically generated, branch-safe, and immutable once created.

```csharp
public struct SpawnRequest : IComponentData
{
    public ObjectId PrefabId; // ✅ Network/Save safe. Deterministic integer.
    public float3 Position;
}
```

### 5.2 Authoring with `ObjectDefinition`

```csharp
public class SpawnerAuthoring : MonoBehaviour
{
    // The inspector provides a custom search window (supports "ca=" category filters)[SearchContext("ca=enemy", ObjectDefinition.SearchProviderType)]
    public ObjectDefinition EnemyToSpawn;

    public class Baker : Baker<SpawnerAuthoring>
    {
        public override void Bake(SpawnerAuthoring auth)
        {
            AddComponent(GetEntity(TransformUsageFlags.None), new SpawnRequest
            {
                // Implicit cast from ObjectDefinition -> ObjectId
                PrefabId = auth.EnemyToSpawn, 
                Position = auth.transform.position
            });
        }
    }
}
```

### 5.3 Instantiating at Runtime (`ObjectDefinitionRegistry`)
To spawn the actual entity, we look up the `ObjectId` in the `ObjectDefinitionRegistry`. This is an `O(1)` array lookup.

```csharp
[BurstCompile]
public partial struct SpawnSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Get the master registry mapped by the BovineLabs ObjectManagementSettings
        var registry = SystemAPI.GetSingleton<ObjectDefinitionRegistry>();
        var ecb = new EntityCommandBuffer(state.WorldUpdateAllocator);

        foreach (var (request, entity) in SystemAPI.Query<RefRO<SpawnRequest>>().WithEntityAccess())
        {
            // Resolve ObjectId -> Actual Entity Prefab
            Entity prefabToSpawn = registry[request.ValueRO.PrefabId];
            
            Entity instance = ecb.Instantiate(prefabToSpawn);
            ecb.SetComponent(instance, LocalTransform.FromPosition(request.ValueRO.Position));
            
            ecb.DestroyEntity(entity);
        }
        
        ecb.Playback(state.EntityManager);
    }
}
```

### 5.4 Object Groups & Categories
`ObjectDefinition` assets can be assigned to `ObjectGroups` (e.g., "Organic", "Mechanical", "Tier 1").

At runtime, use the `ObjectGroupMatcher` singleton buffer to perform incredibly fast `O(1)` checks to see if a specific `ObjectId` belongs to a `GroupId`.

```csharp
var groupMatcher = SystemAPI.GetSingletonBuffer<ObjectGroupMatcher>();

// Check if the target's ObjectId belongs to the "Armored" GroupId
if (groupMatcher.Matches(damageEffect.TargetGroupId, target.ObjectId))
{
    // Apply damage
}
```


## 1. System Design — Zero Cyclomatic Complexity

NASA's coding standards mandate strict predictability. We apply this to ECS by enforcing **Zero Cyclomatic Complexity (CC = 1) in System `OnUpdate` methods**.

A System is an **orchestrator**, not a calculator. It should query data, handle state dependencies, schedule jobs, and return. It should **never** contain `if`, `for`, `foreach`, or `while` loops containing game logic.

### 1.1 `ISystem` vs `SystemBase`
*   **RULE:** All gameplay systems must be `partial struct [Name]System : ISystem`.
*   **RULE:** All gameplay systems must have `[BurstCompile]` on the struct and the `OnUpdate` / `OnCreate` methods.
*   *Exception:* Use `SystemBase` ONLY when bridging to managed code (like UI ToolKit ViewModels) or interacting with `UnityEngine.Object`.

### 1.2 The Orchestrator Pattern

```csharp
// ❌ BAD: High Complexity, un-Burst-able managed calls, structural changes in loop
public partial class BadCombatSystem : SystemBase
{
    protected override void OnUpdate()
    {
        foreach (var (health, entity) in SystemAPI.Query<RefRW<Health>>().WithEntityAccess())
        {
            if (health.ValueRO.Current <= 0) // Branch
            {
                if (!EntityManager.HasComponent<Dead>(entity)) // Branch + Sync point
                {
                    EntityManager.AddComponent<Dead>(entity); // Structural change stall
                }
            }
        }
    }
}

// ✅ GOOD: CC = 1. ISystem orchestrates, ECB handles structural changes, Job handles logic.
[BurstCompile]
public partial struct EliteCombatSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var ecb = SystemAPI.GetSingleton<EndSimulationEntityCommandBufferSystem.Singleton>()
                           .CreateCommandBuffer(state.WorldUnmanaged).AsParallelWriter();

        // 1. Schedule Job
        state.Dependency = new CheckDeathJob { ECB = ecb }
            .ScheduleParallel(state.Dependency);
    }
}

[BurstCompile]
[WithNone(typeof(Dead))] // Narrow query eliminates internal branching!
public partial struct CheckDeathJob : IJobEntity
{
    public EntityCommandBuffer.ParallelWriter ECB;

    private void Execute(Entity entity, [ChunkIndexInQuery] int chunkIndex, in Health health)
    {
        // Math instead of branching where possible, or simple early outs.
        if (health.Current > 0f) return;
        
        ECB.AddComponent<Dead>(chunkIndex, entity);
    }
}
```

---

## 2. Advanced Custom Jobs (`BovineLabs.Core.Jobs`)

Unity's built-in jobs (`IJobEntity`, `IJobParallelFor`) are great, but `BovineLabs.Core` provides highly specialized jobs for extreme edge cases.

### 2.1 `IJobForThread`
When you need to divide a fixed range of work across a **known number of threads** (instead of relying on Unity's batching), use `IJobForThread`. Each worker thread receives a contiguous slice of indices.

```csharp
[BurstCompile]
private struct SpatialPartitionJob : IJobForThread
{
    [NativeDisableParallelForRestriction] public NativeArray<int> SpatialGrid;

    public void Execute(int index)
    {
        // 'index' is the iteration index, automatically sliced per-thread.
        SpatialGrid[index] = 0;
    }
}

// Usage: Schedule across exactly 4 worker threads
state.Dependency = new SpatialPartitionJob { SpatialGrid = grid }
    .ScheduleParallel(grid.Length, threadCount: 4, state.Dependency);
```

### 2.2 `IJobHashMapDefer` & `IJobParallelHashMapDefer`
Iterating over a `NativeHashMap` in a Burst job normally requires a single thread or extracting keys/values to arrays first. `BovineLabs.Core` allows you to iterate HashMaps directly in parallel.

```csharp
[BurstCompile]
private struct ProcessSpatialMapJob : IJobParallelHashMapDefer
{
    [ReadOnly] public NativeParallelMultiHashMap<int, Entity> SpatialMap;
    
    // Required interface method
    public void ExecuteNext(int entryIndex, int jobIndex)
    {
        // Use the BovineLabs .Read() extension to safely extract the key/value
        this.Read(SpatialMap, entryIndex, out int cellHash, out Entity occupant);
        
        // Process the occupant...
    }
}

// Usage: Pass the map, the minimum items per job (batch size), and dependency
state.Dependency = new ProcessSpatialMapJob { SpatialMap = map }
    .ScheduleParallel(map, minIndicesPerJobCount: 64, state.Dependency);
```

---

## 3. The `DynamicHashMap` Ecosystem

**The Problem:** You cannot store managed dictionaries or `NativeHashMap`s directly on an Entity. If a unit needs a local dictionary (e.g., an Inventory mapping `ItemId -> Count`), ECS developers usually resort to parallel arrays or sorting `DynamicBuffer`s.

**The BovineLabs Solution:** Reinterpreting `DynamicBuffer<byte>` as fully functional, zero-allocation HashMaps. The `BovineLabs.DynamicGenerator` source generator automatically creates the `.Initialize()` and `.AsMap()` extension methods for you.

### 3.1 Defining a Dynamic Map
Define a buffer component implementing the desired interface:

```csharp
// 1. Declare an Untyped Hash Map (Stores variable value types mapped to a FixedString)
[InternalBufferCapacity(0)]
public struct Blackboard : IDynamicUntypedHashMap<FixedString32Bytes>
{
    byte IDynamicUntypedHashMap<FixedString32Bytes>.Value { get; }
}

// 2. Declare a standard HashMap
[InternalBufferCapacity(0)]
public struct InventoryMap : IDynamicHashMap<int, int>
{
    byte IDynamicHashMap<int, int>.Value { get; }
}
```

### 3.2 Initializing (In a Baker or System)
Before using the map, you must allocate its internal headers via `Initialize()`.

```csharp
public class UnitBaker : Baker<UnitAuthoring>
{
    public override void Bake(UnitAuthoring authoring)
    {
        var entity = GetEntity(TransformUsageFlags.Dynamic);
        
        // Auto-generated by BovineLabs.DynamicGenerator
        var buffer = AddBuffer<InventoryMap>(entity);
        buffer.InitializeHashMap<InventoryMap, int, int>(capacity: 16);
    }
}
```

### 3.3 Using DynamicMaps in Systems
Always extract the map using `.AsMap()` before manipulating it.

```csharp
[BurstCompile]
public partial struct InventorySystem : IJobEntity
{
    public void Execute(DynamicBuffer<InventoryMap> inventoryBuffer)
    {
        // 1. Cast the raw bytes back into the HashMap wrapper
        var inventory = inventoryBuffer.AsMap();
        
        // 2. Standard Dictionary API!
        inventory.Add(itemId: 45, count: 10);
        
        if (inventory.TryGetValue(45, out var count))
        {
            inventory[45] = count + 1;
        }

        // 3. Ultra-fast batch additions
        // inventory.AddBatchUnsafe(keysArray, valuesArray);

        // 4. Enumeration
        foreach (var kvp in inventory)
        {
            var id = kvp.Key;
            var amount = kvp.Value;
        }
    }
}
```

### 3.4 `DynamicVariableMap` (Multi-Column Relational Data)
If you need to map a Key to multiple related Data Columns (like a micro-database per entity), use `IDynamicVariableMap`.

```csharp
using BovineLabs.Core.Iterators.Columns;

// Maps: TargetEntity -> (DamageFloat) AND (DamageTypeInt)
[InternalBufferCapacity(0)]
public struct DamageRelations : IDynamicVariableMap<Entity, float, int, OrderedListColumn<int>>
{
    byte IDynamicVariableMap<Entity, float, int, OrderedListColumn<int>>.Value { get; }
}
```

---

## 4. Advanced Iterators and Lookups

Standard `ComponentLookup<T>` has overhead when querying. `BovineLabs.Core` provides highly unsafe, highly optimized variants when you need to squeeze out maximum performance in tight loops.

### 4.1 `UnsafeComponentLookup<T>`
Use when you need to bypass Unity's parallel job safety checks manually. **Requires deep understanding of your job's write patterns to avoid race conditions.**

```csharp
public partial struct PhysicsSystem : ISystem
{
    private UnsafeComponentLookup<Velocity> unsafeVelocityLookup;

    public void OnCreate(ref SystemState state)
    {
        // Requires manual dependency injection in OnUpdate!
        unsafeVelocityLookup = state.GetUnsafeComponentLookup<Velocity>(isReadOnly: false);
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        unsafeVelocityLookup.Update(ref state);
        
        // Unsafe lookups can be written to concurrently if you guarantee 
        // no two threads touch the same Entity ID.
    }
}
```

### 4.2 High-Performance Manual Chunk Iteration
When `IJobEntity` or `IJobChunk` abstracts too much, and you need to manually parse chunks (especially with Enableable components), use `QueryEntityEnumerator` and `CustomChunkIterator`.

```csharp
// Iterating queries directly avoiding job scheduling overhead for tiny datasets
var queryEnumerator = new QueryEntityEnumerator(myQuery);

while (queryEnumerator.MoveNextChunk(out ArchetypeChunk chunk, out ChunkEntityEnumerator chunkEnumerator))
{
    // Extracts the array pointer safely
    var transforms = chunk.GetChunkComponentDataPtrRW(ref transformHandle);

    // chunkEnumerator automatically skips entities where components are disabled!
    while (chunkEnumerator.NextEntityIndex(out int i))
    {
        transforms[i].Position += new float3(0, 1, 0);
    }
}
```

### 4.3 Fallback Collections
When building highly concurrent systems, sometimes you want to use a `NativeParallelMultiHashMap` but you don't know the exact capacity needed. Overfilling it causes a crash.

Use `NativeParallelMultiHashMapFallback<TKey, TValue>`. It attempts to write to the fast HashMap, but if capacity is reached, it seamlessly falls back to a `NativeQueue`. Later, `Apply()` merges the fallback queue into the main map.

```csharp
// 1. Create with estimated capacity
var map = new NativeParallelMultiHashMapFallback<Entity, int>(EstimatedSize, Allocator.TempJob);

// 2. Write in parallel. If we guess wrong and it overflows, it won't crash!
var writer = map.AsWriter();
writer.Add(entity, damageAmount);

// 3. Apply handles recalculating buckets and merging the fallback queue safely
state.Dependency = map.Apply(state.Dependency, out var safeReader);
```

## 1. Facets (The `IFacet` System)

Unity’s native `IAspect` is useful, but it requires manual boilerplate for lookups, type handles, and query building. The `BovineLabs.Core` framework uses a Roslyn Source Generator (`BovineLabs.FacetGenerator`) to completely automate this process at compile time, providing a much more robust abstraction called `IFacet`.

### 1.1 Declaring a Facet
To create a facet, define a `partial struct` that implements `IFacet`. The generator will automatically create the `Lookup`, `TypeHandle`, `ResolvedChunk`, and `CreateQueryBuilder` definitions for you.

```csharp
using BovineLabs.Core;
using Unity.Entities;

// ✅ GOOD: A cleanly defined Facet. All properties are automatically wired.
public readonly partial struct CombatFacet : IFacet
{
    // 1. Standard Component Access
    private readonly RefRW<Health> health;
    private readonly RefRO<DefenseStats> defense;

    // 2. Enableable Component Access
    private readonly EnabledRefRO<Invulnerable> isInvulnerable;

    // 3. Buffer Access (Optional means the query won't strictly require it)[FacetOptional] 
    private readonly DynamicBuffer<StatusEffect> statusEffects;

    // 4. Nested Facets (Composing facets together)
    [Facet] 
    private readonly MovementFacet movement;

    // 5. Singleton Injection (Injected directly into Lookups and Handles)
    [Singleton] 
    private readonly CombatSettings singletonSettings;

    // REQUIRED: Must declare a partial Lookup struct to trigger generation for IJobEntity use
    public partial struct Lookup { } 
}
```

### 1.2 Using Facets in Systems and Jobs
Because `IFacet` generates `TypeHandle` and `Lookup` structs, you can use them seamlessly in both Chunk iterations and random-access lookups.

```csharp
[BurstCompile]
public partial struct CombatSystem : ISystem
{
    private CombatFacet.TypeHandle facetHandle;
    private CombatFacet.Lookup facetLookup;
    private EntityQuery query;

    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {
        // 1. Auto-generated initialization!
        this.facetHandle.Create(ref state);
        this.facetLookup.Create(ref state);
        this.query = CombatFacet.CreateQueryBuilder().Build(ref state);
    }[BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // 2. Update handles (automatically handles [Singleton] injection if present)
        this.facetHandle.Update(ref state);
        this.facetLookup.Update(ref state);

        // 3. Chunk Iteration via Facets
        state.Dependency = new ProcessCombatChunkJob
        {
            FacetHandle = this.facetHandle
        }.ScheduleParallel(this.query, state.Dependency);
    }
}

[BurstCompile]
private struct ProcessCombatChunkJob : IJobChunk
{
    public CombatFacet.TypeHandle FacetHandle;

    public void Execute(in ArchetypeChunk chunk, int unfilteredChunkIndex, bool useEnabledMask, in v128 chunkEnabledMask)
    {
        // Resolve the entire chunk into Facet instances
        var resolved = this.FacetHandle.Resolve(chunk);

        for (var i = 0; i < chunk.Count; i++)
        {
            CombatFacet combatData = resolved[i];
            // Execute logic on the facet...
        }
    }
}
```

---

## 2. Unified Entity Manipulation (`IEntityCommands`)

**The Problem:** In DOTS, you have 4 different ways to add a component to an entity:
1. `EntityManager.AddComponent<T>()` (Immediate)
2. `EntityCommandBuffer.AddComponent<T>()` (Deferred)
3. `EntityCommandBuffer.ParallelWriter.AddComponent<T>()` (Deferred Parallel)
4. `IBaker.AddComponent<T>()` (Baking time)

This forces developers to write duplicate setup functions for Entities spawned at runtime vs. Entities baked in the editor.

**The Solution:** Use `IEntityCommands`. It abstracts the underlying context so you write your entity composition logic **exactly once**.

### 2.1 The Shared Composition Method
Write static methods that accept `ref T commands where T : IEntityCommands`.

```csharp
public static class UnitFactory
{
    // Works identically in Bakers, Main Thread Systems, and Parallel Jobs!
    public static void SetupUnit<T>(ref T commands, float3 position, int team)
        where T : IEntityCommands
    {
        commands.AddComponent(LocalTransform.FromPosition(position));
        commands.AddComponent(new Team { Value = team });
        commands.AddComponent<Health>();
        commands.AddBuffer<DamageEvent>();
        
        commands.SetComponentEnabled<Stunned>(false);
    }
}
```

### 2.2 Using `IEntityCommands` Across Contexts

**In a Baker:**
```csharp
public class UnitBaker : Baker<UnitAuthoring>
{
    public override void Bake(UnitAuthoring authoring)
    {
        var entity = GetEntity(TransformUsageFlags.Dynamic);
        var commands = new BakerCommands(this, entity); // Pass the Baker
        
        UnitFactory.SetupUnit(ref commands, authoring.transform.position, authoring.Team);
    }
}
```

**In an IJobEntity (Parallel ECB):**
```csharp
[BurstCompile]
private partial struct SpawnUnitJob : IJobEntity
{
    public EntityCommandBuffer.ParallelWriter ECB;

    private void Execute([ChunkIndexInQuery] int chunkIndex, in SpawnRequest request)
    {
        var entity = ECB.Instantiate(chunkIndex, request.Prefab);
        var commands = new CommandBufferParallelCommands(ECB, chunkIndex, entity);
        
        UnitFactory.SetupUnit(ref commands, request.Position, request.Team);
    }
}
```

---

## 3. The Automated Lifecycle Pipeline

Sprinkling `EntityManager.DestroyEntity` or `ECB.AddComponent<T>` throughout random gameplay systems causes uncontrollable sync points, unpredictable order of operations, and fragmented chunk memory.

**NASA Rule:** Initialization and Destruction are strictly phased.

### 3.1 The Lifecycle Phases
We use `BovineLabs.Core.LifeCycle`. It provides two specific groups:
1. `InitializeSystemGroup` (Runs early, right after `BeginSimulationSystemGroup`).
2. `DestroySystemGroup` (Runs right before the end of the frame/scene unloading).

### 3.2 Initialization (`InitializeEntity`)
Instead of querying for "Entities without component X", we mark entities explicitly for initialization.

```csharp
// 1. Authoring
// Add the `LifeCycleAuthoring` MonoBehaviour to your prefab. 
// It automatically bakes the `InitializeEntity` component.

// 2. System
[UpdateInGroup(typeof(InitializeSystemGroup))]
public partial struct InitializeUnitSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Only queries entities freshly spawned that need setup
        new InitJob().ScheduleParallel();
    }
}

[BurstCompile]
[WithAll(typeof(InitializeEntity))] // Narrow query to only initializing entities
private partial struct InitJob : IJobEntity
{
    private void Execute(Entity entity, ref Health health)
    {
        health.Current = health.Max; // Perform initial setup
    }
}
```
*Note: `InitializeEntitySystem` automatically removes the `InitializeEntity` component at the end of the `InitializeSystemGroup`, so you never have to strip it manually.*

### 3.3 Destruction (`DestroyEntity`)
Never destroy an entity immediately. Always use the `DestroyEntity` enableable component.

```csharp
[BurstCompile]
private partial struct CheckHealthJob : IJobEntity
{
    // Inject the enableable reference for DestroyEntity
    private void Execute(in Health health, EnabledRefRW<DestroyEntity> destroy)
    {
        if (health.Current <= 0)
        {
            // Marking it enabled queues it for the DestroySystemGroup
            destroy.ValueRW = true;
        }
    }
}
```

### 3.4 Deep Hierarchy Destruction (`LinkedEntityGroup`)
When building complex entities (like a vehicle made of multiple entities), Unity's default `DestroyEntity` can leave orphaned children or leak memory if not handled carefully.

By using the `DestroyEntity` component, `BovineLabs.Core` handles this automatically:
*   `DestroyOnDestroySystem` runs first in the `DestroySystemGroup`.
*   It recursively traverses `LinkedEntityGroup`s of any entity marked with `DestroyEntity`.
*   It enables `DestroyEntity` on all child entities automatically, ensuring safe, total cleanup before actual memory deletion occurs.

### 3.5 Automated Timer Destruction (`DestroyTimer<T>`)
A very common pattern is "Destroy this projectile/VFX after X seconds". `BovineLabs.Core.Model` provides `DestroyTimer<T>` to automate this without custom ECBs.

```csharp
// 1. Define a simple float wrapper
public struct LifetimeTimer : IComponentData { public float Value; }

// 2. System orchestrates the timer[UpdateInGroup(typeof(SimulationSystemGroup))]
public partial struct LifetimeSystem : ISystem
{
    private DestroyTimer<LifetimeTimer> destroyTimer;

    public void OnCreate(ref SystemState state)
    {
        this.destroyTimer.OnCreate(ref state);
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Automatically decrements LifetimeTimer by DeltaTime.
        // If it hits 0, it automatically enables DestroyEntity!
        this.destroyTimer.OnUpdate(ref state);
    }
}
```

## 1. Advanced SubScene & World Management

In a standard Unity DOTS project, SubScenes load into every world indiscriminately. When building multiplayer games (Client/Server) or separating UI into a `ServiceWorld`, we need surgical control over *what* loads *where* and *when*.

### 1.1 World Targeting via `SubSceneSettings`
Do not rely on Unity's default Auto-Load for complex projects. We use `BovineLabs.Core.Authoring.SubScenes.SubSceneSettings` (a ScriptableObject) to map SubScenes to specific `WorldFlags`.

*   **RULE:** All gameplay SubScenes should be managed via `SubSceneSet` assets.
*   **TargetWorlds:** Define exactly where a scene goes (`Game`, `Service`, `Client`, `Server`, `Menu`).

```csharp
// Example Setup Workflow (No Code Required for Setup):
// 1. Create a SubSceneSettings asset in BovineLabs -> Settings
// 2. Define a "ServerOnly" SubSceneSet targeting SubSceneLoadFlags.Server
// 3. Define a "ClientOnly" SubSceneSet targeting SubSceneLoadFlags.Client
// 4. Place a SubSceneLoadAuthoring component in your Bootstrap scene to drive it.
```

### 1.2 Loading Behaviors and `AssetSet`
*   **Required Loading:** The system pauses the simulation (via `PauseGame`) until the SubScene is fully loaded. Use this for the initial level geometry.
*   **Asset Sets:** Sometimes you need managed `GameObject`s to exist alongside your ECS world (e.g., AudioSources, specialized UI). Use `AssetSet` in your `SubSceneSettings` to automatically instantiate standard GameObjects into specific worlds.

### 1.3 Runtime SubScene Control
Never use `SceneSystem.LoadSceneAsync` directly in gameplay code. Instead, interact with the `LoadSubScene` and `SubSceneLoaded` enableable components.

```csharp
[BurstCompile]
public partial struct PortalSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // 1. Trigger a load by enabling the LoadSubScene component on the scene entity
        foreach (var (loadTag, entity) in SystemAPI.Query<EnabledRefRW<LoadSubScene>>().WithAll<NextLevelTag>())
        {
            loadTag.ValueRW = true;
        }

        // 2. Check if a scene has finished loading
        foreach (var (loaded, entity) in SystemAPI.Query<EnabledRefRO<SubSceneLoaded>>().WithAll<NextLevelTag>())
        {
            // Teleport players, start logic!
        }
    }
}
```

---

## 2. High-Performance Physics States

**The Problem:** Unity Physics `CollisionEvent` and `TriggerEvent` are *stateless*. They fire every single frame two colliders intersect. Manually tracking `Enter`, `Stay`, and `Exit` states usually involves slow, single-threaded HashMaps and structural changes.

**The Solution:** `BovineLabs.Core.PhysicsStates` provides a fully Burst-compiled, 100% parallelized stateful event system. It automatically buffers Enter/Stay/Exit states directly onto your entities.

### 2.1 Setup & Authoring
Add the `StatefulCollisionEventAuthoring` or `StatefulTriggerEventAuthoring` MonoBehaviour to your colliders.

```csharp
// In the editor, check "Event Details" if you need contact points and impulses.
// If you only need to know "Did I hit a wall?", leave it unchecked to save memory.
```

### 2.2 Processing Stateful Events
Query the `DynamicBuffer<StatefulCollisionEvent>` or `DynamicBuffer<StatefulTriggerEvent>`. These buffers are automatically cleared and repopulated every frame by the BovineLabs physics pipeline.

```csharp[UpdateInGroup(typeof(SimulationSystemGroup))]
[UpdateAfter(typeof(StatefulCollisionEventSystem))] // MUST run after the generation system
[BurstCompile]
public partial struct ProcessSpikesSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        new ProcessSpikeDamageJob().ScheduleParallel();
    }
}

[BurstCompile]
private partial struct ProcessSpikeDamageJob : IJobEntity
{
    // The buffer contains ALL collisions for this entity this frame
    private void Execute(Entity entity, ref Health health, in DynamicBuffer<StatefulCollisionEvent> collisions)
    {
        foreach (var collision in collisions)
        {
            // We ONLY want to apply damage on the exact frame they ENTER the spikes
            if (collision.State == StatefulEventState.Enter)
            {
                // Note: collision.EntityB is ALWAYS the 'other' entity. 
                // The BovineLabs system guarantees it formats the buffer relative to the entity being iterated.
                health.Current -= 50f;
                
                // If "EventDetails" was checked on the authoring component, we can get impact data:
                if (collision.TryGetDetails(out var details))
                {
                    var force = details.EstimatedImpulse;
                    var contactPoint = details.AverageContactPointPosition;
                }
            }
        }
    }
}
```

---

## 3. NetCode Relevancy (Interest Management)

**The Problem:** In Unity NetCode, sending every ghost (networked entity) to every client uses massive bandwidth.
**The Solution:** The BovineLabs Relevancy extension provides a spatial-hashing-based interest management system. Ghosts are only serialized to clients if they fall within the client's `InputBounds`.

### 3.1 Relevancy Components
*   `InputBounds`: Added to the connection entity (e.g., the player's camera or avatar). Defines the AABB of what the player can "see".
*   `RelevanceAlways`: Added to ghosts that must ALWAYS be sent to everyone (e.g., GameState, Scoreboards).
*   `RelevanceManual`: Opts a ghost out of the spatial grid so you can manage its relevancy via custom logic.

### 3.2 Automated Spatial Relevancy
The `RelevancySystem` automatically quantizes the `InputBounds` of all clients, hashes them against a `SpatialMap`, and flags ghosts as relevant or irrelevant based on the `RelevanceConfig`. You do not need to write custom NetCode relevancy jobs for 95% of use cases; simply attach the bounds to the player!

---

## 4. Deterministic Pause System

**The Problem:** Disabling system updates to "pause" the game causes Unity's `SystemAPI.Time.ElapsedTime` to continue ticking in the background. When you unpause, the `FixedStepSimulationSystemGroup` sees a massive time gap and attempts to run hundreds of catch-up ticks instantly, causing massive lag spikes and physics explosions.

**The Solution:** `BovineLabs.Core.Pause`. It intercepts the `IRateManager` of the world. When paused, it literally freezes ECS time progression, meaning zero catch-up ticks occur when unpaused.

### 4.1 Triggering a Pause
Use the `PauseGame` component to pause the world.

```csharp
[BurstCompile]
public partial struct PauseInputSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        if (InputPressed_Escape)
        {
            if (PauseGame.IsPaused(ref state))
            {
                PauseGame.Unpause(ref state);
            }
            else
            {
                // PauseAll = false means normal gameplay pauses, but UI/Service systems can keep running
                PauseGame.Pause(ref state, pauseAll: false); 
            }
        }
    }
}
```

### 4.2 Controlling What Updates While Paused
By default, standard gameplay systems will stop updating. You control exceptions using marker interfaces:

*   `IDisableWhilePaused`: Add this to a custom `ComponentSystemGroup` to explicitly shut it down during a pause. (BovineLabs automatically applies this to `FixedStepSimulationSystemGroup`).
*   `IUpdateWhilePaused`: Add this to any `ISystem` (like a UI Renderer or a Menu input processor) to force it to continue updating even while the world is paused.

```csharp
// ✅ GOOD: This system drives the pause menu UI, so it MUST run while the game is paused.
[UpdateInGroup(typeof(PresentationSystemGroup))]
[BurstCompile]
public partial struct PauseMenuRenderSystem : ISystem, IUpdateWhilePaused
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // UI Logic here...
    }
}
```