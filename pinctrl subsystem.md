# Pinctrl Subsystem: Linux vs Barebox Design Differences

## Why barebox pinctrl is much smaller

The core philosophy difference: **Linux builds a full in-memory model of the DT; barebox treats the DT as the model and operates on it directly at use time.**

---

## 1. Lazy DT parsing (you already know this)

**Linux** parses DT in two distinct phases:

- **Phase 1 – probe time** (`rockchip_pinctrl_parse_dt()`): walks the *entire* DT subtree under the pinctrl node. Every `function` child and its `group` grandchildren are parsed. For each group, `rockchip,pins` is read, all `<bank pin mux config_phandle>` tuples extracted, and per-pin config phandles dereferenced via `pinconf_generic_parse_dt_config()`. Results land in `info->functions[]` and `info->groups[]` — in-memory copies of everything in the DT, allocated with `devm_kcalloc`.

- **Phase 2 – `pinctrl_get()` time** (`pinctrl_dt_to_map()`): reads the consumer's `pinctrl-N` properties, finds each config node, calls the driver's `dt_node_to_map()` which allocates `struct pinctrl_map[]` entries and stashes them in the global `pinctrl_maps` list. Then `add_setting()` iterates maps, resolves function/group names to numeric selectors, and builds `struct pinctrl_setting` objects stored per `struct pinctrl_state`.

**Barebox** does neither of these. `set_state(pdev, np)` is called only when a consumer actually applies a state. Inside, the driver reads `rockchip,pins` on the spot and writes to hardware immediately:

```c
// barebox: rockchip_pinctrl_set_state()
list = of_get_property(np, "rockchip,pins", &size);
for (i = 0; i < size; i += 4) {
    bank_num = be32_to_cpu(*list++);
    pin_num  = be32_to_cpu(*list++);
    func     = be32_to_cpu(*list++);
    phandle  = list++;
    np_config = of_find_node_by_phandle(be32_to_cpup(phandle));
    rockchip_set_mux(bank, pin_num, func);
    rockchip_set_pull(bank, pin_num, parse_bias_config(np_config));
    ...
}
```

No pre-parsed groups, no pre-parsed configs, no intermediate structures.

---

## 2. No map table / no intermediate representation layer

**Linux** has a multi-level translation pipeline:

```
DT node
  → dt_node_to_map()       → struct pinctrl_map[]      (registered in pinctrl_maps list)
  → add_setting()          → struct pinctrl_setting     (function/group selectors resolved)
  → pinctrl_commit_state() → pinmux_enable_setting()    (calls driver set_mux(selector))
                           → pinconf_apply_setting()    (calls driver pin_config_set())
```

Each stage allocates heap memory. The map table is a global list of `struct pinctrl_maps` nodes. Each entry in a map is named (carries `dev_name`, `ctrl_dev_name`, `name`). `add_setting()` resolves the string names to numeric group/function selectors by iterating over all registered groups/functions. All of this just to translate the DT into what the driver needs to do.

**Barebox** collapses the entire pipeline to a single call:

```
DT node → set_state(pdev, np) → hardware
```

No maps, no settings, no selectors. The driver callback gets the raw DT node and does everything itself.

---

## 3. No group/function abstraction layer

**Linux** mandates a rich ops interface for the driver:

```c
// Driver must implement all of these:
.get_groups_count()     // total number of groups
.get_group_name()       // name of group by index
.get_group_pins()       // pin list of group by index
.get_functions_count()
.get_function_name()
.get_function_groups()  // which groups a function supports
.set_mux()             // apply mux by (group_selector, func_selector)
.dt_node_to_map()      // parse DT node → map entries
.dt_free_map()
```

The core uses these to resolve names to selectors at `pinctrl_get()` time, then stores the selectors. At `pinctrl_select_state()` time it calls `set_mux(group_selector, func_selector)` — two levels of indirection.

**Barebox** has one callback:

```c
struct pinctrl_ops {
    int (*set_state)(struct pinctrl_device *pdev, struct device_node *np);
    int (*set_direction)(struct pinctrl_device *pdev, unsigned int pin, bool input);
    int (*get_direction)(struct pinctrl_device *pdev, unsigned int pin);
};
```

No group enumeration, no function enumeration, no name-to-selector resolution. The driver gets the DT node and decides what to do. The entire group/function abstraction and its supporting name-lookup machinery in the Linux core simply does not exist.

---

## 4. No pin descriptor registration

**Linux** registers every physical pin at probe time:

- Driver provides a static `struct pinctrl_pin_desc[]` array with all pin names and numbers.
- `pinctrl_register_pins()` allocates a `struct pin_desc` per pin and inserts each into a radix tree (`pctldev->pin_desc_tree`).
- Each `pin_desc` carries: name, mux_usecount, mux_owner string, mux_setting pointer, gpio_owner string, mux_lock mutex.
- Used for: name-to-number lookup, mux conflict detection, debugfs, GPIO integration.

