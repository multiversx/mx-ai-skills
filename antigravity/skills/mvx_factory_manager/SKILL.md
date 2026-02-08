---
name: mvx_factory_manager
description: Factory pattern for deploying and managing child contracts from a template.
---

# MultiversX Factory/Manager Pattern

Three core operations: Deploy, Track, Upgrade.

## Deploy

```rust
#[endpoint(deployChild)]
fn deploy_child(&self, child_id: ManagedBuffer, init_arg: BigUint) -> ManagedAddress {
    let template = self.template_address().get();
    let metadata = CodeMetadata::UPGRADEABLE | CodeMetadata::READABLE | CodeMetadata::PAYABLE;

    let new_address = self.tx()
        .typed(child_proxy::ChildProxy)
        .init(init_arg)
        .from_source(template)
        .code_metadata(metadata)
        .returns(ReturnsNewManagedAddress)
        .sync_call();

    self.child_addresses().insert(new_address.clone());
    self.child_by_id(&child_id).set(new_address.clone());
    new_address
}
```

## Track (Registry)

```rust
#[storage_mapper("templateAddress")]
fn template_address(&self) -> SingleValueMapper<ManagedAddress>;

#[storage_mapper("childAddresses")]
fn child_addresses(&self) -> SetMapper<ManagedAddress>;

#[storage_mapper("childById")]
fn child_by_id(&self, id: &ManagedBuffer) -> SingleValueMapper<ManagedAddress>;

#[storage_mapper("admins")]
fn admins(&self) -> UnorderedSetMapper<ManagedAddress>;
```

**Pattern**: SetMapper (iteration + contains) + SingleValueMapper (ID lookup).

## Upgrade

```rust
#[endpoint(upgradeChild)]
fn upgrade_child(&self, child_id: ManagedBuffer) {
    self.require_admin();
    let child = self.child_by_id(&child_id).get();
    require!(self.child_addresses().contains(&child), "Not managed");

    let metadata = CodeMetadata::UPGRADEABLE | CodeMetadata::READABLE | CodeMetadata::PAYABLE;
    self.tx().to(child).typed(child_proxy::ChildProxy).upgrade()
        .code_metadata(metadata)
        .from_source(self.template_address().get())
        .upgrade_async_call_and_exit();
}
```

## CodeMetadata

| Flag | Effect |
|---|---|
| `UPGRADEABLE` | Child can be upgraded (almost always needed) |
| `READABLE` | Other contracts can read storage |
| `PAYABLE` | Can receive EGLD directly |

## Anti-Patterns

- Deploying without tracking (can't upgrade later)
- Missing `UPGRADEABLE` flag
- Upgrading without validating address is a managed child
