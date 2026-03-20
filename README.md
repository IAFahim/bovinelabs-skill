# BovineLabs Core Unity ECS Standards

> **Audience:** Developers working on Unity projects utilizing Entities (DOTS), Burst, Jobs, and the `BovineLabs.Core` framework.
> **Philosophy:** Zero cyclomatic complexity systems, strict zero-allocation hot paths, maximum Burst compliance, and leveraging `BovineLabs` code-generation and extensions to eliminate boilerplate.


## Table of Contents

0. [Code Comment Policy — The Absolute Rule](#0-code-comment-policy--the-absolute-rule)
1. [Assembly Definition Architecture](#1-assembly-definition-architecture)
2. [World & Bootstrap Architecture](#2-world--bootstrap-architecture)
3. [Core Utilities: Logging, Math, and Assertions](#3-core-utilities-logging-math-and-assertions)
4. [Naming Conventions & Layout](#4-naming-conventions--layout)
5. [Standard Component Design Rules](#5-standard-component-design-rules)
6. [The `IEnableableComponent` vs. Structural Changes](#6-the-ienableablecomponent-vs-structural-changes)
7. [The `KSettings` System (Burst-Safe String IDs)](#7-the-ksettings-system-burst-safe-string-ids)
8. [Global vs. World-Specific Settings](#8-global-vs-world-specific-settings)
9. [`[Singleton]` Buffers (The Many-to-One Pattern)](#9-singleton-buffers-the-many-to-one-pattern)
10. [Object Management: `IUID` and `ObjectDefinition`](#10-object-management-iuid-and-objectdefinition)
11. [System Design — Zero Cyclomatic Complexity](#11-system-design--zero-cyclomatic-complexity)
12. [Advanced Custom Jobs (`BovineLabs.Core.Jobs`)](#12-advanced-custom-jobs-bovinelabscorejobs)
13. [The `DynamicHashMap` Ecosystem](#13-the-dynamichashmap-ecosystem)
14. [Advanced Iterators and Lookups](#14-advanced-iterators-and-lookups)
15. [Facets (The `IFacet` System)](#15-facets-the-ifacet-system)
16. [Unified Entity Manipulation (`IEntityCommands`)](#16-unified-entity-manipulation-ientitycommands)
17. [The Automated Lifecycle Pipeline](#17-the-automated-lifecycle-pipeline)
18. [Advanced SubScene & World Management](#18-advanced-subscene--world-management)
19. [High-Performance Physics States](#19-high-performance-physics-states)
20. [NetCode Relevancy (Interest Management)](#20-netcode-relevancy-interest-management)
21. [Deterministic Pause System](#21-deterministic-pause-system)


## 0. Code Comment Policy — The Absolute Rule

> **RULE: Production code contains zero comments. No exceptions.**

Self-documenting names, clear structure, and well-named systems replace every comment. If you feel the urge to write a comment, rename the variable, extract the method, or redesign the structure until the comment is unnecessary.

**Why:** Comments lie. Code changes; comments don't. A comment that was accurate six months ago is a trap today. Naming things correctly is permanent documentation.

```csharp
// ❌ PRODUCTION CODE VIOLATION — never ship this
private void Execute(ref Health h, in Regen r) // Apply regen to health
{
    h.Current = math.min(h.Current + r.Rate * dt, h.Max); // Clamp to max
}

// ✅ CORRECT — the code is the documentation
private void Execute(ref Health health, in Regeneration regeneration)
{
    health.Current = math.min(health.Current + regeneration.Rate * deltaTime, health.Max);
}
```

> **Note on this document:** All code examples throughout this README contain inline comments and annotations **for teaching purposes only**. These explain *why* a pattern is used so you can understand it deeply. This explanatory style must never appear in your actual `.cs` files. Your codebase is not a tutorial.

---

## 1. Assembly Definition Architecture

The foundation of our ECS architecture relies on strict compilation boundaries. If code can reach where it shouldn't, architecture degrades.

### 1.1 `autoReferenced: false` is Non-Negotiable

Every assembly must explicitly declare its dependencies. `autoReferenced: true` is banned.

- **Why?** Editor-only types will leak into player builds, build times will skyrocket, and circular dependencies will form.

### 1.2 The Six-Layer Assembly Pattern

Use the `BovineLabs Assembly Builder` (`BovineLabs -> Tools -> Assembly Builder`) to generate compliant layers.

| Assembly            | Purpose                                      | Constraint / Attributes                             | Key Rule                               |
|---------------------|----------------------------------------------|-----------------------------------------------------|----------------------------------------|
| `Feature.Data`      | `IComponentData`, `IBufferElementData`, Tags | None (Ships everywhere)                             | **No logic. Pure data structs only.**  |
| `Feature`           | `ISystem`, Jobs, ViewModels                  | None                                                | No MonoBehaviours. No Editor types.    |
| `Feature.Authoring` | Bakers, MonoBehaviours                       | `UNITY_EDITOR` / `[DisableAutoTypeRegistration]`    | **Never** compile into player builds.  |
| `Feature.Debug`     | Debug panels, diagnostic tools               | `UNITY_EDITOR \|\| BL_DEBUG`                        | Ships only in dev builds.              |
| `Feature.Editor`    | Custom inspectors (`ElementEditor`)          | `includePlatforms: [Editor]`                        | Tools for designers.                   |
| `Feature.Tests`     | Unit & Integration tests                     | `TestAssemblies` / `[DisableAutoCreation]`          | Test systems never run in live worlds. |

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

- **GameWorld:** Main simulation world.
- **ServiceWorld:** Runs background services, UI synchronization, and systems that persist across scene loads.
- **MenuWorld:** Exists for main menus without spinning up heavy physics/gameplay systems.

```csharp
// ✅ GOOD — Custom bootstrap initializing the BovineLabs ecosystem
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

// ✅ System only runs in the persistent Service world
[WorldSystemFilter(Worlds.Service)]
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

// ✅ GOOD: Check.Assume tells Burst compiler to optimize under this assumption.
// Only evaluates if ENABLE_UNITY_COLLECTIONS_CHECKS or UNITY_DOTS_DEBUG is defined.
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

*Output Format:* `[Level] | [Frame] | [World] | [Message]` → `D | 1204 | Game | Processing 5 events`

### 3.3 SIMD Math: `mathex`

When processing arrays, always use `BovineLabs.Core.Utility.mathex`. It leverages `float4` and `int4` block processing for massive speedups.

```csharp
// ❌ BAD: Iterating an array manually
float max = float.MinValue;
for (int i = 0; i < array.Length; i++) max = math.max(max, array[i]);

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

- **Format:** `[Company].[Project].[Feature]` (e.g., `BovineLabs.Combat`)
- **Authoring:** `[Company].[Project].[Feature].Authoring`

### 4.2 Component Naming

Components describe WHAT an entity IS or HAS, not what it DOES.

- **Tags:** Adjectives or States (e.g., `Dead`, `Stunned`, `Invulnerable`). *Never* suffix with "Tag".
- **Data:** Nouns (e.g., `Health`, `Velocity`). *Never* suffix with "Component" or "Data".
- **Buffers:** Singular Noun (e.g., `InventoryItem`, `Waypoint`).

### 4.3 System & Job Naming

- **Systems:** `[Verb][Subject]System` (e.g., `ApplyDamageSystem`, `ProcessInputSystem`).
- **Jobs:** `[Verb][Subject]Job`.
  - If using `IJobChunk`, suffix with `ChunkJob`.
  - If using `IJobEntity`, just suffix `Job`.

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
    public bool IsDead => Current <= 0;        // VIOLATION
    public void TakeDamage(float amt) { ... }  // VIOLATION
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


## 6. The `IEnableableComponent` vs. Structural Changes

Structural changes (adding/removing components or destroying entities) stall the main thread, invalidate caches, and prevent parallel processing.

### 6.1 Structural Change Rules

- **RULE:** Never use `EntityCommandBuffer.AddComponent<T>` or `RemoveComponent<T>` for state toggles that happen frequently (e.g., `IsStunned`, `IsSelected`).
- **RULE:** Always use `IEnableableComponent` for transient states.

```csharp
// ❌ BAD: Causes a chunk move and memory allocation every time a unit gets stunned.
public struct Stunned : IComponentData { }
// Usage: ecb.AddComponent<Stunned>(entity);

// ✅ GOOD: Zero allocation, zero chunk moves. Memory is pre-allocated.
public struct Stunned : IComponentData, IEnableableComponent { }
// Usage: SystemAPI.SetComponentEnabled<Stunned>(entity, true);
```

### 6.2 Profiling Change Filters

If a system uses `[WithChangeFilter(typeof(MyComponent))]` but that component is written to every frame, the filter is useless.

- **RULE:** Add `[ChangeFilterTracking]` to highly-queried components.
- Use the `BovineLabs -> Tools -> Change Filter` window to ensure your component isn't thrashing (changing >85% of frames).


## 7. The `KSettings` System (Burst-Safe String IDs)

Enums are rigid and require recompilation to change. Hardcoding integer IDs is prone to conflicts. Comparing strings inside Burst jobs is heavily restricted and slow.

**The Solution:** We use `KSettings<T, TV>`. It allows designers to define human-readable strings in a ScriptableObject, which are deterministically converted to unmanaged values (`byte`, `int`, `short`) that are 100% Burst-compatible.

### 7.1 Defining a K-Setting

Create a class inheriting from `KSettingsBase<T, TV>` or `KSettings<T, TV>`.

```csharp
using BovineLabs.Core.Keys;

// ✅ Defines a mapping of string -> byte for Character States
public class CharacterStates : KSettings<CharacterStates, byte>
{
    // Optionally provide hardcoded defaults; designers can add more in the Editor
    protected override IEnumerable<NameValue<byte>> SetReset()
    {
        yield return new NameValue<byte>("idle", 0);
        yield return new NameValue<byte>("running", 1);
        yield return new NameValue<byte>("attacking", 2);
    }
}
```

### 7.2 Authoring with `[K]`

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

### 7.3 Resolving Inside Burst

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


## 8. Global vs. World-Specific Settings

We heavily rely on `ScriptableObjects` for design data, but we must cleanly bridge them into the ECS world.

### 8.1 `SettingsSingleton<T>` (Global / Pre-World Data)

Use `SettingsSingleton<T>` for data that must exist *before* ECS worlds spin up (e.g., Boot configurations, UI themes, Master Audio volumes).

- **RULE:** They are automatically collected and included in the build via `CoreBuildSetup`. Do not put them in a `Resources` folder.
- **Access:** `MyGlobalSettings.I` (Available instantly, everywhere).

### 8.2 `SettingsBase` (World-Baked Data)

For ECS gameplay data, inherit from `SettingsBase`.

- Use `[SettingsGroup("Category")]` for the Editor Window.
- Use `[SettingsWorld("Client", "Server")]` to target specific worlds (crucial for Netcode).

```csharp
[SettingsGroup("Combat")]
[SettingsWorld("Server", "Local")] // Strip from ThinClients
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


## 9. `[Singleton]` Buffers (The Many-to-One Pattern)

When making modular games, multiple subscenes or mods might contribute to a single list of data (e.g., a master list of all craftable items). We use `BovineLabs.Core.Settings.SingletonAttribute` to automatically merge them.

### 9.1 Defining and Baking

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

### 9.2 Runtime Merging & The Initialization Signal

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


## 10. Object Management: `IUID` and `ObjectDefinition`

Traditional ECS requires passing `Entity` prefab references. This breaks down when:

1. You need to reference a prefab across a network (Entities have different IDs on Client vs Server).
2. You need to save/load a reference to disk.
3. You need a designer to pick a prefab in an Inspector for a component that hasn't been baked yet.

**The Solution:** We use `ObjectId` (a deterministic `int`) backed by `ObjectDefinition` assets.

### 10.1 Creating Object Definitions

1. Implement `IUID` on a ScriptableObject, or use the built-in `ObjectDefinition`.
2. Use the `[AutoRef]` attribute so it automatically registers itself to a Manager asset.
3. The ID is automatically generated, branch-safe, and immutable once created.

```csharp
public struct SpawnRequest : IComponentData
{
    public ObjectId PrefabId; // ✅ Network/Save safe. Deterministic integer.
    public float3 Position;
}
```

### 10.2 Authoring with `ObjectDefinition`

```csharp
public class SpawnerAuthoring : MonoBehaviour
{
    // The inspector provides a custom search window (supports "ca=" category filters)
    [SearchContext("ca=enemy", ObjectDefinition.SearchProviderType)]
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

### 10.3 Instantiating at Runtime (`ObjectDefinitionRegistry`)

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

### 10.4 Object Groups & Categories

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


## 11. System Design — Zero Cyclomatic Complexity

NASA's coding standards mandate strict predictability. We apply this to ECS by enforcing **Zero Cyclomatic Complexity (CC = 1) in System `OnUpdate` methods**.

A System is an **orchestrator**, not a calculator. It should query data, handle state dependencies, schedule jobs, and return. It should **never** contain `if`, `for`, `foreach`, or `while` loops containing game logic.

### 11.1 `ISystem` vs `SystemBase`

- **RULE:** All gameplay systems must be `partial struct [Name]System : ISystem`.
- **RULE:** All gameplay systems must have `[BurstCompile]` on the struct and the `OnUpdate` / `OnCreate` methods.
- *Exception:* Use `SystemBase` ONLY when bridging to managed code (like UI Toolkit ViewModels) or interacting with `UnityEngine.Object`.

### 11.2 The Orchestrator Pattern

```csharp
// ❌ BAD: High complexity, un-Burst-able managed calls, structural changes in loop
public partial class BadCombatSystem : SystemBase
{
    protected override void OnUpdate()
    {
        foreach (var (health, entity) in SystemAPI.Query<RefRW<Health>>().WithEntityAccess())
        {
            if (health.ValueRO.Current <= 0)             // Branch
            {
                if (!EntityManager.HasComponent<Dead>(entity)) // Branch + Sync point
                {
                    EntityManager.AddComponent<Dead>(entity);  // Structural change stall
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

        // Schedule Job
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


## 12. Advanced Custom Jobs (`BovineLabs.Core.Jobs`)

Unity's built-in jobs (`IJobEntity`, `IJobParallelFor`) are great, but `BovineLabs.Core` provides highly specialized jobs for extreme edge cases.

### 12.1 `IJobForThread`

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

### 12.2 `IJobHashMapDefer` & `IJobParallelHashMapDefer`

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


## 13. The `DynamicHashMap` Ecosystem

**The Problem:** You cannot store managed dictionaries or `NativeHashMap`s directly on an Entity. If a unit needs a local dictionary (e.g., an Inventory mapping `ItemId -> Count`), ECS developers usually resort to parallel arrays or sorting `DynamicBuffer`s.

**The BovineLabs Solution:** Reinterpreting `DynamicBuffer<byte>` as fully functional, zero-allocation HashMaps. The `BovineLabs.DynamicGenerator` source generator automatically creates the `.Initialize()` and `.AsMap()` extension methods for you.

### 13.1 Defining a Dynamic Map

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

### 13.2 Initializing (In a Baker or System)

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

### 13.3 Using DynamicMaps in Systems

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

### 13.4 `DynamicVariableMap` (Multi-Column Relational Data)

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


## 14. Advanced Iterators and Lookups

Standard `ComponentLookup<T>` has overhead when querying. `BovineLabs.Core` provides highly unsafe, highly optimized variants when you need to squeeze out maximum performance in tight loops.

### 14.1 `UnsafeComponentLookup<T>`

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

### 14.2 High-Performance Manual Chunk Iteration

When `IJobEntity` or `IJobChunk` abstracts too much, and you need to manually parse chunks (especially with Enableable components), use `QueryEntityEnumerator` and `CustomChunkIterator`.

```csharp
// Iterating queries directly, avoiding job scheduling overhead for tiny datasets
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

### 14.3 Fallback Collections

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


## 15. Facets (The `IFacet` System)

Unity's native `IAspect` is useful, but it requires manual boilerplate for lookups, type handles, and query building. The `BovineLabs.Core` framework uses a Roslyn Source Generator (`BovineLabs.FacetGenerator`) to completely automate this process at compile time, providing a much more robust abstraction called `IFacet`.

### 15.1 Declaring a Facet

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

    // 3. Buffer Access (Optional means the query won't strictly require it)
    [FacetOptional]
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

### 15.2 Using Facets in Systems and Jobs

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
        // Auto-generated initialization!
        this.facetHandle.Create(ref state);
        this.facetLookup.Create(ref state);
        this.query = CombatFacet.CreateQueryBuilder().Build(ref state);
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Update handles (automatically handles [Singleton] injection if present)
        this.facetHandle.Update(ref state);
        this.facetLookup.Update(ref state);

        // Chunk Iteration via Facets
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


## 16. Unified Entity Manipulation (`IEntityCommands`)

**The Problem:** In DOTS, you have 4 different ways to add a component to an entity:

1. `EntityManager.AddComponent<T>()` (Immediate)
2. `EntityCommandBuffer.AddComponent<T>()` (Deferred)
3. `EntityCommandBuffer.ParallelWriter.AddComponent<T>()` (Deferred Parallel)
4. `IBaker.AddComponent<T>()` (Baking time)

This forces developers to write duplicate setup functions for Entities spawned at runtime vs. Entities baked in the editor.

**The Solution:** Use `IEntityCommands`. It abstracts the underlying context so you write your entity composition logic **exactly once**.

### 16.1 The Shared Composition Method

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

### 16.2 Using `IEntityCommands` Across Contexts

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

**In an `IJobEntity` (Parallel ECB):**

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


## 17. The Automated Lifecycle Pipeline

Sprinkling `EntityManager.DestroyEntity` or `ECB.AddComponent<T>` throughout random gameplay systems causes uncontrollable sync points, unpredictable order of operations, and fragmented chunk memory.

**NASA Rule:** Initialization and Destruction are strictly phased.

### 17.1 The Lifecycle Phases

We use `BovineLabs.Core.LifeCycle`. It provides two specific groups:

1. `InitializeSystemGroup` (Runs early, right after `BeginSimulationSystemGroup`).
2. `DestroySystemGroup` (Runs right before the end of the frame/scene unloading).

### 17.2 Initialization (`InitializeEntity`)

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

### 17.3 Destruction (`DestroyEntity`)

Never destroy an entity immediately. Always use the `DestroyEntity` enableable component.

```csharp
[BurstCompile]
private partial struct CheckHealthJob : IJobEntity
{
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

### 17.4 Deep Hierarchy Destruction (`LinkedEntityGroup`)

When building complex entities (like a vehicle made of multiple entities), Unity's default `DestroyEntity` can leave orphaned children or leak memory if not handled carefully.

By using the `DestroyEntity` component, `BovineLabs.Core` handles this automatically:

- `DestroyOnDestroySystem` runs first in the `DestroySystemGroup`.
- It recursively traverses `LinkedEntityGroup`s of any entity marked with `DestroyEntity`.
- It enables `DestroyEntity` on all child entities automatically, ensuring safe, total cleanup before actual memory deletion occurs.

### 17.5 Automated Timer Destruction (`DestroyTimer<T>`)

A very common pattern is "Destroy this projectile/VFX after X seconds". `BovineLabs.Core.Model` provides `DestroyTimer<T>` to automate this without custom ECBs.

```csharp
// 1. Define a simple float wrapper
public struct LifetimeTimer : IComponentData { public float Value; }

// 2. System orchestrates the timer
[UpdateInGroup(typeof(SimulationSystemGroup))]
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


## 18. Advanced SubScene & World Management

In a standard Unity DOTS project, SubScenes load into every world indiscriminately. When building multiplayer games (Client/Server) or separating UI into a `ServiceWorld`, we need surgical control over *what* loads *where* and *when*.

### 18.1 World Targeting via `SubSceneSettings`

Do not rely on Unity's default Auto-Load for complex projects. We use `BovineLabs.Core.Authoring.SubScenes.SubSceneSettings` (a ScriptableObject) to map SubScenes to specific `WorldFlags`.

- **RULE:** All gameplay SubScenes should be managed via `SubSceneSet` assets.
- **TargetWorlds:** Define exactly where a scene goes (`Game`, `Service`, `Client`, `Server`, `Menu`).

```csharp
// Example Setup Workflow (No Code Required for Setup):
// 1. Create a SubSceneSettings asset in BovineLabs -> Settings
// 2. Define a "ServerOnly" SubSceneSet targeting SubSceneLoadFlags.Server
// 3. Define a "ClientOnly" SubSceneSet targeting SubSceneLoadFlags.Client
// 4. Place a SubSceneLoadAuthoring component in your Bootstrap scene to drive it.
```

### 18.2 Loading Behaviors and `AssetSet`

- **Required Loading:** The system pauses the simulation (via `PauseGame`) until the SubScene is fully loaded. Use this for the initial level geometry.
- **Asset Sets:** Sometimes you need managed `GameObject`s to exist alongside your ECS world (e.g., AudioSources, specialized UI). Use `AssetSet` in your `SubSceneSettings` to automatically instantiate standard GameObjects into specific worlds.

### 18.3 Runtime SubScene Control

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


## 19. High-Performance Physics States

**The Problem:** Unity Physics `CollisionEvent` and `TriggerEvent` are *stateless*. They fire every single frame two colliders intersect. Manually tracking `Enter`, `Stay`, and `Exit` states usually involves slow, single-threaded HashMaps and structural changes.

**The Solution:** `BovineLabs.Core.PhysicsStates` provides a fully Burst-compiled, 100% parallelized stateful event system. It automatically buffers Enter/Stay/Exit states directly onto your entities.

### 19.1 Setup & Authoring

Add the `StatefulCollisionEventAuthoring` or `StatefulTriggerEventAuthoring` MonoBehaviour to your colliders. In the inspector, enable **Event Details** if you need contact points and impulses. Leave it unchecked if you only need to know whether a collision occurred — this saves memory per entity.

### 19.2 Processing Stateful Events

Query the `DynamicBuffer<StatefulCollisionEvent>` or `DynamicBuffer<StatefulTriggerEvent>`. These buffers are automatically cleared and repopulated every frame by the BovineLabs physics pipeline.

```csharp
[UpdateInGroup(typeof(SimulationSystemGroup))]
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
                // collision.EntityB is ALWAYS the 'other' entity.
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


## 20. NetCode Relevancy (Interest Management)

**The Problem:** In Unity NetCode, sending every ghost (networked entity) to every client uses massive bandwidth.

**The Solution:** The BovineLabs Relevancy extension provides a spatial-hashing-based interest management system. Ghosts are only serialized to clients if they fall within the client's `InputBounds`.

### 20.1 Relevancy Components

- `InputBounds`: Added to the connection entity (e.g., the player's camera or avatar). Defines the AABB of what the player can "see".
- `RelevanceAlways`: Added to ghosts that must ALWAYS be sent to everyone (e.g., GameState, Scoreboards).
- `RelevanceManual`: Opts a ghost out of the spatial grid so you can manage its relevancy via custom logic.

### 20.2 Automated Spatial Relevancy

The `RelevancySystem` automatically quantizes the `InputBounds` of all clients, hashes them against a `SpatialMap`, and flags ghosts as relevant or irrelevant based on the `RelevanceConfig`. You do not need to write custom NetCode relevancy jobs for 95% of use cases — simply attach the bounds to the player!


## 21. Deterministic Pause System

**The Problem:** Disabling system updates to "pause" the game causes Unity's `SystemAPI.Time.ElapsedTime` to continue ticking in the background. When you unpause, the `FixedStepSimulationSystemGroup` sees a massive time gap and attempts to run hundreds of catch-up ticks instantly, causing massive lag spikes and physics explosions.

**The Solution:** `BovineLabs.Core.Pause` intercepts the `IRateManager` of the world. When paused, it literally freezes ECS time progression, meaning zero catch-up ticks occur when unpaused.

### 21.1 Triggering a Pause

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

### 21.2 Controlling What Updates While Paused

By default, standard gameplay systems will stop updating. You control exceptions using marker interfaces:

- `IDisableWhilePaused`: Add this to a custom `ComponentSystemGroup` to explicitly shut it down during a pause. (BovineLabs automatically applies this to `FixedStepSimulationSystemGroup`).
- `IUpdateWhilePaused`: Add this to any `ISystem` (like a UI Renderer or a Menu input processor) to force it to continue updating even while the world is paused.

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

---

---

# Part 2 — The Testing Bible
## Absolute Determinism, Zero-Leak Validation, and Sub-Millisecond Benchmarking

> **Audience:** Every developer committing code to this project.
> **Philosophy:** Untested ECS code is broken code. In a multithreaded, Burst-compiled, unmanaged memory environment, the compiler will not save you from race conditions, memory leaks, or chunk fragmentation. **Testing is not an afterthought; it is the mathematical proof that your systems function.**


## Part 2 — Table of Contents

1. [The Philosophy of DOTS Testing](#1-the-philosophy-of-dots-testing)
2. [Assembly & Fixture Architecture](#2-assembly--fixture-architecture)
3. [Memory Leak Detection (The Zero-Tolerance Rule)](#3-memory-leak-detection-the-zero-tolerance-rule)
4. [Level 1: Pure Function Testing](#4-level-1-pure-function-testing)
5. [Level 2: Job Testing in Isolation](#5-level-2-job-testing-in-isolation)
6. [Level 3: System Orchestration Testing](#6-level-3-system-orchestration-testing)
7. [Advanced: Testing DynamicHashMaps & Custom Collections](#7-advanced-testing-dynamichashmaps--custom-collections)
8. [Advanced: Testing Facets & IEnableableComponents](#8-advanced-testing-facets--ienableablecomponents)
9. [Advanced: Testing Blob Assets](#9-advanced-testing-blob-assets)
10. [Performance Benchmarking (The NASA Speed Standard)](#10-performance-benchmarking-the-nasa-speed-standard)
11. [Math Assertions & Helpers](#11-math-assertions--helpers)
12. [The Elite PR Testing Checklist](#12-the-elite-pr-testing-checklist)


## 1. The Philosophy of DOTS Testing

In Object-Oriented Programming (OOP), testing requires heavy mocking, dependency injection, and interface stubbing. **Data-Oriented Design (DOD) makes testing vastly superior and simpler.**

Because we enforce **Zero Cyclomatic Complexity (CC=1)** in our systems (as outlined in Part 3), testing becomes a pure mathematical exercise:
`Input Data -> Execute Job/System -> Output Data`.

**Why we test rigorously in ECS:**
1. **Unmanaged Memory:** A missed `.Dispose()` will crash the game.
2. **Burst Compilation:** Managed fallbacks or unsupported IL instructions will silently destroy performance.
3. **Race Conditions:** Parallel jobs writing to the same entity components will cause non-deterministic bugs.
4. **Structural Changes:** Accidental structural changes inside a job will invalidate chunk arrays.


## 2. Assembly & Fixture Architecture

Test assemblies must be strictly quarantined from runtime assemblies. If a test system leaks into a production build, it will corrupt the live `SimulationSystemGroup`.

### 2.1 The Test Assembly Definition (`Feature.Tests.asmdef`)

```json
{
  "name": "Feature.Tests",
  "references":[
    "BovineLabs.Core",
    "BovineLabs.Testing",
    "Unity.Burst",
    "Unity.Collections",
    "Unity.Entities",
    "Unity.Mathematics",
    "Unity.PerformanceTesting",
    "UnityEngine.TestRunner",
    "UnityEditor.TestRunner"
  ],
  "optionalUnityReferences": [ "TestAssemblies" ],
  "includePlatforms": [ "Editor" ],
  "autoReferenced": false
}
```

### 2.2 The Non-Negotiable `[DisableAutoCreation]`
Every test assembly **MUST** have an `AssemblyInfo.cs` at its root containing:
```csharp
// Feature.Tests/AssemblyInfo.cs
using Unity.Entities;

// CRITICAL: Prevents dummy test systems from injecting themselves into the real game world
[assembly: DisableAutoCreation]
```

### 2.3 The `ECSTestsFixture` Base Class
Never write a raw NUnit `[Test]`. **Always** inherit from `BovineLabs.Testing.ECSTestsFixture`.

This base class automatically:
1. Backs up the current Unity `PlayerLoop`.
2. Creates a completely isolated, pristine `World` named "Test World".
3. Forces the `JobsUtility.JobDebuggerEnabled = true` to catch race conditions.
4. Initializes the `BLLogger` and `BlobAssetStore` specifically for the test.
5. Safely tears down and disposes the world and all tracked jobs after every test.

```csharp
using BovineLabs.Testing;
using NUnit.Framework;

public class MyCombatTests : ECSTestsFixture
{
    // You have access to:
    // this.World (The isolated test world)
    // this.Manager (The EntityManager for this.World)
    // this.ManagerDebug (For internal consistency checks)
}
```


## 3. Memory Leak Detection (The Zero-Tolerance Rule)

In ECS, creating a `NativeArray` or `DynamicHashMap` with `Allocator.Persistent` or `Allocator.TempJob` without a corresponding `.Dispose()` causes a memory leak.

**Rule:** Every test that allocates memory or tests a system that allocates memory MUST be marked with `[TestLeakDetection]`.

### 3.1 How `[TestLeakDetection]` Works
Provided by `BovineLabs.Testing`, this attribute acts as an NUnit `TestAction`.
* **Before Test:** It calls `UnsafeUtility.ForgiveLeaks()` and forces `NativeLeakDetectionMode.Enabled`.
* **After Test:** It calls `UnsafeUtility.CheckForLeaks()`. If even 1 byte is leaked, the test **instantly fails**.

```csharp
// ❌ BAD: A leaky test that will pass but crash the game later
[Test]
public void LeakyTest()
{
    var list = new NativeList<int>(10, Allocator.TempJob);
    Assert.AreEqual(0, list.Length);
    // FAIL: Forgot list.Dispose();
}

// ✅ GOOD: The leak detection will catch the missing Dispose()
[Test]
[TestLeakDetection] // <--- REQUIRED
public void SafeTest()
{
    using var list = new NativeList<int>(10, Allocator.TempJob);
    Assert.AreEqual(0, list.Length);
}
```


## 4. Level 1: Pure Function Testing

The fastest, most reliable tests are pure math and logic tests. They require no `EntityManager`, no chunks, and no ECS overhead.

If you have complex logic (e.g., trajectory calculation, damage mitigation formulas), extract it from the Job into a `static` method and test it directly.

```csharp
// The Logic
public static class CombatMath
{
    public static float CalculateMitigatedDamage(float incomingDamage, float armor)
    {
        return math.max(1f, incomingDamage * (100f / (100f + armor)));
    }
}

// The Test
public class CombatMathTests
{
    [TestCase(100f, 0f, ExpectedResult = 100f)]
    [TestCase(100f, 100f, ExpectedResult = 50f)]
    [TestCase(100f, 300f, ExpectedResult = 25f)]
    [TestCase(5f, 1000f, ExpectedResult = 1f)]
    public float CalculateMitigatedDamage_ReturnsCorrectValues(float dmg, float armor)
    {
        return CombatMath.CalculateMitigatedDamage(dmg, armor);
    }
}
```


## 5. Level 2: Job Testing in Isolation

Testing a Burst-compiled job ensures that your parallel processing works correctly without the overhead of spinning up an entire System.

Use `.Run()` instead of `.Schedule()` to execute the job synchronously on the main thread for the test.

```csharp
[Test]
[TestLeakDetection]
public void ProcessDamageJob_WhenHealthDepleted_SetsDeadToTrue()
{
    // 1. Arrange: Create pure data arrays
    var healths = new NativeArray<Health>(1, Allocator.TempJob);
    healths[0] = new Health { Current = 10f, Max = 100f };
    
    var damageRequests = new NativeArray<DamageEvent>(1, Allocator.TempJob);
    damageRequests[0] = new DamageEvent { Amount = 50f }; // Fatal
    
    var isDeadFlags = new NativeArray<bool>(1, Allocator.TempJob);
    isDeadFlags[0] = false;

    // 2. Act: Execute the job directly
    var job = new ProcessDamageJob
    {
        Healths = healths,
        Damages = damageRequests,
        IsDead = isDeadFlags
    };
    
    job.Run(1); // Run for 1 iteration synchronously

    // 3. Assert
    Assert.AreEqual(-40f, healths[0].Current);
    Assert.IsTrue(isDeadFlags[0]);

    // 4. Cleanup (Caught by [TestLeakDetection] if forgotten)
    healths.Dispose();
    damageRequests.Dispose();
    isDeadFlags.Dispose();
}
```


## 6. Level 3: System Orchestration Testing

This is the most common ECS test. You create a controlled `World`, spawn entities, run a specific `ISystem`, and verify the resulting component states.

### 6.1 Testing a standard ISystem

```csharp
public class ApplyPoisonSystemTests : ECSTestsFixture
{
    [Test]
    [TestLeakDetection]
    public void ApplyPoisonSystem_ReducesHealth_OverTime()
    {
        // 1. Arrange: Setup the World and Entities
        var entity = Manager.CreateEntity(typeof(Health), typeof(PoisonEffect));
        
        Manager.SetComponentData(entity, new Health { Current = 100f });
        Manager.SetComponentData(entity, new PoisonEffect { DamagePerTick = 5f });

        // Get the system under test
        var system = World.GetOrCreateSystem<ApplyPoisonSystem>();
        
        // Mock DeltaTime using BovineLabs' UpdateWorldTimeSystem replacement or manually
        WorldUnmanaged.Time = new Unity.Core.TimeData(elapsedTime: 1f, deltaTime: 1f);

        // 2. Act: Update the system
        system.Update(WorldUnmanaged);
        
        // CRITICAL: You must complete jobs before asserting!
        Manager.CompleteAllTrackedJobs();

        // 3. Assert
        var health = Manager.GetComponentData<Health>(entity);
        Assert.AreEqual(95f, health.Current);
    }
}
```

### 6.2 Testing Systems with EntityCommandBuffers
If your system uses an ECB (like destroying an entity), you must manually force the ECB to playback in your test, OR update the ECB system group.

```csharp
[Test]
[TestLeakDetection]
public void DeathSystem_WhenHealthZero_DestroysEntity()
{
    // Arrange
    var entity = Manager.CreateEntity(typeof(Health));
    Manager.SetComponentData(entity, new Health { Current = 0f }); // Dead

    // Act
    World.GetOrCreateSystem<DeathSystem>().Update(WorldUnmanaged);
    Manager.CompleteAllTrackedJobs();
    
    // The DeathSystem scheduled ECB commands, but hasn't played them back yet!
    Assert.IsTrue(Manager.Exists(entity), "Entity should still exist before ECB playback");

    // Force the ECB system to playback
    World.GetOrCreateSystem<EndSimulationEntityCommandBufferSystem>().Update(WorldUnmanaged);
    Manager.CompleteAllTrackedJobs();

    // Assert
    Assert.IsFalse(Manager.Exists(entity), "Entity should be destroyed after ECB playback");
}
```


## 7. Advanced: Testing DynamicHashMaps & Custom Collections

`BovineLabs.Core` introduces `DynamicHashMap`, which lives inside an Entity's `DynamicBuffer`. Testing this requires setting up the buffer and initializing the helper.

```csharp
[Test]
[TestLeakDetection]
public void InventorySystem_AddsItemToDynamicHashMap()
{
    // 1. Arrange
    var entity = Manager.CreateEntity(typeof(InventoryMap)); // IDynamicHashMap<int, int>
    
    // Initialize the DynamicHashMap using BovineLabs extensions
    var buffer = Manager.GetBuffer<InventoryMap>(entity);
    buffer.InitializeHashMap<InventoryMap, int, int>(capacity: 16);

    // 2. Act
    var system = World.GetOrCreateSystem<InventorySystem>();
    system.Update(WorldUnmanaged);
    Manager.CompleteAllTrackedJobs();

    // 3. Assert
    // Get the map back out to verify
    var map = Manager.GetBuffer<InventoryMap>(entity).AsMap();
    
    Assert.AreEqual(1, map.Count);
    Assert.IsTrue(map.TryGetValue(itemId: 101, out var itemCount));
    Assert.AreEqual(5, itemCount);
}
```


## 8. Advanced: Testing Facets & IEnableableComponents

### 8.1 Testing Facet Resolution
To ensure a `BovineLabs.FacetGenerator` facet resolves correctly, test it directly using the `Lookup` or `ResolvedChunk`.

```csharp
[Test]
[TestLeakDetection]
public void CombatFacet_TryGet_ResolvesCorrectly()
{
    // Arrange
    var entity = Manager.CreateEntity(typeof(Health), typeof(Invulnerable));
    Manager.SetComponentEnabled<Invulnerable>(entity, false);

    // Act: Create state and lookup manually for the test
    var state = World.Unmanaged.ResolveSystemStateRef(World.GetOrCreateSystem<CombatSystem>().SystemHandle);
    
    var lookup = new CombatFacet.Lookup();
    lookup.Create(ref state);
    lookup.Update(ref state); // MUST update to refresh cache

    // Assert
    Assert.IsTrue(lookup.TryGet(entity, out var facet), "Facet should resolve");
    Assert.IsFalse(facet.IsInvulnerable, "EnabledRefRO should correctly read disabled state");
}
```

### 8.2 Testing IEnableableComponent States
Do not use `HasComponent<T>` to check enableable state. Use `IsComponentEnabled<T>`.

```csharp
[Test]
[TestLeakDetection]
public void StunSystem_EnablesStunnedComponent()
{
    var entity = Manager.CreateEntity(typeof(Stunned), typeof(StunRequest));
    Manager.SetComponentEnabled<Stunned>(entity, false); // Explicitly disable

    World.GetOrCreateSystem<StunSystem>().Update(WorldUnmanaged);
    Manager.CompleteAllTrackedJobs();

    Assert.IsTrue(Manager.IsComponentEnabled<Stunned>(entity), "Stunned should be enabled");
    Assert.IsFalse(Manager.HasComponent<StunRequest>(entity), "Request should be consumed");
}
```


## 9. Advanced: Testing Blob Assets

Blob assets require unmanaged allocation. Ensure you use the `BlobBuilder` and dispose of the reference properly, or the `[TestLeakDetection]` will fail.

```csharp
[Test]
[TestLeakDetection]
public void PathfindingSystem_ReadsBlobAssetCorrectly()
{
    // Arrange: Build a blob asset in the test
    var builder = new BlobBuilder(Allocator.Temp);
    ref var root = ref builder.ConstructRoot<WaypointBlob>();
    var array = builder.Allocate(ref root.Points, 2);
    array[0] = new float3(0,0,0);
    array[1] = new float3(10,0,0);
    
    // Create reference and ensure we track it for disposal
    var blobRef = builder.CreateBlobAssetReference<WaypointBlob>(Allocator.Persistent);
    builder.Dispose();

    var entity = Manager.CreateEntity(typeof(WaypointPath));
    Manager.SetComponentData(entity, new WaypointPath { Blob = blobRef });

    // Act
    World.GetOrCreateSystem<PathfindingSystem>().Update(WorldUnmanaged);
    Manager.CompleteAllTrackedJobs();

    // Assert
    var pos = Manager.GetComponentData<LocalTransform>(entity).Position;
    AssertMath.AreApproximatelyEqual(new float3(10,0,0), pos, 0.01f);

    // Cleanup Blob (CRITICAL)
    blobRef.Dispose();
}
```
*Tip:* `ECSTestsFixture` provides a `this.BlobAssetStore` you can use if you are testing Baking systems.


## 10. Performance Benchmarking (The NASA Speed Standard)

We don't just test if code works; we test if it's fast enough and allocation-free. We use `Unity.PerformanceTesting`.

**Rule:** Core mathematical systems, tight loops, and collections MUST have performance benchmarks written.

```csharp
using Unity.PerformanceTesting;

public class MathExPerformanceTests : ECSTestsFixture
{
    [Test]
    [Performance] // Marks for the Performance Test Runner
    public void SIMD_Max_PerformanceTest()
    {
        int length = 100_000;
        var input = new NativeArray<float>(length, Allocator.Persistent);
        var random = new Unity.Mathematics.Random(1234);
        
        for (var i = 0; i < input.Length; i++)
            input[i] = random.NextFloat(-10000, 10000);

        // Measure.Method runs the lambda multiple times to calculate median/deviation
        Measure
            .Method(() => 
            {
                // This is the hot path being measured
                float result = BovineLabs.Core.Utility.mathex.max(input);
            })
            .WarmupCount(5)
            .MeasurementCount(20)
            .Run();

        input.Dispose();
    }
}
```

### 10.1 Profiling Job Performance
To benchmark a job, measure its `.Run()` method.

```csharp
[Test]
[Performance]
public void PathfindingJob_Performance()
{
    // Setup arrays...
    
    Measure.Method(() => 
    {
        new PathfindingJob { Nodes = nativeNodes }.Run(100_000);
    })
    .SampleGroup(new SampleGroup("Pathfinding 100k", SampleUnit.Microsecond))
    .WarmupCount(1)
    .MeasurementCount(10)
    .Run();
    
    // Dispose arrays...
}
```


## 11. Math Assertions & Helpers

Standard `Assert.AreEqual` fails violently with floating-point math and vectors.
**Rule:** Always use `BovineLabs.Testing.AssertMath`.

```csharp
using BovineLabs.Testing;

[Test]
public void Movement_UpdatesPosition()
{
    float3 expectedPos = new float3(1.0001f, 0, 5.0f);
    float3 actualPos = GetActualPosition();

    // ❌ BAD: Will fail due to floating point inaccuracies
    Assert.AreEqual(expectedPos, actualPos);

    // ✅ GOOD: Checks XYZ within the delta
    AssertMath.AreApproximatelyEqual(expectedPos, actualPos, 0.001f);
}

[Test]
public void Rotation_UpdatesCorrectly()
{
    quaternion expectedRot = quaternion.Euler(0, math.PI, 0);
    quaternion actualRot = GetActualRotation();

    // ✅ GOOD: Checks XYZW within the delta
    AssertMath.AreApproximatelyEqual(expectedRot, actualRot, 0.001f);
}
```


## 12. The Elite PR Testing Checklist

Before opening a Pull Request, your code must pass this tribunal:

1. [ ] **Coverage:** Are all new Systems covered by at least one `[Test]` in a `.Tests` assembly?
2. [ ] **Leak Free:** Are all system tests marked with `[TestLeakDetection]`?
3. [ ] **No Managed Mocks:** Are you instantiating real `NativeArray`s and `Entity` components instead of using mocking frameworks like Moq?
4. [ ] **Zero Complexity Validation:** Is the system's `OnUpdate` CC=1? If not, extract the logic into a pure function and write a test for that pure function.
5. [ ] **Completion:** Do your tests call `Manager.CompleteAllTrackedJobs()` before asserting data?
6. [ ] **ECB Syncing:** If your system uses `EntityCommandBuffer`, did you explicitly update the ECB system in the test to force playback?
7. [ ] **Performance (Hot Paths):** If you wrote a custom iterator, `IJobParallelHashMapDefer`, or core math extension, is there a `[Performance]` benchmark test attached?

> **"If it's not tested, it's broken. If it allocates in the hot path, it's rejected. If it leaks memory, it's reverted."**