**Barebox** has no pin descriptor registration. `struct pinctrl_device` has only `base` and `npins` (for GPIO range routing). No radix tree, no per-pin structs, no name table.

---

## 5. No use-count / ownership tracking

**Linux** tracks who owns each pin's mux:

- `pinmux_request_gpio()` and `pin_request()` set `pin_desc->mux_usecount` and `pin_desc->mux_owner`.
- Trying to mux a pin already claimed by another device fails explicitly.
- `pinctrl_gpio_request()` integrates with gpiolib to enforce the same ownership model for GPIO users.
- Releasing a state calls `pinmux_disable_setting()` which decrements usecounts and clears owners.

**Barebox** has none of this. `set_state()` writes the hardware unconditionally. In a bootloader this is fine — there is no concurrent use.

---

## 6. No `struct pinctrl` / `struct pinctrl_state` as rich objects

In **Linux**:

```c
struct pinctrl {           // per consumer device
    struct list_head states;   // list of pinctrl_state
    struct pinctrl_state *state; // current active state
    struct list_head dt_maps;
    struct kref users;
    ...
};

struct pinctrl_state {     // one per named state ("default", "sleep", ...)
    struct list_head settings; // list of pinctrl_setting
    const char *name;
};

struct pinctrl_setting {   // one per map entry
    enum pinctrl_map_type type;
    struct pinctrl_dev *pctldev;
    union { mux; configs; } data;
};
```

`pinctrl_get()` allocates a `struct pinctrl`, `create_state()` allocates `struct pinctrl_state` objects, `add_setting()` allocates `struct pinctrl_setting` objects. All are heap-allocated, reference-counted, and live for the lifetime of the device.

In **barebox**:

```c
struct pinctrl {
    struct device_node consumer_np;  // that's it
};

struct pinctrl_state {
    struct property prop;  // directly a DT property
};
```

A `pinctrl_state` IS a DT property. A `pinctrl` IS a node. No heap allocation for intermediate objects. `pinctrl_lookup_state()` just searches for `pinctrl-N` properties by matching `pinctrl-names`. `pinctrl_select_state()` reads the phandles from the property and calls `pinctrl_config_one()` directly.

---

## 7. No locking infrastructure

**Linux** has six locking points:
- `pinctrl_list_mutex` — global list of pinctrl consumer handles
- `pinctrl_maps_mutex` — global map table
- `pinctrldev_list_mutex` — global device list
- `pctldev->mutex` — per pin controller device
- `pin_desc->mux_lock` — per pin (for mux conflict detection)

**Barebox** has zero mutexes. Single-threaded bootloader, no need.

---

## 8. No sleep/PM state management

**Linux** has:
- `pctldev->hog_default` and `pctldev->hog_sleep` states
- `pinctrl_force_sleep()` / `pinctrl_force_default()` for system suspend
- Per-driver suspend/resume (e.g. `rockchip_pinctrl_suspend/resume`)
- The `"sleep"` pinctrl state applied by the PM core

**Barebox** has none of this. Boot, configure, boot.

---

## 9. No debugfs

**Linux**: `/sys/kernel/debug/pinctrl/<device>/` exposes pins, pinmux, pinconf, gpio-ranges, pinmux-functions, pinmux-pins for every registered controller.

**Barebox**: none.

---

## The full Linux flow vs the full barebox flow

### Linux — what happens from probe to hardware write

```
1. PROBE (driver)
   rockchip_pinctrl_parse_dt()
     → for every function child in DT:
         rockchip_pinctrl_parse_functions()
           → allocate func->groups[] array
           → for every group child:
               rockchip_pinctrl_parse_groups()
                 → read rockchip,pins property
                 → allocate grp->pins[], grp->data[] arrays
                 → for each pin tuple:
                     resolve bank, store pin number, mux func
                     pinconf_generic_parse_dt_config()  ← reads config phandle
                       → allocate configs[] array
   devm_pinctrl_register()
     → pinctrl_register_pins()   ← allocate pin_desc per pin → insert into radix tree
     → pinmux_check_ops()
     → pinconf_check_ops()
     → create_pinctrl()          ← hog the controller's own pins

2. CONSUMER PROBE (e.g. UART driver calls devm_pinctrl_get_select())
   pinctrl_get()
     → create_pinctrl()
         → pinctrl_dt_to_map()
             → for each pinctrl-N property on consumer node:
                 dt_to_map_one_config()
                   → find pctldev by walking DT parents
                   → rockchip_dt_node_to_map()    ← driver callback
                       → look up group by name in info->groups[]
                       → allocate pinctrl_map[] (MUX_GROUP + CONFIGS_PIN per pin)
                   → dt_remember_or_free_map()
                       → pinctrl_register_mappings()  ← add to global pinctrl_maps list
         → for_each_pin_map():   ← iterate global map table
             add_setting()
               → find or create pinctrl_state by name
               → allocate pinctrl_setting
               → pinmux_map_to_setting()
                   → resolve function name → func selector
                   → resolve group name   → group selector
                   → store selectors in setting->data.mux
               → append to state->settings list

3. SELECT STATE
   pinctrl_select_state()
     → pinctrl_commit_state()
         → for each setting in state->settings:
             pinmux_enable_setting()
               → pin_request()     ← check/set mux_owner, usecount
               → ops->set_mux(pctldev, group_selector, func_selector)
             pinconf_apply_setting()
               → ops->pin_config_set(pctldev, pin, configs[], num_configs)
```

