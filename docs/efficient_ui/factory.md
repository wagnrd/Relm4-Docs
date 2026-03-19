# Factories

Factories allow you to efficiently manage collections of similar widgets. They automatically handle creating, updating, and removing widgets as your data changes.

## When to Use Factories

Use factories when you need:

- A list of items (like a to-do list)
- A grid of elements
- Dynamic number of widgets
- Efficient add/remove/reorder operations

## Factory Types

Relm4 provides several factory collection types:

- `FactoryVecDeque` - Ordered collection with efficient add/remove at both ends
- `FactoryVec` - Simple vector-based collection
- `FactoryHashMap` - Key-value based collection
- `AsyncFactoryVecDeque` - Async version for async operations

## Basic Factory Example

Here's a counter list application:

```rust
use gtk::prelude::*;
use relm4::{
    ComponentParts, ComponentSender, RelmApp, SimpleComponent,
    factory::{DynamicIndex, FactoryComponent, FactorySender, FactoryVecDeque},
};

#[derive(Debug)]
struct CounterItem {
    value: u32,
}

#[derive(Debug)]
enum CounterMsg {
    Increment,
    Decrement,
}

#[derive(Debug)]
enum CounterOutput {
    Remove(DynamicIndex),
}

#[derive(Debug)]
enum AppMsg {
    AddCounter,
    RemoveCounter(DynamicIndex),
}

struct AppModel {
    counters: FactoryVecDeque<CounterItem>,
    next_id: u32,
}

struct AppWidgets {
    container: gtk::Box,
}

impl FactoryComponent for CounterItem {
    type Init = u32;
    type Input = CounterMsg;
    type Output = CounterOutput;
    type CommandOutput = ();
    type ParentWidget = gtk::Box;

    fn init_model(value: Self::Init, _index: &DynamicIndex, _sender: FactorySender<Self>) -> Self {
        CounterItem { value }
    }

    fn update(&mut self, msg: Self::Input, _sender: FactorySender<Self>) {
        match msg {
            CounterMsg::Increment => {
                self.value = self.value.wrapping_add(1);
            }
            CounterMsg::Decrement => {
                self.value = self.value.wrapping_sub(1);
            }
        }
    }
}

impl SimpleComponent for AppModel {
    type Init = ();
    type Input = AppMsg;
    type Output = ();
    type Widgets = AppWidgets;
    type Root = gtk::Window;

    fn init_root() -> Self::Root {
        gtk::Window::builder()
            .title("Factory Example")
            .default_width(400)
            .default_height(300)
            .build()
    }

    fn init(
        _: Self::Init,
        window: Self::Root,
        sender: ComponentSender<Self>,
    ) -> ComponentParts<Self> {
        let counters = FactoryVecDeque::builder()
            .launch_default()
            .forward(sender.input_sender(), |msg| match msg {
                CounterOutput::Remove(index) => AppMsg::RemoveCounter(index),
            });

        let model = AppModel {
            counters,
            next_id: 0,
        };

        let container = gtk::Box::builder()
            .orientation(gtk::Orientation::Vertical)
            .spacing(5)
            .margin_all(10)
            .build();

        let add_button = gtk::Button::with_label("Add Counter");

        window.set_child(Some(&container));
        container.append(&add_button);

        add_button.connect_clicked(move |_| {
            sender.input(AppMsg::AddCounter);
        });

        let widgets = AppWidgets { container };

        ComponentParts { model, widgets }
    }

    fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
        match msg {
            AppMsg::AddCounter => {
                self.counters.guard().push_back(self.next_id);
                self.next_id += 1;
            }
            AppMsg::RemoveCounter(index) => {
                self.counters.guard().remove(index.current_index());
            }
        }
    }
}

fn main() {
    let app = Relm4::new("org.relm4.FactoryExample");
    app.run::<AppModel>(());
}
```

## Factory Component Implementation

When implementing `FactoryComponent`, you define:

```rust
impl FactoryComponent for MyItem {
    type Init = InitialValue;     // Data to create item
    type Input = ItemMsg;        // Messages item can receive
    type Output = ItemOutput;    // Messages item can send
    type CommandOutput = ();      // For async (use () for sync)
    type ParentWidget = gtk::Box; // Container widget type
    
    fn init_model(value: Self::Init, index: &DynamicIndex, sender: FactorySender<Self>) -> Self {
        // Create the item from initial value
    }
    
    fn update(&mut self, msg: Self::Input, sender: FactorySender<Self>) {
        // Handle messages
    }
}
```

