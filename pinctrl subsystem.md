# Pinctrl Subsystem: Linux vs Barebox Design Differences

---

## Part 1 — Big Picture: All the Keywords Explained

### The hardware reality first

Imagine an SoC with 100 physical pads on the chip package. Each pad can be:
- A UART TX pin
- A SPI CLK pin
- A GPIO
- Pull-up or pull-down
- Drive strength high or low

There is a **pinctrl hardware block** inside the SoC — a set of registers. You write to those registers and the pad changes behavior. That's all pinctrl ultimately does: **write some registers**.

We'll use one running example throughout: a UART peripheral that needs two pads (TX and RX).

---

### Controller vs Consumer

**Controller** = the pinctrl hardware block + the driver for it.
It owns the registers. It knows how to write them. There is usually one per SoC (sometimes a few for different power domains).

**Consumer** = any driver that *needs* its pins configured before it can work.
The UART driver is a consumer. It doesn't know how to write the pinctrl registers — that's not its job. It just says: "I need my pins set up in the default state." The UART driver *consumes* the pinctrl service.

```
┌─────────────────────────────────────┐
│  SoC                                │
│                                     │
│  ┌──────────────┐  ┌─────────────┐  │
│  │ pinctrl HW   │  │  UART HW    │  │
│  │ (registers)  │  │             │  │
│  └──────┬───────┘  └──────┬──────┘  │
│         │ controls        │ uses    │
│  ┌──────┴───────┐  ┌──────┴──────┐  │
│  │ pinctrl      │◄─│ UART        │  │
│  │ driver       │  │ driver      │  │
│  │ (controller) │  │ (consumer)  │  │
│  └──────────────┘  └─────────────┘  │
└─────────────────────────────────────┘
```

In DT, the consumer has a property pointing to its pin configuration:
```dts
/* Consumer: the UART node */
uart0: uart@10000 {
    pinctrl-names = "default";
    pinctrl-0 = <&uart0_pins>;   /* ← points to config inside the controller */
};

/* Controller: the pinctrl node owns uart0_pins */
pinctrl: pinctrl@20000 {
    uart0_pins: uart0-pins {
        rockchip,pins = <1 10 2 &pcfg_pull_up>,   /* bank1, pin10, mux2, pull-up */
                        <1 11 2 &pcfg_pull_up>;    /* bank1, pin11, mux2, pull-up */
    };
};
```

---

### State

A **state** is a named configuration. The most common state is `"default"` — what the pins should be when the device is active. Another common state is `"sleep"` — what the pins should be when the system suspends (maybe drive them low to save power).

The consumer says: "I want state *default*" or "I want state *sleep*."

States come from `pinctrl-names` in DT:
```dts
pinctrl-names = "default", "sleep";
pinctrl-0 = <&uart0_default_pins>;   /* state index 0 = "default" */
pinctrl-1 = <&uart0_sleep_pins>;     /* state index 1 = "sleep"   */
```

---

### Function and Group

These two are paired. They describe **what a set of pins does**.

**Function** = a named peripheral function. Example: `"uart0"`, `"spi1"`, `"i2c2"`. This is what you're muxing *to*. The IOMUX register field gets set to the value that connects the pad to the UART block inside the SoC.

**Group** = the specific set of pins that implement a function. Example: function `"uart0"` might have two groups: `"uart0-default"` (TX+RX on pins 10,11) and `"uart0-alt"` (TX+RX on pins 20,21 — alternate routing on a different physical location on the package).

Together they answer: **which pins (group) are being set to which mux mode (function).**

```
Function: "uart0"
  ├── Group: "uart0-default"  → pins [10, 11], mux value = 2
  └── Group: "uart0-alt"      → pins [20, 21], mux value = 2
```

Linux needs this abstraction because the core has to manage it generically across hundreds of SoC drivers. The core asks: "give me all groups for function uart0" and picks the right one based on what the DT says.

---

### Selector

A **selector** is just a numeric index (integer ID) for a group or function, used internally by Linux.

Linux resolves names to selectors at `pinctrl_get()` time so that at `pinctrl_select_state()` time it doesn't have to do string comparisons — it just uses the number.

```
"uart0"           → function selector = 3
"uart0-default"   → group selector    = 7
```

Then `set_mux(pctldev, group_selector=7, func_selector=3)` is what actually goes to the driver.

This is purely a Linux optimization — resolve strings once, use IDs thereafter. Barebox skips this entirely.

---

### Map and Map Table

A **map entry** is one row in a translation table that says:

> "Device X, in state Y, needs controller Z to configure group G as function F"

Or for pin config (pull-up etc):

> "Device X, in state Y, needs controller Z to set configs C on pin P"

In code, `struct pinctrl_map`:
```c
struct pinctrl_map {
    const char *dev_name;      /* "uart0" — the consumer */
    const char *name;          /* "default" — the state name */
    const char *ctrl_dev_name; /* "pinctrl@20000" — the controller */
    enum pinctrl_map_type type;/* MUX_GROUP or CONFIGS_PIN */
    union {
        struct { const char *function; const char *group; } mux;
        struct { const char *group_or_pin; unsigned long *configs; } configs;
    } data;
};
```

The **map table** (`pinctrl_maps`) is a global list of all these entries, for all consumers in the system.

Why does Linux need this? Because it was originally designed to also support non-DT platforms (board files) that register mappings statically at compile time. DT is just another way to populate this same table. So the DT gets parsed into maps, and maps get processed into settings. It's an extra translation hop that barebox skips entirely.

---

### Setting

A **setting** is a map entry *after* name resolution — groups and functions have been resolved to selectors, the controller pointer is known (no more strings). It's ready to apply to hardware.

`struct pinctrl_setting`:
```c
struct pinctrl_setting {
    enum pinctrl_map_type type;
    struct pinctrl_dev *pctldev;  /* actual pointer to controller */
    union {
        struct { unsigned int group; unsigned int func; } mux;      /* selectors */
        struct { unsigned int group_or_pin; unsigned long *configs; } configs;
    } data;
};
```

Each `pinctrl_state` (the named state object) holds a list of settings. When you call `pinctrl_select_state()`, it iterates the settings and calls the driver for each one.

---

### Pin Descriptor (`pin_desc`)

In Linux, every physical pin on the controller has a `struct pin_desc` object, stored in a radix tree. It acts as a **registry of who owns each pin**.

```c
struct pin_desc {
    const char *name;            /* "PIN10" or "uart-tx" */
    unsigned int mux_usecount;   /* how many consumers have claimed this pin */
    const char *mux_owner;       /* which consumer claimed it: "uart0" */
    const char *gpio_owner;      /* if claimed as GPIO */
};
```

This exists because two different drivers could accidentally try to mux the same physical pin — one wants it as UART TX, another as SPI CLK. Linux catches this at `pinctrl_select_state()` time and returns an error. Without pin descriptors there is nothing to check against.

---

### Use-count

The **use-count** (`mux_usecount`) on a `pin_desc` tracks how many active consumers have claimed that pin. When count goes from 0→1 the pin is "claimed". When it goes back to 0 the pin is "released" and can be given to someone else.

This solves: "UART driver called `pinctrl_get()`, started using pin 10. Then some other driver also tries to mux pin 10 as SPI CLK. Detect and reject."

In a bootloader (barebox) there is no multitasking, no concurrent drivers — impossible for two things to fight over a pin — so use-count is pointless.

---

### `dev_name` / `ctrl_dev_name` / `name` in a map entry

These three strings are how the Linux map table ties things together **by name** (not pointers, because devices may not be probed yet when the map is registered):

- `dev_name` — name of the **consumer** device, e.g. `"10000.uart"`. This map entry is *for* this device.
- `ctrl_dev_name` — name of the **controller** device, e.g. `"20000.pinctrl"`. This map entry is *served by* this controller.
- `name` — the **state name** this map entry belongs to, e.g. `"default"`.

When the consumer calls `pinctrl_get()`, the core scans the global map table looking for entries where `dev_name` matches this consumer. It then looks up `ctrl_dev_name` to get the actual controller pointer. These strings are the glue before the controller pointer is resolved.

---

### Putting it all together: the Linux journey for one UART pin

