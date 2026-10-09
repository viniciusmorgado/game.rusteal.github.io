# Structs and data tables

**The goal:** a struct declared in Rust that Unreal understands: the row of a data table, or a group of values a class's property holds.

```rust
use bindings::engine::StaticMesh;
use rusteal_runtime::runtime::{SoftObjectRef, SubclassOf};
use rusteal_runtime::ustruct;

#[ustruct]
pub struct WeaponTableRow {
    #[uproperty(EditAnywhere)]
    static_mesh: SoftObjectRef<StaticMesh>,

    #[uproperty(EditAnywhere)]
    weapon_to_spawn: SubclassOf<Weapon>,

    #[uproperty(EditAnywhere)]
    damage: f32,
}
```

A `#[ustruct]` is Unreal's `USTRUCT()`. Every field is a `#[uproperty]`, of the kinds a class's properties are, without `default`: a struct starts zeroed. Its memory belongs to Unreal (a data table's row, a property of an object, an `OwnedStruct`), so the Rust type itself is a marker, and the fields are read and written through a trait generated next to it, `WeaponTableRowExt`, on `UStructRef<WeaponTableRow>` and `OwnedStruct<WeaponTableRow>`:

```rust
use super::pickup::WeaponTableRowExt;

let damage = row.damage();
row.set_damage(30.0);
```

## Use case: a data table of weapons

**The goal:** designers list the weapons in a table, and a pickup shows and grants the one it is set to.

### In the editor

1. In the **Content Browser**, right-click an empty area and choose **Miscellaneous > Data Table**.
2. In the **Pick Row Structure** window, choose **WeaponTableRow** and click **OK**.
3. Name it `DT_WeaponData` and open it.
4. Click **Add** to add a row, name it (`Pistol`), and fill in **Static Mesh**, **Weapon To Spawn** and **Damage** in the **Row Editor**.
5. Click **Save**.

> **Screenshot:** DT_WeaponData open, with a Pistol row filled in the Row Editor. *(images/classes/data_table_rows.png)*

### In Rust

The pickup holds a row handle, which the editor shows as a table and a row name:

```rust
use bindings::engine::{FDataTableRowHandle, StaticMeshComponent, StaticMeshComponentExt};
use bindings::prelude::*;

#[uclass(parent = Actor)]
pub struct WeaponPickup {
    #[component(root)]
    mesh: StaticMeshComponent,

    #[uproperty(EditAnywhere, category = "Pickup")]
    weapon_type: OwnedStruct<FDataTableRowHandle>,

    #[uproperty]
    weapon_class: SubclassOf<Weapon>,
}

#[uclass_impl]
impl WeaponPickup {
    #[ufunction(Override)]
    fn receive_begin_play(&mut self) {
        let weapon_type = self.weapon_type();

        let Some(row) = weapon_type.get_row::<WeaponTableRow>() else {
            ulog!(LOG_WARNING, "[MyGame] pickup without a weapon type");
            return;
        };

        if let Ok(static_mesh) = row.static_mesh().load_synchronous()
            && let Ok(mesh) = self.mesh().and_then(|m| m.checked())
        {
            mesh.set_static_mesh(static_mesh);
        }

        self.set_weapon_class(row.weapon_to_spawn());
    }
}
```

`get_row::<T>()` is C++'s `FindRow<T>`: it returns `None` when the handle points to no table, to a missing row, or to a table of another row struct. A row can also be found by table and name:

```rust
let row = find_data_table_row::<WeaponTableRow>(table, FName::new("Pistol"));
```

Then, on each **WeaponPickup** in the level (or on its Blueprint child), set **Weapon Type**: the **Data Table** to `DT_WeaponData` and the **Row Name** to `Pistol`.

> **Screenshot:** the Details panel of a WeaponPickup with Weapon Type set to DT_WeaponData and the Pistol row. *(images/classes/row_handle.png)*

## Use case: a struct property

A class can hold your struct as a property:

```rust
#[uproperty(EditAnywhere)]
stats: OwnedStruct<WeaponTableRow>,
```

The getter returns a copy: change it and set it back.

```rust
let stats = self.stats();
stats.as_ref().set_damage(stats.as_ref().damage() * 2.0);
self.set_stats(&stats);
```

An array inside a struct, such as a gearbox's gear ratios, is a `UeArray` view into the struct it belongs to: change it on the copy, then set the copy back.

```rust
let transmission = movement.get_transmission_setup();
transmission.as_ref().forward_gear_ratios().set_all(&[4.25, 2.52, 1.66, 1.22, 1.0])?;
movement.set_transmission_setup(&transmission);
```

## Hot reload

A changed `#[ustruct]` keeps its old fields until the editor restarts: the Output Log says so when it happens. Classes reload without a restart; structs do not, yet.
