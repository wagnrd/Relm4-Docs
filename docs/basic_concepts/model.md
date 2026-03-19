# Model

The model is the heart of your application - it stores all the state that your UI needs to display.

## What is a Model?

In Relm4, a model is a Rust struct that holds your application's data. Think of it as the "brain" of your application that remembers everything important.

```rust
struct AppModel {
    counter: u8,
    is_active: bool,
    username: String,
}
```

## Model Requirements

For a type to be used as a Relm4 model, it must satisfy a few requirements:

1. **Must implement `Sized`** - All Rust types that don't have dynamic size satisfy this
2. **Must be `'static`** - The model cannot contain references with shorter lifetimes
3. **Stored in your component struct** - The model is typically owned by your component

## Example: Counter Model

A simple counter application only needs to store the counter value:

```rust
use gtk::prelude::*;
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
            .default_height(100)
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
            .spacing(5)
            .margin_all(5)
            .build();

        let label = gtk::Label::new(Some(&format!("Counter: {}", model.counter)));
        label.set_margin_all(5);

        window.set_child(Some(&vbox));
        vbox.append(&label);

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

## Model Best Practices

### Keep Models Simple

Your model should only store data, not logic:

```rust
// Good: Model only stores data
struct AppModel {
    items: Vec<String>,
    selected_index: Option<usize>,
}

// Avoid: Model with embedded logic
struct AppModel {
    items: Vec<String>,
    // Don't do this - keep logic in update functions
    fn get_selected(&self) -> Option<&String> { ... }
}
```

### Use Appropriate Types

Choose the right type for your data:

```rust
struct Settings {
    volume: u8,           // 0-255 for audio
    is_muted: bool,        // Simple on/off state
    username: String,      // Text data
    created_at: std::time::Instant,  // Time tracking
}
```

### Immutable Updates

Prefer creating new instances over mutating in place when you need to track changes:

```rust
// For simple updates, mutating is fine
self.counter += 1;

// For complex state, consider immutable patterns
fn add_item(&mut self, item: String) {
    let mut new_items = self.items.clone();
    new_items.push(item);
    self.items = new_items;
}
```

## Summary

- The model stores all application state
- It's a simple Rust struct with no special requirements
- Keep models focused on data, not logic
- Choose appropriate types for your data
