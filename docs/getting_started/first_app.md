# Your First App

In this chapter, we'll build a complete counter application step by step. This will introduce you to all the core concepts of Relm4.

## What We're Building

A simple counter with increment and decrement buttons:

```
┌─────────────────────────────┐
│         Counter App         │
│                             │
│         Counter: 0          │
│                             │
│    [-]           [+]        │
│                             │
└─────────────────────────────┘
```

## Project Setup

Create a new Rust project:

```bash
cargo new my_counter --bin
cd my_counter
```

Add Relm4 to your `Cargo.toml`:

```toml
[dependencies]
relm4 = "0.10"
gtk = { version = "0.10", package = "gtk4" }
```

## The Complete Code

Here's the full application - we'll break it down piece by piece:

```rust
use gtk::prelude::{BoxExt, ButtonExt, GtkWindowExt};
use relm4::{ComponentParts, ComponentSender, RelmApp, SimpleComponent};

struct AppModel {
    counter: u8,
}

#[derive(Debug)]
enum AppMsg {
    Increment,
    Decrement,
}

struct AppWidgets {
    label: gtk::Label,
}

impl SimpleComponent for AppModel {
    type Init = u8;
    type Input = AppMsg;
    type Output = ();
    type Widgets = AppWidgets;
    type Root = gtk::Window;

    fn init_root() -> Self::Root {
        gtk::Window::builder()
            .title("Counter App")
            .default_width(300)
            .default_height(150)
            .build()
    }

    fn init(
        counter: Self::Init,
        window: Self::Root,
        sender: ComponentSender<Self>,
    ) -> ComponentParts<Self> {
        let model = AppModel { counter };

        let vbox = gtk::Box::builder()
            .orientation(gtk::Orientation::Vertical)
            .spacing(10)
            .margin_all(20)
            .build();

        let label = gtk::Label::new(Some(&format!("Counter: {}", model.counter)));
        label.set_margin_all(10);

        let inc_button = gtk::Button::with_label("Increment (+)");
        let dec_button = gtk::Button::with_label("Decrement (-)");

        window.set_child(Some(&vbox));
        vbox.append(&label);
        vbox.append(&inc_button);
        vbox.append(&dec_button);

        inc_button.connect_clicked(move |_| {
            sender.input(AppMsg::Increment);
        });

        dec_button.connect_clicked(move |_| {
            sender.input(AppMsg::Decrement);
        });

        let widgets = AppWidgets { label };

        ComponentParts { model, widgets }
    }

    fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
        match msg {
            AppMsg::Increment => {
                self.counter = self.counter.wrapping_add(1);
            }
            AppMsg::Decrement => {
                self.counter = self.counter.wrapping_sub(1);
            }
        }
    }

    fn update_view(&self, widgets: &mut Self::Widgets, _sender: ComponentSender<Self>) {
        widgets.label.set_label(&format!("Counter: {}", self.counter));
    }
}

fn main() {
    let app = RelmApp::new("org.relm4.CounterExample");
    app.run::<AppModel>(0);
}
```

## Step-by-Step Explanation

### Step 1: Imports

```rust
use gtk::prelude::{BoxExt, ButtonExt, GtkWindowExt};
use relm4::{ComponentParts, ComponentSender, RelmApp, SimpleComponent};
```

We import:
- GTK traits for widget methods (`BoxExt`, `ButtonExt`, `GtkWindowExt`)
- Relm4 types for component implementation

### Step 2: Define the Model

```rust
struct AppModel {
    counter: u8,
}
```

The model stores our application state. For this simple app, we only need to store the counter value.

### Step 3: Define Messages

```rust
#[derive(Debug)]
enum AppMsg {
    Increment,
    Decrement,
}
```

Messages describe what happened in the UI. We have two actions: increment and decrement.

### Step 4: Define Widgets Struct

```rust
struct AppWidgets {
    label: gtk::Label,
}
```

This struct holds references to the widgets we need to update. We only need the label since buttons don't change.

### Step 5: Implement SimpleComponent

#### Associated Types