```
1. Boot: rockchip_pinctrl_probe()
   ├── Parse entire DT subtree
   │   └── For each group node (uart0-pins, spi1-pins, ...):
   │       ├── Read rockchip,pins property
   │       ├── Allocate grp->pins[], grp->data[] arrays
   │       └── Extract config phandles → allocate configs[] array
   ├── Register all pins → allocate pin_desc per pin → insert into radix tree
   └── Register functions/groups with core

2. UART driver probe: devm_pinctrl_get_select(dev, "default")
   │
   ├── pinctrl_get(uart_dev)
   │   ├── pinctrl_dt_to_map()          ← read consumer's pinctrl-0 property
   │   │   ├── find config node (uart0_pins) via phandle
   │   │   ├── rockchip_dt_node_to_map() ← driver callback
   │   │   │   ├── look up group "uart0-pins" in pre-parsed info->groups[]
   │   │   │   ├── alloc pinctrl_map[0]: type=MUX_GROUP, func="uart0", group="uart0-pins"
   │   │   │   └── alloc pinctrl_map[1,2]: type=CONFIGS_PIN, pin=10, configs=[pull-up]
   │   │   └── register maps into global pinctrl_maps list
   │   └── add_setting() for each map:
   │       ├── resolve "uart0"       → func selector = 3
   │       ├── resolve "uart0-pins"  → group selector = 7
   │       └── store in pinctrl_setting (selector numbers + controller pointer)
   │
   └── pinctrl_select_state(p, "default")
       └── pinctrl_commit_state()
           ├── for each MUX_GROUP setting:
           │   ├── pin_request(pin=10)  ← check usecount, set owner="uart0"
           │   └── ops->set_mux(pctldev, group=7, func=3)  ← WRITE REGISTER
           └── for each CONFIGS_PIN setting:
               └── ops->pin_config_set(pctldev, pin=10, [PULL_UP])  ← WRITE REGISTER
```

### The barebox journey for the same UART pin

```
1. Boot: rockchip_pinctrl_probe()
   └── map registers, add to pinctrl_list   ← that's ALL

2. UART driver probe: of_pinctrl_select_state_default(np)
   └── pinctrl_select_state()
       └── read phandle from pinctrl-0 property
           └── pinctrl_config_one(uart0_pins node)
               └── find controller by walking DT parents
                   └── ops->set_state(pdev, uart0_pins_node)
                       └── rockchip_pinctrl_set_state()
                           ├── read rockchip,pins from node on the fly
                           ├── rockchip_set_mux(bank1, pin10, mux2)  ← WRITE REGISTER
                           └── rockchip_set_pull(bank1, pin10, PULL_UP)  ← WRITE REGISTER
```

---

### Keyword reference card

| Keyword | What it is in one line |
|---|---|
| **Controller** | The pinctrl HW block + its driver. Owns the registers. |
| **Consumer** | Any driver that needs its pins set up before it can work (UART, SPI, I2C...). |
| **State** | A named pin configuration: `"default"`, `"sleep"`, `"idle"`. |
| **Function** | A mux mode name: `"uart0"`, `"spi1"`. What the pin is being muxed *to*. |
| **Group** | The set of physical pins that together implement a function. Paired with Function. |
| **Selector** | Integer ID for a group or function. Linux resolves name→ID once at get time. |
| **Map entry** | One row in a table: consumer X, state Y, controller Z, mux group G to function F. |
| **Map table** | Global list of all map entries for all consumers in the system. |
| **Setting** | A resolved map entry: controller pointer + numeric selectors. Ready to apply. |
| **Pin descriptor** | Per-physical-pin object in a radix tree. Tracks name, owner, use-count. |
| **Use-count** | How many consumers currently have a pin's mux claimed. Prevents conflicts. |
| **dev_name** | String name of the consumer in a map entry (before the pointer is resolved). |
| **ctrl_dev_name** | String name of the controller in a map entry (before the pointer is resolved). |
| **name** (in map) | The state name (`"default"`, `"sleep"`) this map entry belongs to. |
| **Hog** | When the controller claims some of its own pins at probe time (self-consumer). |

---

## Part 2 — Why barebox is smaller: Design Differences

The core philosophy difference: **Linux builds a full in-memory model of the DT; barebox treats the DT as the model and operates on it directly at use time.**

---

### 1. Lazy DT parsing (you already know this)

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

### 2. No map table / no intermediate representation layer

**Linux** has a multi-level translation pipeline:

```
DT node
  → dt_node_to_map()       → struct pinctrl_map[]      (registered in pinctrl_maps list)
  → add_setting()          → struct pinctrl_setting     (function/group selectors resolved)
  → pinctrl_commit_state() → pinmux_enable_setting()    (calls driver set_mux(selector))
                           → pinconf_apply_setting()    (calls driver pin_config_set())
```

Each stage allocates heap memory. The map table is a global list of `struct pinctrl_maps` nodes. Each entry in a map carries `dev_name`, `ctrl_dev_name`, `name` strings. `add_setting()` resolves the string names to numeric group/function selectors. All of this just to translate the DT into what the driver needs to do.

**Barebox** collapses the entire pipeline to a single call:

```
DT node → set_state(pdev, np) → hardware
```

No maps, no settings, no selectors. The driver callback gets the raw DT node and does everything itself.

---

### 3. No group/function abstraction layer

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

### 4. No pin descriptor registration

**Linux** registers every physical pin at probe time:

- Driver provides a static `struct pinctrl_pin_desc[]` array with all pin names and numbers.
- `pinctrl_register_pins()` allocates a `struct pin_desc` per pin and inserts each into a radix tree (`pctldev->pin_desc_tree`).
- Each `pin_desc` carries: name, mux_usecount, mux_owner string, mux_setting pointer, gpio_owner string, mux_lock mutex.
- Used for: name-to-number lookup, mux conflict detection, debugfs, GPIO integration.

**Barebox** has no pin descriptor registration. `struct pinctrl_device` has only `base` and `npins` (for GPIO range routing). No radix tree, no per-pin structs, no name table.

---

### 5. No use-count / ownership tracking

**Linux** tracks who owns each pin's mux:

- `pinmux_request_gpio()` and `pin_request()` set `pin_desc->mux_usecount` and `pin_desc->mux_owner`.
- Trying to mux a pin already claimed by another device fails explicitly.
- `pinctrl_gpio_request()` integrates with gpiolib to enforce the same ownership model for GPIO users.
- Releasing a state calls `pinmux_disable_setting()` which decrements usecounts and clears owners.

**Barebox** has none of this. `set_state()` writes the hardware unconditionally. In a bootloader this is fine — there is no concurrent use.

---

### 6. No `struct pinctrl` / `struct pinctrl_state` as rich objects

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

### 7. No locking infrastructure

**Linux** has five locking points:
- `pinctrl_list_mutex` — global list of pinctrl consumer handles
- `pinctrl_maps_mutex` — global map table
- `pinctrldev_list_mutex` — global device list
- `pctldev->mutex` — per pin controller device
- `pin_desc->mux_lock` — per pin (for mux conflict detection)

**Barebox** has zero mutexes. Single-threaded bootloader, no need.

---

### 8. No sleep/PM state management

**Linux** has:
- `pctldev->hog_default` and `pctldev->hog_sleep` states
- `pinctrl_force_sleep()` / `pinctrl_force_default()` for system suspend
- Per-driver suspend/resume (e.g. `rockchip_pinctrl_suspend/resume`)
- The `"sleep"` pinctrl state applied by the PM core

**Barebox** has none of this. Boot, configure, done.

---

### 9. No debugfs

**Linux**: `/sys/kernel/debug/pinctrl/<device>/` exposes pins, pinmux, pinconf, gpio-ranges, pinmux-functions, pinmux-pins for every registered controller.

**Barebox**: none.

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

---

## Part 3 — What does `set_state` actually do? How does it map to Linux?

### First: one pin or a group?

`set_state` receives a **DT node** — in the rockchip case that node holds a property
called `rockchip,pins` which is a **list of tuples**, one per pin.

```dts
uart0_pins: uart0-pins {
    rockchip,pins =
        <1  10  2  &pcfg_pull_up>,    /* bank=1, pin=10, mux=2, config=pull-up */
        <1  11  2  &pcfg_pull_up>;    /* bank=1, pin=11, mux=2, config=pull-up */
};
```

So `set_state` is called **once per group node**, but inside it loops over **every pin
in that group**. It handles a group, not a single pin.

---

### What `set_state` does per pin, step by step

For each `<bank  pin  mux  config_phandle>` tuple in the `rockchip,pins` list,
barebox's `rockchip_pinctrl_set_state` does four things:

