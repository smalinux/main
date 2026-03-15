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
  │  (circuit)│               │          ├──── pin
  └──────────┘                └──────────┘
         ↑                         ↑
       "pad"                     "pin"
   (on the die)            (on the package)
```

In pinctrl documentation and driver code the two words are used interchangeably.
When you see `pad` in a datasheet or driver comment it means the same physical thing
you are already thinking of as "pin." No difference in meaning for pinctrl purposes.