```rust
type Init = u8;           // Initial counter value
type Input = AppMsg;      // Messages we receive
type Output = ();         // Messages we send (none for root)
type Widgets = AppWidgets; // Our widget struct
type Root = gtk::Window;  // The root widget type
```

#### init_root()

```rust
fn init_root() -> Self::Root {
    gtk::Window::builder()
        .title("Counter App")
        .default_width(300)
        .default_height(150)
        .build()
}
```

Creates the root window. This is called before any other initialization.

#### init()

```rust
fn init(
    counter: Self::Init,
    window: Self::Root,
    sender: ComponentSender<Self>,
) -> ComponentParts<Self> {
    let model = AppModel { counter };
    
    // Create the UI
    let vbox = gtk::Box::builder()
        .orientation(gtk::Orientation::Vertical)
        .spacing(10)
        .margin_all(20)
        .build();

    let label = gtk::Label::new(Some(&format!("Counter: {}", model.counter)));
    let inc_button = gtk::Button::with_label("Increment (+)");
    let dec_button = gtk::Button::with_label("Decrement (-)");

    // Build the widget hierarchy
    window.set_child(Some(&vbox));
    vbox.append(&label);
    vbox.append(&inc_button);
    vbox.append(&dec_button);

    // Connect button signals
    inc_button.connect_clicked(move |_| {
        sender.input(AppMsg::Increment);
    });

    dec_button.connect_clicked(move |_| {
        sender.input(AppMsg::Decrement);
    });

    let widgets = AppWidgets { label };

    ComponentParts { model, widgets }
}
```

The `init` function:
1. Creates the model from the initial value
2. Creates all widgets
3. Builds the widget hierarchy
4. Connects signal handlers
5. Returns both model and widgets

#### update()

```rust
fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
    match msg {
        AppMsg::Increment => {
            self.counter = self.counter.wrapping_add(1);
        }
        AppMsg::Decrement => {
            self.counter = self.counter.wrapping_sub(1);
        }
    }
}
```

Processes incoming messages and updates the model. We use `wrapping_add/sub` to avoid panics on overflow.

#### update_view()

```rust
fn update_view(&self, widgets: &mut Self::Widgets, _sender: ComponentSender<Self>) {
    widgets.label.set_label(&format!("Counter: {}", self.counter));
}
```

Updates the UI to reflect the current model state. This is called after every `update`.

### Step 6: Run the App

```rust
fn main() {
    let app = RelmApp::new("org.relm4.CounterExample");
    app.run::<AppModel>(0);
}
```

Create the app and run it with an initial counter value of 0.

## How It Works

```
User clicks "+"
       │
       ▼
Button sends AppMsg::Increment
       │
       ▼
update() modifies model.counter
       │
       ▼
update_view() updates label text
       │
       ▼
UI reflects new counter value
```

## Running the App

```bash
cargo run
```

You should see a window with a counter and two buttons.

## Improvements You Can Make

### Add Input Validation

```rust
fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
    match msg {
        AppMsg::Increment if self.counter < 255 => {
            self.counter += 1;
        }
        AppMsg::Decrement if self.counter > 0 => {
            self.counter -= 1;
        }
        _ => {}
    }
}
```

### Add Keyboard Shortcuts

```rust
use gtk::gdk::Key;

window.connect_key_pressed(move |_, key, _, _| {
    match key {
        Key::plus | Key::KP_Add => sender.input(AppMsg::Increment),
        Key::minus | Key::KP_Subtract => sender.input(AppMsg::Decrement),
        _ => gtk::Inhibit(false),
    }
});
```

### Store Additional State

```rust
struct AppModel {
    counter: u8,
    increment_amount: u8,
    last_updated: std::time::Instant,
}
```

## Summary

In this chapter, you learned:

1. **Model** - Stores application state
2. **Messages** - Describe UI events
3. **Widgets** - Hold references to UI elements
4. **init()** - Creates the initial UI
5. **update()** - Processes messages, updates model
6. **update_view()** - Updates UI to match model
7. **main()** - Runs the application

This pattern - model, messages, update, view - is the foundation of all Relm4 applications.