```
1. rockchip_set_mux(bank, pin_num, func)
       → writes the IOMUX register: "connect this pad to peripheral X"

2. rockchip_set_pull(bank, pin_num, parse_bias_config(np_config))
       → writes the pull register: pull-up / pull-down / floating

3. rockchip_set_gpio(bank, pin_num, parse_gpio_direction(np_config))
       → if the config node says input-enable or output-high/low,
         set the GPIO direction register

4. rockchip_set_drive_perpin(bank, pin_num, drive_strength)   [if present]
       → writes drive strength register

5. rockchip_set_schmitt(bank, pin_num, ...)                   [if present]
       → writes schmitt-trigger register
```

That is the complete job: **for every pin in the group, set mux + pull + direction +
drive + schmitt directly in hardware registers.**

---

### What is the Linux equivalent of `set_state`?

Linux splits this work across **two separate driver callbacks**, called from two
separate places in the core:

#### Callback 1: `set_mux` — handles only the IOMUX part

Called from `pinmux_enable_setting()` inside `pinctrl_commit_state()`.

```c
/* Linux core calls this: */
ops->set_mux(pctldev, func_selector, group_selector);

/* Rockchip's implementation: */
static int rockchip_pmx_set(struct pinctrl_dev *pctldev,
                            unsigned selector,   /* func selector */
                            unsigned group)      /* group selector */
{
    const unsigned int *pins = info->groups[group].pins;   /* pin list from probe-time parse */
    const struct rockchip_pin_config *data = info->groups[group].data;

    for (cnt = 0; cnt < info->groups[group].npins; cnt++) {
        bank = pin_to_bank(info, pins[cnt]);
        rockchip_set_mux(bank, pins[cnt] - bank->pin_base, data[cnt].func);
        /*                                                  ↑
                                    func value was stored at probe time
                                    from parsing rockchip,pins in DT   */
    }
}
```

`set_mux` only sets the IOMUX register. Nothing else. No pull, no drive strength.
It also uses pre-parsed data (`info->groups[group]`) — it does not touch the DT here.

#### Callback 2: `pin_config_set` — handles pull, drive, schmitt per pin

Called from `pinconf_apply_setting()` inside `pinctrl_commit_state()`, separately
after all mux settings are applied.

```c
/* Linux core calls this once per pin: */
ops->pin_config_set(pctldev, pin_number, configs[], num_configs);

/* Rockchip's implementation: */
static int rockchip_pinconf_set(struct pinctrl_dev *pctldev,
                                unsigned int pin,
                                unsigned long *configs,
                                unsigned num_configs)
{
    for (i = 0; i < num_configs; i++) {
        param = pinconf_to_config_param(configs[i]);
        arg   = pinconf_to_config_argument(configs[i]);

        switch (param) {
        case PIN_CONFIG_BIAS_PULL_UP:
            rockchip_set_pull(bank, pin - bank->pin_base, param);
            break;
        case PIN_CONFIG_DRIVE_STRENGTH:
            rockchip_set_drive_perpin(bank, pin - bank->pin_base, arg);
            break;
        case PIN_CONFIG_INPUT_SCHMITT_ENABLE:
            rockchip_set_schmitt(bank, pin - bank->pin_base, 1);
            break;
        ...
        }
    }
}
```

`pin_config_set` only sets pull/drive/schmitt. Not the IOMUX. And it operates on
**one pin at a time** — the core calls it in a loop over all pins in the group.
The `configs[]` array it receives was built at probe time by
`pinconf_generic_parse_dt_config()` which already read and converted all the config
phandle properties into a packed `unsigned long` array.

---

### Side-by-side comparison for uart0_pins with 2 pins

```
DT:  uart0_pins { rockchip,pins = <1 10 2 &pcfg_pull_up>, <1 11 2 &pcfg_pull_up>; }
```

**Barebox** — one call to `set_state(pdev, uart0_pins_node)`:

```
set_state receives the uart0_pins node
  Loop iteration 1 (pin 10):
    read: bank=1, pin=10, func=2, config_phandle → np_config
    rockchip_set_mux(bank1, 10, 2)           ← IOMUX register
    rockchip_set_pull(bank1, 10, PULL_UP)    ← pull register
  Loop iteration 2 (pin 11):
    read: bank=1, pin=11, func=2, config_phandle → np_config
    rockchip_set_mux(bank1, 11, 2)           ← IOMUX register
    rockchip_set_pull(bank1, 11, PULL_UP)    ← pull register
```

**Linux** — three separate calls, each operating on pre-parsed data:

```
pinmux_enable_setting(setting)           ← setting has group_selector=7, func_selector=3
  get_group_pins(group=7) → [pin10, pin11]
  pin_request(pin10, "uart0")            ← usecount check
  pin_request(pin11, "uart0")            ← usecount check
  ops->set_mux(pctldev, func=3, group=7)
    rockchip_pmx_set():
      for pin10: rockchip_set_mux(bank1, 10, 2)   ← IOMUX register
      for pin11: rockchip_set_mux(bank1, 11, 2)   ← IOMUX register

pinconf_apply_setting(setting_for_pin10) ← separate setting, configs=[PULL_UP]
  ops->pin_config_set(pctldev, pin=10, [PULL_UP], 1)
    rockchip_pinconf_set():
      rockchip_set_pull(bank1, 10, PULL_UP)        ← pull register

pinconf_apply_setting(setting_for_pin11) ← separate setting, configs=[PULL_UP]
  ops->pin_config_set(pctldev, pin=11, [PULL_UP], 1)
    rockchip_pinconf_set():
      rockchip_set_pull(bank1, 11, PULL_UP)        ← pull register
```

The hardware register writes at the bottom are **identical**. The difference is purely
in how much scaffolding surrounds them.

---

### Where does the data come from in each case?

This is the other key difference — barebox reads from DT at call time, Linux reads
from pre-parsed arrays:

| Data needed | Barebox source | Linux source |
|---|---|---|
| IOMUX function value (`2`) | Read from `rockchip,pins` in DT **right now** | `info->groups[group].data[cnt].func` — parsed at probe time into an array |
| Pull-up/down | Read `pcfg_pull_up` phandle → node → property **right now** | `configs[]` array — built at probe time by `pinconf_generic_parse_dt_config()` |
| Drive strength | Read `drive-strength` from config node **right now** | Same `configs[]` array |
| Which pins are in the group | Read `rockchip,pins` count **right now** | `info->groups[group].pins[]` — allocated and filled at probe time |

---

### Summary: `set_state` = `set_mux` + `pin_config_set` + DT reading, all in one

```
barebox set_state(pdev, np)
  │
  ├── reads DT on the fly          ← Linux does this at probe time
  │
  ├── loops over all pins          ← Linux core does this in pinmux_enable_setting
  │                                   and pinconf_apply_setting
  │
  ├── rockchip_set_mux()           ← same as Linux's set_mux doing rockchip_set_mux()
  │
  └── rockchip_set_pull()          ← same as Linux's pin_config_set doing rockchip_set_pull()
      rockchip_set_drive_perpin()
      rockchip_set_schmitt()
```

The actual hardware-touching functions (`rockchip_set_mux`, `rockchip_set_pull`,
`rockchip_set_drive_perpin`, `rockchip_set_schmitt`) are **the same code** in both
barebox and Linux. Everything above them is what differs.

---

## Part 4 — What does "mux" actually mean?

### Mux is per-pin, not per-group

**Mux** = multiplexer. One physical pad on the chip package can be wired internally
to **multiple peripherals** inside the SoC. A small register field (typically 2-4 bits)
selects which one is actually connected at any given time.

```
                    ┌─── UART0 TX  (mux value = 1)
                    │
Physical pad ───────┼─── SPI1 CLK  (mux value = 2)
  (one pin)         │
                    ├─── I2C2 SDA  (mux value = 3)
                    │
                    └─── GPIO      (mux value = 0)
                              ↑
                    write this number into the IOMUX register
                    for this pad to pick one connection
```

Writing that value is the mux operation — **selecting which peripheral gets connected
to one physical pad**. It happens one pad at a time, each pad has its own register field.

---

### Group and Function are the software layer on top

When you configure a UART you need **two pads** (TX + RX). Those two pads together
are called a **group** — the named collection of pads that together serve one purpose.
Each pad still has its own mux register write, but you configure them together because
they belong to the same use case.

**Function** is the name of what the group is being muxed to: `"uart0"`, `"spi1"`, `"i2c2"`.

