# The Position Function

Factory collections support a `position()` function that returns the current index of an item, which is useful for dynamic positioning and layout decisions.

## Basic Usage

```rust
fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
    match msg {
        AppMsg::MoveToTop(index) => {
            let pos = self.items.guard().position(index.clone());
            if pos > 0 {
                self.items.guard().move_to(pos, 0);
            }
        }
        AppMsg::MoveToBottom(index) => {
            let pos = self.items.guard().position(index.clone());
            let len = self.items.guard().len();
            if pos < len - 1 {
                self.items.guard().move_to(pos, len - 1);
            }
        }
    }
}
```

## Common Patterns

### Move Item Up/Down

```rust
fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
    match msg {
        AppMsg::MoveUp(index) => {
            let mut guard = self.items.guard();
            let pos = guard.position(index.clone());
            if pos > 0 {
                guard.move_to(pos, pos - 1);
            }
        }
        AppMsg::MoveDown(index) => {
            let mut guard = self.items.guard();
            let pos = guard.position(index.clone());
            if pos < guard.len() - 1 {
                guard.move_to(pos, pos + 1);
            }
        }
    }
}
```

### Get Position for Layout

```rust
fn update_view(&self, widgets: &mut Self::Widgets, _sender: ComponentSender<Self>) {
    let guard = self.items.guard();
    for (index, item) in guard.iter().enumerate() {
        // Position widgets based on index
        let col = index % 4;
        let row = index / 4;
        // ... update widget positions
    }
}
```

### Conditional Display Based on Position

```rust
fn update_view(&self, widgets: &mut Self::Widgets, _sender: ComponentSender<Self>) {
    let guard = self.items.guard();
    for (index, item) in guard.iter().enumerate() {
        // Show separator for all items except the first
        widgets.show_separator.set_visible(index > 0);
        
        // Show index in label
        widgets.index_label.set_label(&format!("#{}", index + 1));
    }
}
```

## Position and DynamicIndex

When using `DynamicIndex`, the position can change as items are added or removed:

```rust
#[derive(Debug)]
enum ItemMsg {
    SendToPosition(usize),
}

impl FactoryComponent for MyItem {
    fn update(&mut self, msg: Self::Input, sender: FactorySender<Self>) {
        match msg {
            ItemMsg::SendToPosition(target_pos) => {
                let current_pos = sender.position();
                if current_pos != target_pos {
                    sender.output(ItemOutput::MoveTo(target_pos)).unwrap();
                }
            }
        }
    }
}
```

## Summary

- `position(index)` returns the current position of a `DynamicIndex`
- Useful for move operations and layout decisions
- Position can change as collection is modified
- Combine with guard for atomic operations