### Barebox — what happens from probe to hardware write

```
1. PROBE (driver)
   rockchip_pinctrl_probe()
     → map grf/pmu regmaps
     → pinctrl_register(&info->pctl_dev)
         → list_add_tail into pinctrl_list
         → pinctrl_select_state_default(pdev->dev)  ← hog own pins if any

2. CONSUMER PROBE (device core calls of_pinctrl_select_state_default())
   of_pinctrl_select_state()
     → pinctrl_lookup_state()    ← find "default" in pinctrl-names, return prop ptr
     → pinctrl_select_state()
         → read phandles from pinctrl-0 property
         → for each phandle:
             np_config = of_find_node_by_phandle(phandle)
             pinctrl_config_one()
               → walk DT parents to find controller node
               → of_device_ensure_probed()  ← probe controller if not yet probed
               → find pdev in pinctrl_list
               → pdev->ops->set_state(pdev, np_config)
                   → rockchip_pinctrl_set_state()
                       → read rockchip,pins from np_config on the fly
                       → for each <bank pin mux config_phandle>:
                           rockchip_set_mux()    ← write register NOW
                           rockchip_set_pull()   ← write register NOW
```

---

## Summary table

| Feature                       | Linux                                           | Barebox                   |
| ----------------------------- | ----------------------------------------------- | ------------------------- |
| DT parsed at probe            | Yes — full subtree into groups/functions arrays | No                        |
| DT parsed at pinctrl_get()    | Yes — into map table + settings list            | No                        |
| DT parsed at state apply      | No — uses pre-built settings                    | Yes — on the fly          |
| Intermediate map table        | Yes — global `pinctrl_maps` list                | No                        |
| Intermediate settings objects | Yes — per state, per map entry                  | No                        |
| Pin descriptor radix tree     | Yes — one `pin_desc` per physical pin           | No                        |
| Group/function abstraction    | Yes — name→selector resolution                  | No                        |
| Mux ownership tracking        | Yes — usecount + owner string per pin           | No                        |
| Locking                       | Multiple mutexes                                | None                      |
| PM / sleep states             | Yes                                             | No                        |
| Debugfs                       | Yes                                             | No                        |
| Driver ops required           | ~8 callbacks                                    | 1 callback (`set_state`)  |
| `struct pinctrl` size         | Full object with states list, dt_maps, kref     | Just `device_node`        |
| `struct pinctrl_state` size   | Object with settings list                       | Just `property` (DT prop) |
____
```
  What barebox eliminated, in detail

  1. No intermediate map table

  Linux has a 3-stage pipeline: DT → pinctrl_map[] (global list) → pinctrl_setting[] per state → hardware. Each stage allocates heap memory and does name resolution. Barebox collapses this
  to: DT → hardware, directly in set_state().

  2. No probe-time DT parsing in the driver

  Linux rockchip's rockchip_pinctrl_parse_dt() at probe time walks the entire DT subtree — every function, every group, every rockchip,pins tuple, every config phandle — and builds
  info->functions[] and info->groups[] arrays with all configs pre-extracted. This happens regardless of whether those pin groups are ever used. Barebox reads only what's needed, only when
  it's needed.

  3. No group/function abstraction layer

  Linux requires the driver to implement ~8 callbacks (get_groups_count, get_group_name, get_group_pins, get_functions_count, get_function_name, get_function_groups, set_mux, dt_node_to_map).
   The core uses these to resolve string names to numeric selectors at pinctrl_get() time, stores the selectors, then calls set_mux(group_selector, func_selector) later. Barebox has one
  callback: set_state(pdev, np) — no selector indirection at all.

  4. No pin descriptor radix tree

  Linux allocates a struct pin_desc for every physical pin at probe time and inserts them into a radix tree. Each carries name, mux_usecount, mux_owner, mux_setting pointer, gpio_owner,
  mux_lock. Barebox has none of this.

  5. No use-count / ownership tracking

  Linux prevents two consumers from claiming the same pin's mux. Barebox just writes unconditionally — in a bootloader there's no concurrent use to protect against.

  6. struct pinctrl and struct pinctrl_state are trivial

  In barebox, struct pinctrl is just a device_node, and struct pinctrl_state is just a property (the raw DT property). In Linux they are full objects with lists, reference counts, and
  heap-allocated sub-objects.

  7. No locking, no PM/sleep states, no debugfs
```