```
Function "uart0"
  └── Group "uart0-default"
        ├── pad 10 → write mux=2 → connects to UART0 TX internally
        └── pad 11 → write mux=2 → connects to UART0 RX internally
```

So the three terms relate like this:

| Term | Level | What it is |
|---|---|---|
| **Mux** | Hardware, per pad | The register write that connects one pad to one peripheral |
| **Group** | Software, collection | Named set of pads that together serve one use case |
| **Function** | Software, label | The name of the peripheral/use case the group is muxed to |

When Linux calls `set_mux(group_selector, func_selector)` it means: *for every pad
in this group, write the mux register value that connects it to this function.* It
loops over all pads in the group, but the actual register write is still one pad at
a time.

---

## Part 5 — What does "pad" mean?

Same thing as "pin" — just a different word for the same physical thing.

**Pad** is the hardware/silicon term: the metal contact point on the chip die that
connects to the package lead.

**Pin** is the board/package term: the lead on the chip package that you solder to
the PCB.

```
Silicon die                 Chip package
  ┌──────────┐                ┌──────────┐
  │  ████    │  bond wire     │          ├──── pin (leg on package)
  │  ████ ───┼────────────────┤  pad     │
  │ (circuit)│                │          ├──── pin
  └──────────┘                └──────────┘
         ↑                         ↑
       "pad"                     "pin"
   (on the die)            (on the package)
```

In pinctrl documentation and driver code the two words are used interchangeably.
When you see `pad` in a datasheet or driver comment it means the same physical thing
you are already thinking of as "pin." No difference in meaning for pinctrl purposes.

---

## Part 6 — Complete Guide: How to Use the Pinctrl Subsystem in DTS

This section covers everything you need to write and read pinctrl Device Tree
bindings, how all the pieces fit together, and how to define new things.

---

### 6.1 The DTS structure at a glance

Every pinctrl setup in DTS has two sides: the **controller** and the **consumers**.

```
┌────────────────────────────────────────────────────────────────────┐
│  Device Tree                                                       │
│                                                                    │
│  pinctrl: pinctrl@... {       ← controller node                   │
│      ...                                                           │
│      pcfg_pull_up: pcfg-pull-up { ← shared config node            │
│          bias-pull-up;                                             │
│      };                                                            │
│                                                                    │
│      uart0 {                  ← function grouping (optional)       │
│          uart0_xfer: uart0-xfer { ← pin group node (the state)    │
│              rockchip,pins = <...>;                                │
│          };                                                        │
│      };                                                            │
│  };                                                                │
│                                                                    │
│  uart0: serial@... {          ← consumer node                     │
│      pinctrl-names = "default";                                    │
│      pinctrl-0 = <&uart0_xfer>;  ← points to pin group node       │
│  };                                                                │
└────────────────────────────────────────────────────────────────────┘
```

---

### 6.2 The consumer side (any device that needs pins)

Every device that needs pin configuration uses these properties:

```dts
device_node {
    pinctrl-names = "default", "sleep";  /* names for each state */
    pinctrl-0 = <&state0_phandle>;       /* config for state "default" (index 0) */
    pinctrl-1 = <&state1_phandle>;       /* config for state "sleep"   (index 1) */
};
```

**Rules:**

- `pinctrl-0`, `pinctrl-1`, ... `pinctrl-N` — each is a list of phandles pointing
  to pin configuration nodes inside a pin controller.
- `pinctrl-names` — maps index 0 to a name, index 1 to a name, etc.
- The numbering must be contiguous starting from 0.
- A state can reference **multiple** phandles (to compose config from multiple nodes):
  ```dts
  pinctrl-0 = <&uart_tx_pins>, <&uart_rx_pins>;
  ```
