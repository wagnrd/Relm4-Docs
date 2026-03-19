# Components

Components are the fundamental building blocks of Relm4 applications. They encapsulate state, logic, and UI into self-contained units.

## What is a Component?

A component in Relm4 is a modular unit that contains:

1. **Model** - The state data
2. **Widgets** - The UI elements
3. **Logic** - How the state changes in response to messages

## SimpleComponent Trait

For most use cases, use `SimpleComponent`:

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
    type Init = u8;           // Initial value passed to the component
    type Input = AppMsg;       // Messages this component receives
    type Output = ();          // Messages this component sends (none for root)
    type Widgets = AppWidgets; // Struct storing widget references
    type Root = gtk::Window;   // The outermost widget

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
            .spacing(10)
            .margin_all(20)
            .build();

        let inc_button = gtk::Button::with_label("Increment (+)");
        let dec_button = gtk::Button::with_label("Decrement (-)");
        let label = gtk::Label::new(Some(&format!("Counter: {}", model.counter)));

        window.set_child(Some(&vbox));
        vbox.append(&inc_button);
        vbox.append(&dec_button);
        vbox.append(&label);

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

## The Component Lifecycle

1. **Initialization**: `init_root()` creates the root widget
2. **Construction**: `init()` creates the model and widgets
3. **Update Loop**: Messages trigger `update()` and `update_view()`
4. **Shutdown**: Optional cleanup in `shutdown()`

## Associated Types

### Init

The initial value for the component:

```rust
type Init = u8;  // Component starts with a counter value
```

### Input

Messages the component receives:

```rust
type Input = AppMsg;  // Component receives AppMsg variants
```

### Output

Messages the component sends to its parent:

```rust
type Output = ();                    // No output (root component)
type Output = ChildOutput;           // Can send ChildOutput messages
```

### Widgets

A struct holding widget references:

```rust
struct AppWidgets {
    label: gtk::Label,
    button: gtk::Button,
    // ... more widgets
}
type Widgets = AppWidgets;
```

### Root

The outermost widget:

```rust
type Root = gtk::Window;           // For window-based components
type Root = gtk::Box;              // For embedded components
type Root = gtk::Popover;         // For popover components
```

## Component Trait Methods

### Required Methods

```rust
fn init_root() -> Self::Root;
fn init(...) -> ComponentParts<Self>;
```

### Optional Methods

```rust
fn update(&mut self, msg: Self::Input, sender: ComponentSender<Self>) {}

fn update_view(&self, widgets: &mut Self::Widgets, sender: ComponentSender<Self>) {}

fn shutdown(&mut self, widgets: &mut Self::Widgets, output: Sender<Self::Output>) {}
```

## Running Components

### As Root Application

```rust
fn main() {
    let app = RelmApp::new("org.relm4.MyApp");
    app.run::<AppModel>(initial_value);
}
```

### As Child Component

```rust
let child = MyComponent::builder()
    .launch(initial_value)
    .forward(sender.input_sender(), transform_output);
```

## Component vs SimpleComponent

`SimpleComponent` is a simplified version of `Component`:

| Feature | SimpleComponent | Component |
|---------|----------------|-----------|
| `update()` signature | `(msg, sender)` | `(msg, sender, root)` |
| `update_view()` signature | `(widgets, sender)` | `(widgets, sender)` |
| `update_cmd()` signature | `(input, output)` | `(msg, sender, root)` |
| `update_cmd_with_view()` | Not available | Available |
| `update_with_view()` | Not available | Available |

Use `SimpleComponent` for simple cases, `Component` when you need more control.

## Summary

- Components are self-contained UI units
- `SimpleComponent` provides a clean API for most use cases
- Define associated types for initialization, input, output, and widgets
- Implement `init()`, `update()`, and `update_view()` methods
- Store widgets in a separate struct for clean separation
