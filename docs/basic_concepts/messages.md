# Messages

Messages are how your UI communicates with your component's logic. They represent events or intentions that can change your application's state.

## What are Messages?

In Relm4, messages are enum variants that describe what happened in the UI:

```rust
#[derive(Debug)]
enum AppMsg {
    Increment,
    Decrement,
    SetValue(u8),
    Reset,
}
```

## Message Requirements

Messages must implement:

- `Debug` - For logging and debugging purposes
- `'static` - Messages cannot contain references

## Message Types

Relm4 uses two kinds of messages:

1. **Input Messages** - Received by the component from external sources
2. **Output Messages** - Sent by the component to parent components

### Input Messages

Input messages are processed by your component's `update` function. They typically represent user actions:

```rust
#[derive(Debug)]
enum AppMsg {
    Increment,
    Decrement,
    SetCounter(u8),
}

impl SimpleComponent for AppModel {
    type Input = AppMsg;
    
    fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
        match msg {
            AppMsg::Increment => {
                self.counter = self.counter.wrapping_add(1);
            }
            AppMsg::Decrement => {
                self.counter = self.counter.wrapping_sub(1);
            }
            AppMsg::SetCounter(value) => {
                self.counter = value;
            }
        }
    }
}
```

### Output Messages

Output messages allow child components to communicate with their parents. They're useful for:

- Notifying parent of state changes
- Requesting actions from parent
- Propagating events up the component tree

```rust
// Child component defines its output type
#[derive(Debug)]
enum ChildOutput {
    ValueChanged(u32),
    ButtonClicked,
}

// Parent component receives child outputs
#[derive(Debug)]
enum ParentMsg {
    FromChild(ChildOutput),
}

impl SimpleComponent for ParentModel {
    type Input = ParentMsg;
    
    fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
        match msg {
            ParentMsg::FromChild(output) => {
                match output {
                    ChildOutput::ValueChanged(val) => {
                        println!("Child value changed to: {}", val);
                    }
                    ChildOutput::ButtonClicked => {
                        println!("Child button was clicked!");
                    }
                }
            }
        }
    }
}
```

## Sending Messages

### From Button Click Handlers

```rust
let button = gtk::Button::with_label("Click me");

button.connect_clicked(move |_| {
    sender.input(AppMsg::Increment);
});
```

### From Signal Callbacks

```rust
let entry = gtk::Entry::new();

entry.connect_changed(move |entry| {
    let text = entry.text().to_string();
    sender.input(AppMsg::TextChanged(text));
});
```

## Message Patterns

### Simple Actions

For simple button clicks or menu selections:

```rust
#[derive(Debug)]
enum Msg {
    Save,
    Load,
    Delete,
    Refresh,
}
```

### Parameterized Actions

When you need to pass data with the message:

```rust
#[derive(Debug)]
enum Msg {
    SetTitle(String),
    SetValue(i32),
    SelectItem(usize),
    UpdateUser { id: u64, name: String },
}
```

### State Machine Messages

For managing complex state transitions:

```rust
#[derive(Debug)]
enum ConnectionMsg {
    Connect(String),      // Connect to server
    Disconnect,          // Disconnect
    Reconnecting,        // Lost connection, trying to reconnect
    Error(String),       // Connection error
}
```

## Complete Example

Here's a full example showing input and output messages working together:

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

struct ChildModel {
    value: u32,
}

#[derive(Debug)]
enum ChildInput {
    SetValue(u32),
}

#[derive(Debug)]
enum ChildOutput {
    ValueChanged(u32),
}

struct ChildWidgets {
    label: gtk::Label,
}

impl SimpleComponent for ChildModel {
    type Init = u32;
    type Input = ChildInput;
    type Output = ChildOutput;
    type Widgets = ChildWidgets;
    type Root = gtk::Window;

    fn init_root() -> Self::Root {
        gtk::Window::builder()
            .title("Child")
            .default_width(200)
            .default_height(100)
            .build()
    }

    fn init(
        value: Self::Init,
        window: Self::Root,
        sender: ComponentSender<Self>,
    ) -> ComponentParts<Self> {
        let model = ChildModel { value };

        let vbox = gtk::Box::builder()
            .orientation(gtk::Orientation::Vertical)
            .spacing(5)
            .margin_all(10)
            .build();

        let label = gtk::Label::new(Some(&format!("Value: {}", model.value)));
        let button = gtk::Button::with_label("Send to Parent");

        button.connect_clicked(move |_| {
            let current = 42; // In real app, get from model
            sender.output(ChildOutput::ValueChanged(current)).unwrap();
        });

        window.set_child(Some(&vbox));
        vbox.append(&label);
        vbox.append(&button);

        let widgets = ChildWidgets { label };

        ComponentParts { model, widgets }
    }

    fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
        match msg {
            ChildInput::SetValue(val) => {
                self.value = val;
            }
        }
    }

    fn update_view(&self, widgets: &mut Self::Widgets, _sender: ComponentSender<Self>) {
        widgets.label.set_label(&format!("Value: {}", self.value));
    }
}

fn main() {
    let app = RelmApp::new("org.relm4.MessageExample");
    app.run::<AppModel>(0);
}
```

## Summary

- Messages are enums that describe events or intentions
- Input messages are processed by the component
- Output messages communicate with parent components
- Send messages using `sender.input()` or `sender.output()`
- Keep messages focused and descriptive