- A state can be empty (for platforms that don't need pin config):
  ```dts
  pinctrl-0 = <>;
  ```

**Standard state names** (recognized by the kernel automatically):
- `"default"` — selected before device probe
- `"init"` — selected before probe, then `"default"` is selected after probe
- `"sleep"` — selected during system suspend
- `"idle"` — selected during runtime idle

The device core handles `"default"` automatically — you do NOT need driver code to
apply it. The driver only needs explicit code for switching states at runtime.

---

### 6.3 The controller side (pin configuration nodes)

The pin configuration nodes live inside the pin controller node. Their format
depends entirely on the specific SoC binding. There is no single universal format,
but there are common patterns.

---

#### 6.3.1 Rockchip format

Uses a vendor-specific `rockchip,pins` property. Each entry is a 4-integer tuple.

Include the header:
```dts
#include <dt-bindings/pinctrl/rockchip.h>
```

Pin name macros from `rockchip.h`:
```
RK_PA0 = 0    RK_PA7 = 7     (port A, pins 0-7)
RK_PB0 = 8    RK_PB7 = 15    (port B, pins 0-7)
RK_PC0 = 16   RK_PC7 = 23    (port C, pins 0-7)
RK_PD0 = 24   RK_PD7 = 31    (port D, pins 0-7)
RK_FUNC_GPIO = 0              (mux value for GPIO mode)
```

**Format: `<bank  pin  mux_function  &config_phandle>`**

```dts
pinctrl: pinctrl {
    compatible = "rockchip,rk3568-pinctrl";
    rockchip,grf = <&grf>;

    /* ---- Shared config nodes ---- */

    pcfg_pull_none: pcfg-pull-none {
        bias-disable;
    };

    pcfg_pull_up: pcfg-pull-up {
        bias-pull-up;
    };

    pcfg_pull_down: pcfg-pull-down {
        bias-pull-down;
    };

    pcfg_pull_up_drv_8ma: pcfg-pull-up-drv-8ma {
        bias-pull-up;
        drive-strength = <8>;
    };

    pcfg_output_high: pcfg-output-high {
        output-high;
    };

    /* ---- Function grouping nodes ---- */

    uart2 {
        uart2_xfer: uart2-xfer {
            rockchip,pins =
                /* bank  pin      mux  config */
                <2 RK_PB0 2 &pcfg_pull_up>,   /* UART2_TX on bank2 pin8, mux=2 */
                <2 RK_PB1 2 &pcfg_pull_up>;   /* UART2_RX on bank2 pin9, mux=2 */
        };

        uart2_rts_cts: uart2-rts-cts {
            rockchip,pins =
                <2 RK_PB2 2 &pcfg_pull_none>,
                <2 RK_PB3 2 &pcfg_pull_none>;
        };
    };

    i2c1 {
        i2c1_xfer: i2c1-xfer {
            rockchip,pins =
                <0 RK_PB3 1 &pcfg_pull_none>,
                <0 RK_PB4 1 &pcfg_pull_none>;
        };
    };

    sdmmc0 {
        sdmmc0_clk: sdmmc0-clk {
            rockchip,pins =
                <1 RK_PD6 1 &pcfg_pull_none_drv_8ma>;
        };
        sdmmc0_cmd: sdmmc0-cmd {
            rockchip,pins =
                <1 RK_PD7 1 &pcfg_pull_up_drv_8ma>;
        };
        sdmmc0_bus4: sdmmc0-bus4 {
            rockchip,pins =
                <1 RK_PD0 1 &pcfg_pull_up_drv_8ma>,
                <1 RK_PD1 1 &pcfg_pull_up_drv_8ma>,
                <1 RK_PD2 1 &pcfg_pull_up_drv_8ma>,
                <1 RK_PD3 1 &pcfg_pull_up_drv_8ma>;
        };
    };
};
```

**How to read `<2 RK_PB0 2 &pcfg_pull_up>`:**
- `2` — pin bank 2 (GPIO2)
- `RK_PB0` — pin 8 within that bank (port B, pin 0)
- `2` — mux function 2 (check the SoC TRM for what function 2 is on this pin)
- `&pcfg_pull_up` — apply pull-up config from the shared config node

**Consumer using it:**
```dts
&uart2 {
    pinctrl-names = "default";
    pinctrl-0 = <&uart2_xfer>;  /* just TX/RX */
    status = "okay";
};

/* Or with flow control: */
&uart2 {
    pinctrl-names = "default";
    pinctrl-0 = <&uart2_xfer>, <&uart2_rts_cts>;  /* TX/RX + RTS/CTS */
    status = "okay";
};

/* SD card with clk + cmd + 4 data lines: */
&sdmmc0 {
    pinctrl-names = "default";
    pinctrl-0 = <&sdmmc0_clk>, <&sdmmc0_cmd>, <&sdmmc0_bus4>;
    status = "okay";
};
```

---

#### 6.3.2 STM32 format

Uses a `pinmux` property where each entry encodes port + pin + alternate function
in a single integer via the `STM32_PINMUX()` macro.

Include the headers:
```dts
#include <dt-bindings/pinctrl/stm32-pinfunc.h>
```

The macro: `STM32_PINMUX(port, line, mode)`
- `port` — a character: `'A'`, `'B'`, `'C'`, ...
- `line` — pin number within the port: 0-15
- `mode` — `GPIO`, `AF0` through `AF15`, `ANALOG`

```dts
&pinctrl {
    usart2_pins: usart2-0 {
        pins1 {
            pinmux = <STM32_PINMUX('A', 2, AF7)>;  /* USART2_TX = PA2, AF7 */
            bias-disable;
            drive-push-pull;
            slew-rate = <0>;
        };
        pins2 {
            pinmux = <STM32_PINMUX('A', 3, AF7)>;  /* USART2_RX = PA3, AF7 */
            bias-disable;
        };
    };

    i2c1_pins: i2c1-0 {
        pins {
            pinmux = <STM32_PINMUX('B', 6, AF4)>,  /* I2C1_SCL = PB6, AF4 */
                     <STM32_PINMUX('B', 7, AF4)>;   /* I2C1_SDA = PB7, AF4 */
            bias-disable;
            drive-open-drain;
            slew-rate = <0>;
        };
    };
};
```

Notice how STM32 puts the mux and config together per sub-node, and the config
properties (bias-disable, drive-push-pull, slew-rate) sit directly alongside
the `pinmux` property.

**Consumer:**
```dts
&usart2 {
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&usart2_pins>;
    pinctrl-1 = <&usart2_sleep_pins>;
    status = "okay";
};
```

---

#### 6.3.3 i.MX / Freescale format

Uses `fsl,pins` with SoC-specific macros that encode the full register info
(mux_reg, conf_reg, input_reg, mux_val, input_val) as a single identifier,
followed by a raw config register value.

```dts
#include <dt-bindings/pinctrl/imx6qdl-pinctrl.h>

&iomuxc {
    usdhc4 {
        pinctrl_usdhc4: usdhc4grp {
            fsl,pins = <
                MX6QDL_PAD_SD4_CMD__SD4_CMD      0x17059
                MX6QDL_PAD_SD4_CLK__SD4_CLK      0x10059
                MX6QDL_PAD_SD4_DAT0__SD4_DATA0   0x17059
                MX6QDL_PAD_SD4_DAT1__SD4_DATA1   0x17059
                MX6QDL_PAD_SD4_DAT2__SD4_DATA2   0x17059
                MX6QDL_PAD_SD4_DAT3__SD4_DATA3   0x17059
            >;
        };
    };

    uart1 {
        pinctrl_uart1: uart1grp {
            fsl,pins = <
                MX6QDL_PAD_CSI0_DAT10__UART1_TX_DATA  0x1b0b1
                MX6QDL_PAD_CSI0_DAT11__UART1_RX_DATA  0x1b0b1
            >;
        };
    };
};
```

The macro name encodes: `PAD_NAME__FUNCTION_NAME`.
The hex value (`0x17059`) is written directly to the pad config register — its bits
control pull-up/down, hysteresis, drive strength, slew rate. You must read the SoC
reference manual to decode these bits.

**Consumer:**
```dts
&usdhc4 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_usdhc4>;
    status = "okay";
};
```

---

#### 6.3.4 Generic format (function + groups/pins)

Some controllers use the generic properties from the DT spec. This is the most
"textbook" format:

```dts
&pinctrl {
    state_default: default {
        uart0_pins {
            function = "uart0";
            groups = "uart0_grp";
        };
    };

    state_sleep: sleep {
        uart0_pins {
            function = "gpio";
            pins = "PIN10", "PIN11";
        };
    };
};
```

Generic properties you can use inside pin config nodes:
- `function` — mux function name (string)
- `groups` — group names (string array)
- `pins` — individual pin names (string array)
- `pinmux` — array of encoded pin+mux integers (uint32 array)

---

### 6.4 All available pin configuration properties

These are the standard properties you can use inside any pin configuration node.
Not all SoCs support all of them — check your binding.

| Property | Type | Description |
|---|---|---|
| `bias-disable` | boolean | Disable any pin bias |
| `bias-pull-up` | boolean or uint32 | Enable pull-up (optional: strength in Ohm) |
| `bias-pull-down` | boolean or uint32 | Enable pull-down (optional: strength in Ohm) |
| `bias-pull-pin-default` | boolean or uint32 | Use pin's default pull state |
| `bias-high-impedance` | boolean | High impedance / floating / tri-state |
| `bias-bus-hold` | boolean | Latch weakly |
| `drive-push-pull` | boolean | Drive actively high and low |
| `drive-open-drain` | boolean | Open drain output |
| `drive-open-source` | boolean | Open source output |
| `drive-strength` | uint32 | Max sink/source current in mA |
| `drive-strength-microamp` | uint32 | Max sink/source current in uA |
| `input-enable` | boolean | Enable input buffer |
| `input-disable` | boolean | Disable input buffer |
| `input-schmitt-enable` | boolean | Enable schmitt trigger |
| `input-schmitt-disable` | boolean | Disable schmitt trigger |
| `input-debounce` | uint32 | Debounce time in usec (0 = disable) |
| `output-low` | boolean | Output mode, drive low |
| `output-high` | boolean | Output mode, drive high |
| `output-enable` | boolean | Enable output buffer (without driving) |
| `output-disable` | boolean | Disable output buffer |
| `output-impedance-ohms` | uint32 | Output impedance in ohms |
| `slew-rate` | uint32 | Slew rate (meaning is SoC-specific) |
| `power-source` | uint32 | Select power supply (e.g., 1.8V vs 3.3V domain) |
| `low-power-enable` | boolean | Enable low power mode |
| `low-power-disable` | boolean | Disable low power mode |
| `sleep-hardware-state` | boolean | This is a sleep-related state |
| `skew-delay` | uint32 | Clock skew delay (number of double-inverters) |

---

### 6.5 How to define new things: step-by-step recipes

#### Recipe 1: Add a new peripheral (e.g., SPI2 on Rockchip)

**Step 1:** Look up the SoC TRM (Technical Reference Manual) for:
- Which bank and pin numbers the peripheral uses
- Which mux function value selects SPI2 on those pins

**Step 2:** Add pin group nodes inside the pinctrl controller node:

```dts
&pinctrl {
    spi2 {
        spi2_clk: spi2-clk {
            rockchip,pins =
                <2 RK_PA0 2 &pcfg_pull_none>;
        };
        spi2_mosi: spi2-mosi {
            rockchip,pins =
                <2 RK_PA1 2 &pcfg_pull_none>;
        };
        spi2_miso: spi2-miso {
            rockchip,pins =
                <2 RK_PA2 2 &pcfg_pull_none>;
        };
        spi2_cs0: spi2-cs0 {
            rockchip,pins =
                <2 RK_PA3 2 &pcfg_pull_up>;
        };
    };
};
```

**Step 3:** Reference from the consumer:

```dts
&spi2 {
    pinctrl-names = "default";
    pinctrl-0 = <&spi2_clk>, <&spi2_mosi>, <&spi2_miso>, <&spi2_cs0>;
    status = "okay";
};
```

---

#### Recipe 2: Add a new shared config node

When multiple peripherals need the same electrical configuration:

```dts
&pinctrl {
    pcfg_pull_up_drv_12ma: pcfg-pull-up-drv-12ma {
        bias-pull-up;
        drive-strength = <12>;
    };
};
```

Then reference it in any pin group:
```dts
rockchip,pins = <1 RK_PA0 1 &pcfg_pull_up_drv_12ma>;
```

---

#### Recipe 3: Add a sleep state

```dts
&pinctrl {
    uart2 {
        uart2_xfer: uart2-xfer {
            rockchip,pins =
                <2 RK_PB0 2 &pcfg_pull_up>,
                <2 RK_PB1 2 &pcfg_pull_up>;
        };
        uart2_sleep: uart2-sleep {
            rockchip,pins =
                <2 RK_PB0 0 &pcfg_pull_none>,  /* mux=0 → GPIO mode */
                <2 RK_PB1 0 &pcfg_pull_none>;
        };
    };
};

&uart2 {
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&uart2_xfer>;
    pinctrl-1 = <&uart2_sleep>;
};
```

---

#### Recipe 4: Use a pin as GPIO (Rockchip)

Set mux to `RK_FUNC_GPIO` (which is 0):

```dts
&pinctrl {
    leds {
        led_pin: led-pin {
            rockchip,pins =
                <0 RK_PA0 RK_FUNC_GPIO &pcfg_pull_none>;
        };
    };
};

gpio-leds {
    pinctrl-names = "default";
    pinctrl-0 = <&led_pin>;

    led {
        gpios = <&gpio0 RK_PA0 GPIO_ACTIVE_HIGH>;
    };
};
```

---

#### Recipe 5: Pin controller hogging its own pins

Sometimes pins must be configured at boot before any consumer asks for them
(e.g., power control pins, reset lines). The pin controller consumes its own pins:

```dts
&pinctrl {
    pinctrl-names = "default";
    pinctrl-0 = <&power_pins>;

    power_pins: power-pins {
        rockchip,pins =
            <0 RK_PA5 0 &pcfg_output_high>;  /* hold power rail high */
    };
};
```

This is called **hogging** — the controller is both provider and consumer.

---

### 6.6 How to read an existing DTS pinctrl setup

When you encounter pinctrl in an existing DTS file, trace it like this:

**Step 1:** Find the consumer. Look for `pinctrl-0`:
```dts
&emmc {
    pinctrl-names = "default";
    pinctrl-0 = <&emmc_clk>, <&emmc_cmd>, <&emmc_bus8>;
};
```

**Step 2:** Follow the phandles to find the pin group nodes:
```dts
emmc_clk: emmc-clk {
    rockchip,pins = <3 RK_PB1 2 &pcfg_pull_none_drv_8ma>;
};
```

**Step 3:** Read each tuple: `<bank pin mux &config>`:
- Bank 3, pin RK_PB1 (= 9), mux function 2, no pull + 8mA drive

**Step 4:** Follow the config phandle to see electrical settings:
```dts
pcfg_pull_none_drv_8ma: pcfg-pull-none-drv-8ma {
    bias-disable;
    drive-strength = <8>;
};
```

**Step 5:** Check the SoC TRM for what mux function 2 means on bank3 pin9.
That's the only thing you can't derive from the DTS — the mux value mapping
is hardware-defined.

---

### 6.7 Common patterns and tips

**Splitting groups by signal type:**
For SD/MMC, it's common to split CLK, CMD, and data into separate groups because
they often need different drive strengths:
```dts
pinctrl-0 = <&sdmmc_clk>, <&sdmmc_cmd>, <&sdmmc_bus4>;
```
CLK might need lower drive strength to reduce EMI, CMD and data need higher.

**Multiple pin controllers:**
A single consumer can reference nodes from different pin controllers:
```dts
pinctrl-0 = <&pinctrl_a_uart_tx>, <&pinctrl_b_uart_rx>;
```
Each phandle's parent controller will be found by walking up the DT tree.

**Composing states from parts:**
You can combine multiple group nodes into one state. This is useful for interfaces
with optional signals (e.g., UART with optional flow control):
```dts
/* Basic: just TX/RX */
pinctrl-0 = <&uart0_xfer>;

/* Full: TX/RX + RTS/CTS */
pinctrl-0 = <&uart0_xfer>, <&uart0_rts_cts>;
```

**The intermediate function node is optional:**
In Rockchip, the grouping node (e.g., `uart2 { ... }`) is just for human
readability. The kernel only cares about the leaf nodes that contain `rockchip,pins`.
You could put everything flat:
```dts
/* This works too, just less organized: */
uart2_xfer: uart2-xfer {
    rockchip,pins = <2 RK_PB0 2 &pcfg_pull_up>,
                    <2 RK_PB1 2 &pcfg_pull_up>;
};
```

---

### 6.8 Quick reference: vendor format comparison

| Vendor | Mux property | Mux encoding | Config location |
|---|---|---|---|
| **Rockchip** | `rockchip,pins` | `<bank pin mux_val &config_phandle>` | Separate shared nodes (pcfg-*) |
| **STM32** | `pinmux` | `STM32_PINMUX(port, line, AFn)` | Same node as pinmux |
| **i.MX** | `fsl,pins` | `MACRO__FUNC  0xCONFIG` | Raw register value inline |
| **Generic** | `function` + `groups`/`pins` | String names | Same node or separate |
| **Qualcomm** | `pins` + `function` | String names | Same node (`bias-*`, `drive-*`) |

Each vendor encodes the same information (which pin, which mux, what config)
differently, but the consumer side (`pinctrl-names`, `pinctrl-0`, etc.) is
always the same across all vendors.

---

## Part 7 — The Three ops structs and How All Structs Fit Together

```
  The core question — what are the three ops and how do they differ:

  ┌─────────────┬──────────────────────────────────────┬───────────────────────────────────┐
  │ ops struct  │               Concern                │         Writes hardware?          │
  ├─────────────┼──────────────────────────────────────┼───────────────────────────────────┤
  │ pinctrl_ops │ Inventory — groups, pins, DT parsing │ No — pure bookkeeping             │
  ├─────────────┼──────────────────────────────────────┼───────────────────────────────────┤
  │ pinmux_ops  │ Muxing — connect pins to peripherals │ Yes — writes IOMUX registers      │
  ├─────────────┼──────────────────────────────────────┼───────────────────────────────────┤
  │ pinconf_ops │ Config — pull, drive, schmitt, slew  │ Yes — writes pad config registers │
  └─────────────┴──────────────────────────────────────┴───────────────────────────────────┘

  They're plugged into a single pinctrl_desc which you register with the core. The core calls mux first, then config, in that order.

  The file also now includes:
  - ASCII diagram showing how the three ops connect to hardware
  - Full struct relationship diagram (driver side, consumer side, device core, map table)
  - Header file guide ("which header do I include?")
  - Decision table ("I am writing X, I need Y")
  - One-sentence summary for every struct in the subsystem

```

### 7.1 The three concerns of a pin controller

A pin controller does three things. Linux splits them into three independent
ops structures, one per concern:

```
What a pin controller does:
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  1. KNOW ABOUT PINS AND GROUPS          → pinctrl_ops               │
│     "I have 128 pins, organized into                                │
│      groups like uart0_grp, spi1_grp"                               │
│                                                                     │
│  2. MUX PINS TO FUNCTIONS               → pinmux_ops                │
│     "Connect this group of pins to                                  │
│      the UART0 peripheral"                                          │
│                                                                     │
│  3. CONFIGURE PIN ELECTRICAL PROPERTIES → pinconf_ops                │
│     "Set this pin to pull-up, 8mA                                   │
│      drive strength"                                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

These three ops are plugged into a single `pinctrl_desc` which is what you
register with the core:

```c
struct pinctrl_desc my_desc = {
    .name    = "my-pinctrl",
    .pins    = my_pins,          /* array of pin descriptors */
    .npins   = ARRAY_SIZE(my_pins),
    .pctlops = &my_pinctrl_ops,  /* ← concern 1: pins & groups */
    .pmxops  = &my_pinmux_ops,   /* ← concern 2: muxing */
    .confops = &my_pinconf_ops,  /* ← concern 3: config */
};

devm_pinctrl_register_and_init(dev, &my_desc, drvdata, &pctldev);
```

---

### 7.2 pinctrl_ops — "What pins and groups do I have?"

This is the **inventory** layer. It tells the core what this controller manages.
It does NOT write any hardware registers. It is pure bookkeeping.

```c
struct pinctrl_ops {
    /* How many groups? */
    int (*get_groups_count)(pctldev);

    /* What is group #N called? */
    const char *(*get_group_name)(pctldev, selector);

    /* Which pins are in group #N? */
    int (*get_group_pins)(pctldev, selector, &pins, &num_pins);

    /* Parse DT node → create map entries (DT bridge) */
    int (*dt_node_to_map)(pctldev, np_config, &map, &num_maps);
    void (*dt_free_map)(pctldev, map, num_maps);

    /* Optional debugfs hook */
    void (*pin_dbg_show)(pctldev, seq_file, offset);
};
```

**When you need it:** Always. Every pinctrl driver must provide this.

**What it answers:**
- "How many groups exist?" → `get_groups_count()`
- "What is group 7 called?" → `get_group_name(7)` → `"uart0-xfer"`
- "Which pins are in group 7?" → `get_group_pins(7)` → `[10, 11]`
- "Parse this DT node into map entries" → `dt_node_to_map()`

**It does NOT:** write any registers, change any mux, configure any pull-up.

```
┌───────────────────────────────────────────┐
│  pinctrl_ops = the DIRECTORY              │
│                                           │
│  group 0: "uart0-xfer"  → pins [10, 11]  │
│  group 1: "uart0-cts"   → pins [12, 13]  │
│  group 2: "spi1-pins"   → pins [20..23]  │
│  group 3: "i2c0-pins"   → pins [30, 31]  │
│  ...                                      │
│                                           │
│  + dt_node_to_map: DT → map translator   │
└───────────────────────────────────────────┘
```

---

### 7.3 pinmux_ops — "Connect these pins to this peripheral"

This is the **muxing** layer. It writes IOMUX registers. It decides which
internal peripheral gets connected to which physical pins.

```c
struct pinmux_ops {
    /* How many functions? */
    int (*get_functions_count)(pctldev);

    /* What is function #N called? */
    const char *(*get_function_name)(pctldev, selector);

    /* Which groups can function #N use? */
    int (*get_function_groups)(pctldev, selector, &groups, &num_groups);

    /* ★ THE MAIN ONE: apply mux for this function + group */
    int (*set_mux)(pctldev, func_selector, group_selector);

    /* GPIO-specific mux overrides */
    int (*gpio_request_enable)(pctldev, range, offset);
    void (*gpio_disable_free)(pctldev, range, offset);
    int (*gpio_set_direction)(pctldev, range, offset, input);

    /* Per-pin request/free hooks */
    int (*request)(pctldev, offset);
    int (*free)(pctldev, offset);

    bool strict;  /* disallow simultaneous GPIO + mux on same pin */
};
```

**When you need it:** When your controller does pin muxing (almost always).

**What it answers:**
- "How many functions?" → `get_functions_count()`
- "What is function 3 called?" → `get_function_name(3)` → `"uart0"`
- "Which groups can uart0 use?" → `get_function_groups(3)` → `["uart0-xfer", "uart0-alt"]`
- "Enable uart0 on group uart0-xfer" → `set_mux(func=3, group=0)` → **writes IOMUX register**

**The core calls `set_mux()` like this:**

```
pinctrl_select_state()
  → pinctrl_commit_state()
    → pinmux_enable_setting()
      → request each pin (usecount check)
      → ops->set_mux(pctldev, func_selector, group_selector)
                                    ↓
                        YOUR DRIVER WRITES THE IOMUX REGISTER
```

```
┌───────────────────────────────────────────┐
│  pinmux_ops = the MUX SWITCH             │
│                                           │
│  func 0: "gpio"   → groups [all]         │
│  func 1: "uart0"  → groups [0, 1]        │
│  func 2: "spi1"   → groups [2]           │
│  func 3: "i2c0"   → groups [3]           │
│                                           │
│  set_mux(func=1, group=0):               │
│    → write 0x02 into IOMUX reg for pin10 │
│    → write 0x02 into IOMUX reg for pin11 │
└───────────────────────────────────────────┘
```

---

### 7.4 pinconf_ops — "Set pull-up, drive strength, etc."

This is the **electrical configuration** layer. It writes pad configuration
registers (pull, drive strength, schmitt, slew rate, etc.).

```c
struct pinconf_ops {
    bool is_generic;  /* use generic pinconf parsing */

    /* Get/set config for a single pin */
    int (*pin_config_get)(pctldev, pin, &config);
    int (*pin_config_set)(pctldev, pin, configs[], num_configs);

    /* Get/set config for an entire group at once */
    int (*pin_config_group_get)(pctldev, selector, &config);
    int (*pin_config_group_set)(pctldev, selector, configs[], num_configs);

    /* Debugfs hooks */
    void (*pin_config_dbg_show)(...);
    void (*pin_config_group_dbg_show)(...);
    void (*pin_config_config_dbg_show)(...);
};
```

**When you need it:** When your controller supports electrical configuration
(pull-up/down, drive strength, etc.) — almost always.

**What it answers:**
- "What is pin 10's current pull config?" → `pin_config_get(10, &config)`
- "Set pin 10 to pull-up + 8mA" → `pin_config_set(10, [PULL_UP, DRV_8MA], 2)`

**The core calls `pin_config_set()` like this:**

```
pinctrl_select_state()
  → pinctrl_commit_state()
    → first: all pinmux_enable_setting() calls     ← mux first
    → then:  all pinconf_apply_setting() calls      ← config second
      → ops->pin_config_set(pctldev, pin, configs[], n)
                                    ↓
                        YOUR DRIVER WRITES THE PAD CONFIG REGISTER
```

```
┌───────────────────────────────────────────┐
│  pinconf_ops = the ELECTRICAL KNOBS      │
│                                           │
│  pin_config_set(pin=10, [PULL_UP]):       │
│    → write pull-up bit in PAD_CFG reg    │
│                                           │
│  pin_config_set(pin=10, [DRV_STRENGTH=8]):│
│    → write drive strength field           │
│                                           │
│  pin_config_group_set(group=2, [SLEW=1]): │
│    → write slew rate for all pins in grp │
└───────────────────────────────────────────┘
```

---

### 7.5 How the three ops relate — the big picture

```
                     pinctrl_desc (what you register)
                    ┌──────────────────────────────┐
                    │  .name = "rockchip-pinctrl"   │
                    │  .pins = [all 128 pins]       │
                    │  .npins = 128                 │
                    │                               │
                    │  .pctlops ─────┐              │
                    │  .pmxops ──────┼──┐           │
                    │  .confops ─────┼──┼──┐        │
                    └────────────────┼──┼──┼────────┘
                                     │  │  │
           ┌─────────────────────────┘  │  │
           │         ┌──────────────────┘  │
           │         │     ┌───────────────┘
           ▼         ▼     ▼
    ┌─────────┐ ┌────────┐ ┌────────┐
    │pinctrl  │ │pinmux  │ │pinconf │
    │  _ops   │ │  _ops  │ │  _ops  │
    ├─────────┤ ├────────┤ ├────────┤
    │INVENTORY│ │MUXING  │ │CONFIG  │
    │         │ │        │ │        │
    │groups   │ │funcs   │ │pull    │
    │pins     │ │set_mux │ │drive   │
    │DT parse │ │GPIO req│ │schmitt │
    └────┬────┘ └───┬────┘ └───┬────┘
         │          │          │
         │     WRITES IOMUX  WRITES PAD
         │     REGISTER      CONFIG REG
    READS ONLY      │          │
    (no HW write)   ▼          ▼
                ┌──────────────────┐
                │  HARDWARE        │
                │  REGISTERS       │
                └──────────────────┘
```

**The order matters:** when the core applies a state, it first does ALL mux
settings (`pinmux_ops->set_mux`) and then ALL config settings
(`pinconf_ops->pin_config_set`). Mux before config. This ensures the pin is
connected to the right peripheral before you set its electrical properties.

---

### 7.6 All header files at a glance — which one do I include?

```
  include/linux/pinctrl/
  │
  ├── pinctrl.h          ← DRIVER: register your controller
  │   structs: pinctrl_desc, pinctrl_ops, pinctrl_pin_desc,
  │            pingroup, pinfunction, pinctrl_gpio_range
  │   funcs:   pinctrl_register(), devm_pinctrl_register_and_init()
  │   YOU: "I'm writing a pin controller driver"
  │
  ├── pinmux.h           ← DRIVER: mux operations
  │   structs: pinmux_ops
  │   YOU: "My controller can mux pins to different functions"
  │
  ├── pinconf.h          ← DRIVER: config operations
  │   structs: pinconf_ops
  │   YOU: "My controller can set pull/drive/schmitt/etc."
  │
  ├── pinconf-generic.h  ← DRIVER: use generic config parsing
  │   enums:   pin_config_param (PIN_CONFIG_BIAS_PULL_UP, etc.)
  │   funcs:   pinconf_generic_parse_dt_config()
  │   YOU: "I want the core to parse DT config properties for me
  │         instead of doing it myself"
  │
  ├── consumer.h         ← CONSUMER: device drivers that need pins
  │   structs: (opaque) pinctrl, pinctrl_state
  │   funcs:   pinctrl_get(), pinctrl_lookup_state(),
  │            pinctrl_select_state(), devm_pinctrl_get_select()
  │            pinctrl_gpio_request(), pinctrl_gpio_direction_*()
  │            pinctrl_pm_select_sleep_state(), etc.
  │   YOU: "I'm a UART/SPI/I2C driver and I need my pins set up"
  │
  ├── machine.h          ← BOARD: static mapping tables (pre-DT era)
  │   structs: pinctrl_map, pinctrl_map_mux, pinctrl_map_configs
  │   enums:   pinctrl_map_type
  │   macros:  PIN_MAP_MUX_GROUP(), PIN_MAP_CONFIGS_PIN(), etc.
  │   funcs:   pinctrl_register_mappings()
  │   YOU: "I'm defining pin mappings in C code (no DT)"
  │
  ├── devinfo.h          ← DEVICE CORE (internal)
  │   structs: dev_pin_info
  │   funcs:   pinctrl_bind_pins()
  │   YOU: you rarely include this directly — the device core uses it
  │         to automatically apply "default" state before probe
  │
  └── pinctrl-state.h    ← CONSTANTS: standard state name strings
      defines: PINCTRL_STATE_DEFAULT  = "default"
               PINCTRL_STATE_INIT     = "init"
               PINCTRL_STATE_SLEEP    = "sleep"
               PINCTRL_STATE_IDLE     = "idle"
      YOU: "I need the standard state name constant"
```

---

### 7.7 All major structs and how they connect

```
DRIVER SIDE (you write a pin controller driver)
═══════════════════════════════════════════════

  pinctrl_pin_desc[]        pingroup[]              pinfunction[]
  ┌──────────────┐       ┌───────────────┐       ┌──────────────────┐
  │ {0, "PA0"}   │       │ "uart0-xfer"  │       │ "uart0"          │
  │ {1, "PA1"}   │       │ pins=[10,11]  │       │ groups=["uart0-  │
  │ {2, "PA2"}   │       │ npins=2       │       │   xfer","uart0-  │
  │ ...          │       ├───────────────┤       │   alt"]          │
  │ {127,"PD7"}  │       │ "spi1-pins"   │       │ ngroups=2        │
  └──────┬───────┘       │ pins=[20..23] │       ├──────────────────┤
         │               └───────┬───────┘       │ "spi1"           │
         │                       │               │ groups=["spi1-   │
         │                       │               │   pins"]         │
         ▼                       ▼               └────────┬─────────┘
  pinctrl_desc ◄─────────────────┘                        │
  ┌──────────────────────┐                                │
  │ .pins = pin_desc[]   │  pinctrl_ops ◄─── uses groups ─┘
  │ .npins = 128         │  ┌────────────────────┐
  │ .pctlops ────────────┼─►│ get_groups_count() │
  │ .pmxops  ────────┐   │  │ get_group_name()   │
  │ .confops ─────┐  │   │  │ get_group_pins()   │
  └───────────────┼──┼───┘  │ dt_node_to_map()   │
                  │  │      └────────────────────┘
                  │  │
                  │  │   pinmux_ops ◄─── uses functions + groups
                  │  │   ┌──────────────────────────┐
                  │  └──►│ get_functions_count()     │
                  │      │ get_function_name()       │
                  │      │ get_function_groups()     │
                  │      │ set_mux(func, group)  ★   │ ← writes IOMUX regs
                  │      │ gpio_request_enable()     │
                  │      └──────────────────────────┘
                  │
                  │   pinconf_ops
                  │   ┌──────────────────────────┐
                  └──►│ pin_config_get(pin)       │
                      │ pin_config_set(pin) ★     │ ← writes PAD_CFG regs
                      │ pin_config_group_set()    │
                      └──────────────────────────┘


CONSUMER SIDE (UART/SPI/I2C driver — uses consumer.h)
═══════════════════════════════════════════════════════

  struct pinctrl            struct pinctrl_state
  ┌─────────────────┐      ┌─────────────────────┐
  │ (opaque handle) │──────│ "default"            │
  │                 │      │ [settings list]      │
  │                 │      ├─────────────────────┤
  │                 │──────│ "sleep"              │
  │                 │      │ [settings list]      │
  └─────────────────┘      └─────────────────────┘
        ▲
        │  devm_pinctrl_get(dev)
        │  pinctrl_lookup_state(p, "default")
        │  pinctrl_select_state(p, state)
        │
  ┌─────┴──────────────┐
  │ Your device driver  │
  │ (e.g. UART)        │
  └────────────────────┘


DEVICE CORE (automatic, you don't call this)
═══════════════════════════════════════════════

  struct dev_pin_info                        pinctrl-state.h
  ┌──────────────────────┐                  ┌────────────────────────┐
  │ .p = pinctrl handle  │                  │ PINCTRL_STATE_DEFAULT  │
  │ .default_state       │                  │ PINCTRL_STATE_INIT     │
  │ .init_state          │                  │ PINCTRL_STATE_SLEEP    │
  │ .sleep_state         │                  │ PINCTRL_STATE_IDLE     │
  │ .idle_state          │                  └────────────────────────┘
  └──────────────────────┘
  Lives inside struct device → dev->pins
  Filled by pinctrl_bind_pins() BEFORE your driver's probe()


MAP TABLE (the glue between DT and settings — machine.h)
═══════════════════════════════════════════════════════════

  pinctrl_map                              pinctrl_map_type
  ┌───────────────────────────────┐       ┌────────────────────────┐
  │ .dev_name = "uart0"           │       │ PIN_MAP_TYPE_MUX_GROUP │
  │ .name = "default"             │       │ PIN_MAP_TYPE_CONFIGS_PIN│
  │ .type = MUX_GROUP             │       │ PIN_MAP_TYPE_CONFIGS_GRP│
  │ .ctrl_dev_name = "pinctrl0"   │       │ PIN_MAP_TYPE_DUMMY     │
  │ .data.mux.function = "uart0"  │       └────────────────────────┘
  │ .data.mux.group = "uart0-xfer"│
  └───────────────────────────────┘
  Normally created by dt_node_to_map() at runtime.
  Can also be defined in C code with PIN_MAP_MUX_GROUP() macros.
```

---

### 7.8 Decision table: "which struct/header do I need?"

| I am writing... | I include... | I fill in... |
|---|---|---|
| A pin controller driver | `pinctrl.h`, `pinmux.h`, `pinconf.h` | `pinctrl_desc` with all three ops |
| A pin controller with generic DT config | Also `pinconf-generic.h` | Set `.is_generic = true` in confops |
| A device driver that needs pins configured | `consumer.h` | Nothing — device core does it via DT |
| A device driver switching states at runtime | `consumer.h` | Call `pinctrl_lookup_state()` + `pinctrl_select_state()` |
| Board code without DT (legacy) | `machine.h` | `pinctrl_map[]` with `PIN_MAP_MUX_GROUP()` macros |
| A GPIO driver that backs onto pinctrl | `consumer.h` | Call `pinctrl_gpio_request()`, `pinctrl_gpio_direction_*()` |

---

### 7.9 One sentence each

| Struct | One sentence |
|---|---|
| `pinctrl_ops` | "Tell the core what groups and pins you have, and translate DT nodes into map entries." |
| `pinmux_ops` | "Tell the core what functions you have, and write the IOMUX register when asked." |
| `pinconf_ops` | "Read and write electrical pad settings (pull, drive, schmitt) for individual pins or groups." |
| `pinctrl_desc` | "The registration form — bundles pins[], pctlops, pmxops, confops into one object." |
| `pinctrl_pin_desc` | "One entry per physical pin: just a number and a name." |
| `pingroup` | "A named set of pin numbers that are used together." |
| `pinfunction` | "A named mux function with a list of groups it can be applied to." |
| `pinctrl_gpio_range` | "Maps a range of GPIO numbers to pin controller pin numbers." |
| `pinctrl` | "Opaque handle a consumer gets from `pinctrl_get()` — represents 'my pins'." |
| `pinctrl_state` | "A named state inside a pinctrl handle — e.g. 'default' or 'sleep'." |
| `pinctrl_map` | "One row in the mapping table: consumer + state + controller + mux/config data." |
| `dev_pin_info` | "Lives inside `struct device` — the device core's cached pinctrl handle and standard states." |
| `pin_config_param` | "Enum of all generic config types: `PIN_CONFIG_BIAS_PULL_UP`, `PIN_CONFIG_DRIVE_STRENGTH`, etc." |