## Factory Collections

### FactoryVecDeque

Best for ordered lists where you add/remove from both ends:

```rust
use relm4::factory::FactoryVecDeque;

struct AppModel {
    items: FactoryVecDeque<MyItem>,
}

// Add to front
fn update(&mut self, msg: Self::Input, _: ComponentSender<Self>) {
    match msg {
        AppMsg::AddToFront => {
            self.items.guard().push_front(value);
        }
        AppMsg::AddToBack => {
            self.items.guard().push_back(value);
        }
    }
}
```

### FactoryVec

Simple indexed collection:

```rust
use relm4::factory::FactoryVec;

struct AppModel {
    items: FactoryVec<MyItem>,
}

fn update(&mut self, msg: Self::Input, _: ComponentSender<Self>) {
    match msg {
        AppMsg::InsertAt(index, value) => {
            self.items.guard().insert(index, value);
        }
    }
}
```

### FactoryHashMap

For key-value associations:

```rust
use relm4::factory::FactoryHashMap;

struct AppModel {
    entries: FactoryHashMap<String, MyItem>,
}

fn update(&mut self, msg: Self::Input, _: ComponentSender<Self>) {
    match msg {
        AppMsg::AddEntry(key, value) => {
            self.entries.guard().insert(key, value);
        }
        AppMsg::RemoveEntry(key) => {
            self.entries.guard().remove(&key);
        }
    }
}
```

## Using the Guard

All mutating operations require a guard:

```rust
fn update(&mut self, msg: Self::Input, _: ComponentSender<Self>) {
    let mut guard = self.items.guard();
    
    match msg {
        AppMsg::Add(value) => {
            guard.push_back(value);
        }
        AppMsg::Remove(index) => {
            guard.remove(index);
        }
        AppMsg::Clear => {
            guard.clear();
        }
        AppMsg::Move(from, to) => {
            guard.move_to(from, to);
        }
    }
}
```

The guard ensures:
1. Mutable access to the collection
2. Automatic widget updates when guard is dropped
3. Thread-safe updates

## DynamicIndex

Use `DynamicIndex` to reference items even when the collection changes:

```rust
#[derive(Debug)]
enum CounterOutput {
    Remove(DynamicIndex),  // Send our index to parent
}

// In the factory component
fn update(&mut self, msg: Self::Input, sender: FactorySender<Self>) {
    match msg {
        CounterMsg::Remove => {
            // Send index to parent for removal
            sender.output(CounterOutput::Remove(self.index.clone())).unwrap();
        }
    }
}

// In parent component
fn update(&mut self, msg: Self::Input, _: ComponentSender<Self>) {
    match msg {
        AppMsg::RemoveCounter(index) => {
            self.counters.guard().remove(index.current_index());
        }
    }
}
```

## Common Operations

### Iterate with Index

```rust
fn update(&mut self, msg: Self::Input, _: ComponentSender<Self>) {
    match msg {
        AppMsg::DoubleAll => {
            let mut guard = self.items.guard();
            for item in guard.iter_mut() {
                item.value *= 2;
            }
        }
    }
}
```

### Find and Modify

```rust
fn update(&mut self, msg: Self::Input, _: ComponentSender<Self>) {
    match msg {
        AppMsg::FindAndUpdate(predicate, new_value) => {
            let mut guard = self.items.guard();
            if let Some(item) = guard.iter_mut().find(predicate) {
                item.value = new_value;
            }
        }
    }
}
```

### Batch Operations

```rust
fn update(&mut self, msg: Self::Input, _: ComponentSender<Self>) {
    match msg {
        AppMsg::ReplaceAll(new_items) => {
            let mut guard = self.items.guard();
            guard.clear();
            for item in new_items {
                guard.push_back(item);
            }
        }
    }
}
```

## Summary

- Factories manage collections of similar widgets efficiently
- Use `FactoryVecDeque` for ordered lists
- Use `FactoryHashMap` for key-value associations
- Always use a guard for mutating operations
- Use `DynamicIndex` to reference items across changes
- Widgets are automatically created/updated/removed as data changes
