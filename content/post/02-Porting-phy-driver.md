---
title: Porting my first phy driver
date: '2024-02-06'
categories:
  - Rust
  - Kernel Driver
  - Linux Kernel
  - net-next
tags:
  - Rust
  - Kernel Driver
  - Linux Kernel
  - net-next
  - network driver
  - rockchip
  - PHY
  - ax88796b
---

[<img class="penguin" src="/static/img/rusty_penguin_2.jpeg" alt="Rusty penguin. Created by DALL·E 3." />](https://github.com/Rust-for-Linux/)

# Why the phy?

I saw that **Fujita Tomonori** had added the **Rust version** of the ax88796b
phy driver for the  Asix PHY, which looked very similar to the C version of this
driver (compare
[ax88796b_rust.rs](https://elixir.bootlin.com/linux/v6.8-rc3/source/drivers/net/phy/ax88796b_rust.rs)
vs
[ax88796b.c](https://elixir.bootlin.com/linux/v6.8-rc3/source/drivers/net/phy/ax88796b.c)).
This driver is meant as a reference driver for further phy drivers. The
[phy.rs](https://elixir.bootlin.com/linux/v6.8-rc3/source/rust/kernel/net/phy.rs)
file uses binding to `struct phy_device` and to pass through C function calls to
phy driver functions.

# Changing Kconfig and Makefile
First step when you add a **new Linux kernel driver** is always to edit the
Makefile, which gives the make system the information to **compile and link** the driver, as well as the Kconfig file, where the user can select to **enable** your driver using the command `make menuconfig`.

So this is how my `drivers/net/phy/Kconfig` looks now:

```
config ROCKCHIP_RUST_PHY
       bool "Rust driver for Rockchip Ethernet PHYs"
       depends on RUST_PHYLIB_ABSTRACTIONS && ROCKCHIP_PHY
       help
         Uses the Rust reference driver for Rockchip PHYs (rockchip_rust.ko).
         The features are equivalent. It supports the integrated Ethernet PHY.
```

Furthermore, since both ax88796b drivers are matched with the same DeviceIds,
the file `drivers/net/phy/Makefile` makes sure that only one of both is compiled
at a time, depending whether the field `CONFIG_ROCKCHIP_RUST_PHY` is defined or
not.

```
ifdef CONFIG_ROCKCHIP_RUST_PHY
obj-$(CONFIG_ROCKCHIP_PHY)     += rockchip_rust.o
else
obj-$(CONFIG_ROCKCHIP_PHY)     += rockchip.o
endif
```

# Make it a module!

Rust **declarative macros** are used to declare the kernel module and its basic
elements, like module name, author, description, drivers, etc.
The `device_table` contains all information that are needed for matching device
and driver (see [next paragraph](#matching-driver-to-device) for more
information on matching).

```
kernel::module_phy_driver! {
    drivers: [PhyRockchip],
    device_table: [
        DeviceId::new_with_driver::<PhyRockchip>(),
    ],
    name: "rust_rockchip_phy",
    author: "John Doe <john.doe@mail.com>",
    description: "Rust Rockchip PHY driver",
    license: "GPL",
}
```

# Matching driver to device
Each phy driver has to implement the **Driver trait** in order to register which
devices it is responsible for. The rockchip driver, for example, is responsible
for any phy showing up with `phy_id & 0xfffffff0 = 0x1234d400`, so 0x1234d400,
0x1234d401, 0x1234d402, etc. The **platform driver** takes care of actually
matching the id to driver.

```
#define INTERNAL_EPHY_ID			0x1234d400

static struct phy_driver rockchip_phy_driver[] = {
{
	.phy_id			= INTERNAL_EPHY_ID,
	.phy_id_mask		= 0xfffffff0,
	.name			= "Rockchip integrated EPHY",
	/* PHY_BASIC_FEATURES */
	.flags			= 0,
...
}
```

The values for matching are nearly a one to one rewrite of the corresponding C
struct:

```
struct PhyRockchip;

#[vtable]
impl Driver for PhyRockchip {
    const FLAGS: u32 = 0;
    const NAME: &'static CStr = c_str!("Rockchip integrated EPHY");
    const PHY_DEVICE_ID: DeviceId = DeviceId::new_with_custom_mask(0x1234d400, 0xfffffff0);
...
```

# Driver functions

Each phy driver can decide which functions to implement and which ones to leave
out. For the `Driver trait`, if a function is implemented, it will be called by
the phy layer.  If the function is not implemented, the **default phy driver
function** that can be found in
[phy.rs](https://elixir.bootlin.com/linux/v6.8-rc3/source/rust/kernel/net/phy.rs)
is called instead.

I implemented only the functions that the original rockchip driver did:  
`soft_reset, config_init, config_aneg, suspend, resume`

```
struct PhyRockchip;

#[vtable]
impl Driver for PhyRockchip {
...
    fn soft_reset(dev: &mut phy::Device) -> Result {
        dev.genphy_soft_reset()
    }
...
}
```

If a function of the **Driver trait** is not implemented, **ENOTSUPP** is returned
in the version of the kernel I used.

```
/// in phy.rs:

#[vtable]
pub trait Driver {
...
    /// Issues a PHY software reset.
    fn soft_reset(_dev: &mut Device) -> Result {
        Err(code::ENOTSUPP)
    }
...
}
```

# Adding bindings
The version of
[phy.rs](https://elixir.bootlin.com/linux/v6.8-rc3/source/rust/kernel/net/phy.rs)
that I used for the driver in January 2024 did not have the function
`config_init`, so I had to add it myself:

```
pub struct Device(Opaque<bindings::phy_device>);

impl Device {
...
    /// Writes BMCR
    pub fn genphy_config_aneg(&mut self) -> Result {
        let phydev = self.0.get();
        // SAFETY: `phydev` is pointing to a valid object by the type invariant of `Self`.
        // So it's just an FFI call.
        // second param = false => autoneg not requested
        to_result(unsafe { bindings::__genphy_config_aneg(phydev, false) })
    }
}
```

Because `struct Device` is defined using `bindings::phy_device` as
initialization, when we use `self.0.get()`, we get back a **raw pointer** to the
`struct phy_device` which can then be passed to `__genphy_config_aneg`.
Since we are calling an unsafe function, the documentation should state why this
pointer is valid during this function call.

The `to_result` function turns a C style error value to a Rust style Result.

# File sizes

In the case of the rockchip driver, it turns out that for both, the **code
file** as well as the **resulting kernel module**, the Rust version is smaller
than the C version of this driver.

```
% wc -l rockchip*
200 drivers/net/phy/rockchip.c
131 drivers/net/phy/rockchip_rust.rs
```

```
% ls -lh rockchip*ko
-rw-r--r-- 1 chrysh chrysh 14K Feb  1 18:42 drivers/net/phy/rockchip.ko
-rw-r--r-- 1 chrysh chrysh 12K Jan 30 16:50 drivers/net/phy/rockchip_rust.ko
```

Unfortunately, I could not try out the Rust Rockchip driver, because I do not
have a board with this phy chip. But it compiles, so ship it!

See the [patch file](https://lore.kernel.org/lkml/20240201-rockchip-rust-phy_depend-v2-3-c5fa4faab924@christina-quast.de/) for the full version of the driver.
