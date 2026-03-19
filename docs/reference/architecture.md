# Architecture

This chapter explains how Relm4 works internally, helping you understand the framework better.

## Component Model

Relm4 implements the Elm architecture with several key concepts:

### The Component Trait

```rust
pub trait Component: Sized + 'static {
    type Init;           // Initial parameter
    type Input;         // Incoming messages
    type Output;        // Outgoing messages
    type Root;          // Root widget
    type Widgets;       // Widget storage
    type CommandOutput; // Background task results
}
```

### SimpleComponent

A simplified version that combines `update` and `update_view`:

```rust
pub trait SimpleComponent: Sized + 'static {
    // Required methods
    fn init_root() -> Self::Root;
    fn init(...) -> ComponentParts<Self>;
    
    // Optional methods
    fn update(&mut self, msg: Self::Input, sender: ComponentSender<Self>) {}
    fn update_view(&self, widgets: &mut Self::Widgets, sender: ComponentSender<Self>) {}
}
```

## Message Flow

```
User Input (click, keypress, etc.)
        │
        ▼
GTK Signal Handler
        │
        ▼
sender.input(Message)
        │
        ▼
Message Channel
        │
        ▼
Component::update()
        │
        ├──► Update Model
        │
        ▼
Component::update_view()
        │
        ▼
GTK Updates
```

## Component Lifecycle

### 1. Initialization Phase

```rust
// 1. Create root widget
let root = Component::init_root();

// 2. Create component instance
let parts = Component::init(init, root, sender);

// 3. Store model and widgets
let ComponentParts { model, widgets } = parts;
```

### 2. Running Phase

```rust
loop {
    // Wait for message
    let msg = receiver.recv();
    
    // Update model
    component.update(msg, sender);
    
    // Update view
    component.update_view(widgets);
}
```

### 3. Shutdown Phase

```rust
impl Drop for Component {
    fn drop(&mut self) {
        self.shutdown(widgets, output_sender);
    }
}
```

## Concurrency Model

### Main Thread

All UI operations happen on the main thread:

```rust
// UI code runs on main thread
fn update_view(&self, widgets: &mut Self::Widgets, _sender: ComponentSender<Self>) {
    widgets.label.set_label("Updated");  // UI update
}
```

### Background Tasks

Workers and commands run on separate threads:

```rust
// Worker runs on background thread
impl Worker for MyWorker {
    fn update(&mut self, msg: Self::Input, sender: ComponentSender<Self>) {
        // Heavy computation happens here
        let result = compute_heavy();
        
        // Send result back to main thread
        sender.output(AppMsg::Result(result)).unwrap();
    }
}
```

## Channel Communication

### ComponentSender

```rust
pub struct ComponentSender<C: Component> {
    input: Sender<C::Input>,
    output: Sender<C::Output>,
    #[cfg(feature = "macros")]
    late_input: OnceCell<Sender<C::Input>>,
}

impl<C: Component> ComponentSender<C> {
    pub fn input(&self, msg: C::Input) {
        self.input.send(msg);
    }
    
    pub fn output(&self, msg: C::Output) -> Result<(), SendError<C::Output>> {
        self.output.send(msg)
    }
}
```

### Message Priorities

```rust
ComponentBuilder::new()
    .priority(glib::Priority::HIGH)  // High priority messages
    .launch(init)
```

## Widget Hierarchy

### Root Widget

The root widget is typically a window:

```rust
type Root = gtk::Window;

fn init_root() -> Self::Root {
    gtk::Window::builder()
        .title("My App")
        .build()
}
```

### Widget Hierarchy

```
Root (gtk::Window)
└── Child (gtk::Box)
    ├── Grandchild 1 (gtk::Button)
    └── Grandchild 2 (gtk::Label)
```

### Parent-Child Relationships

```rust
window.set_child(Some(&box));      // Set single child
box.append(&button);              // Add to container
box.remove(&button);              // Remove from container
```

## Memory Management

### Reference Counting

GTK widgets use reference counting internally:

```rust
let button = gtk::Button::new();
// button has refcount = 1

let reference = button.clone();
// button has refcount = 2

drop(reference);
// button has refcount = 1
```

### Parent Ownership

```rust
// Parent owns children
let box = gtk::Box::new();
let button = gtk::Button::new();

box.append(&button);  // box now owns button
// button is automatically freed when box is destroyed
```

## Component Hierarchy

### Parent-Child Components

```
App (Root Component)
├── Header (Child Component)
│   └── Title Label
├── Content (Child Component)
│   ├── Sidebar (Child Component)
│   └── Main Area (Child Component)
└── Footer (Child Component)
```

### Communication Pattern

```
Child Component
    │
    ├── input: Receives messages
    │
    ▼
sender.output(Message) ──► Parent Component
                              │
                              ├── Receives as input
                              │
                              ▼
                         Parent processes
```

## Error Handling

### Message Handling Errors

```rust
// Messages are infallible
sender.input(AppMsg::Error);

// Output can fail if receiver is dropped
sender.output(AppMsg::Error)?;  // Returns Result
```

### Component Errors

```rust
impl SimpleComponent for MyComponent {
    fn init(...) -> ComponentParts<Self> {
        // Return Result or panic on error
        let data = load_data().expect("Failed to load data");
        ComponentParts { model, widgets }
    }
}
```

## Summary

- Relm4 implements the Elm architecture
- Components consist of model, widgets, and logic
- Messages flow through channels to the main loop
- UI updates happen on the main thread
- Background work runs on separate threads
- GTK manages widget lifetime through reference counting
- Parent-child relationships define ownership
