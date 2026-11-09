# Mapmodes Framework

The Mapmodes Framework gives Antistasi Ultimate a common way to register and switch between map views. It adds a dropdown beside AU's Location Browser, where built-in features and extender addons can offer their own mapmodes with a name and optional icon. Only one provider is active at a time: selecting a mode deactivates the previous provider before activating the next, while Default Mapmode selects no provider overlay. Official modes appear first, with extender modes below a divider, and each player's choice is remembered locally. Creators register through a small client-side API and supply activation and cleanup callbacks. The framework handles registration, ordering, and selection; providers remain responsible for rendering, data processing, and interactions. It does not add a per-frame drawing dispatcher or poll providers for updates.

**TO BE REMOVED | DEV TEST PROVIDER MAPMODE:** 
Before main-mod integration, remove the `functions/placeholder_mapmode` folder, its preparation include, and the remaining marked hooks and stringtable key.

## Register Your Mode

For an extender addon, declare the AU framework's config dependency. This refers to its internal CfgPatches identity and ensures initialization order; it does not require players to install another mod or PBO:

```cpp
requiredAddons[] = {"A3A_ultimate", "A3A_ultimate_mapmodes_framework"};
```

Register once during client postInit, after your functions and any data they require are ready:

```sqf
if (!hasInterface) exitWith {};

[
    "MyAddon_overview",
    localize "STR_MyAddon_Overview",
    { [] call MyAddon_fnc_activateMapmode; },
    { [] call MyAddon_fnc_deactivateMapmode; },
    100,
    "\a3\ui_f\data\igui\cfg\simpleTasks\types\map_ca.paa"
] call A3A_fnc_mapmodesRegister;
```

Replace the example ID, localized label, callbacks, and icon with your own values. Use a unique namespaced ID containing an underscore and define the label in your addon's stringtable. The example uses full function names so it does not depend on component macros. If your addon uses AU macros, you can use its `FUNC` and `LSTRING` equivalents instead. Duplicate IDs are rejected.

The numeric order is optional (default 100); lower values sort first within the official or extender group, followed by case-insensitive label and ID. An extender cannot move above the official group by using a negative order. Official classification comes from AU configuration, not registration arguments.

The sixth argument is an optional icon texture path, displayed before the mode name. Use an Arma texture or an asset supplied by your addon. Omit it or use `""` for no icon. To specify an icon while keeping the default order, pass `100` as the fifth argument. Existing four- and five-argument registrations continue to work.

`MyAddon_fnc_activateMapmode` and `MyAddon_fnc_deactivateMapmode` are **your functions**, not framework helpers. Activation enables your mode's rendering and interactions. Deactivation must remove its owned handlers and controls and stop its workers. Save handler IDs, clean up only what your mode owns, and keep both callbacks safe to call again. The framework does not implement any rendering or data processing for you.

Both callbacks receive no arguments and run locally, unscheduled. They must return promptly and cannot sleep. The framework never calls them per frame. Selecting a different mode calls the previous provider's disable function before the new provider's enable function. Selecting the current mode is a no-op.

The selected mode remains selected while the map is closed. Your provider must handle map openings and recreated displays itself. The framework does not pass a map control or re-run enable merely because the map was reopened. Providers own any ongoing data processing, interaction, or rendering costs.

## Public Functions

| Function   | Call                                                                           | Returns                                                      |
| ---------- | ------------------------------------------------------------------------------ | ------------------------------------------------------------ |
| Register   | `[id, label, activate, deactivate, order, icon] call A3A_fnc_mapmodesRegister` | Boolean; order and icon can be omitted                       |
| Unregister | `[id] call A3A_fnc_mapmodesUnregister`                                         | Boolean; deactivates an active mode first                    |
| Get modes  | `[] call A3A_fnc_mapmodesGetModes`                                             | Detached `[[id, label, official, order, icon], ...]` records |
| Get active | `[] call A3A_fnc_mapmodesGetActive`                                            | ID, or `""` for Default Mapmode                              |
| Select     | `[id] call A3A_fnc_mapmodesSelect`                                             | Boolean; `""` selects Default Mapmode                        |

For example, remove a mode when unloading its feature:

```sqf
["MyAddon_overview"] call A3A_fnc_mapmodesUnregister;
```

Calls are local, not broadcast. Dedicated servers and headless clients do not register modes. Malformed input, duplicate IDs, unknown IDs, and nested mutations from a lifecycle callback return false without replacing registry state. Getters return empty values on machines without an interface. The metadata list excludes Default Mapmode, which is an implicit built-in choice. The icon field is appended to metadata; existing ID, label, official, and order positions are unchanged.