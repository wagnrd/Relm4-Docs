# Output Messages

Output messages allow child components to communicate with their parent components. They enable a unidirectional data flow where events bubble up through the component hierarchy.

## What are Output Messages?

Output messages are messages that a component sends to its parent:

```rust
#[derive(Debug)]
enum ChildOutput {
    ValueChanged(u32),
    ItemSelected(String),
    CloseRequested,
}
```

## When to Use Output Messages

Use output messages when:

- A child component needs to notify the parent of a state change
- The parent needs to coordinate actions across multiple children
- You want to maintain a clear data flow hierarchy
- A component should be reusable and not directly manipulate shared state

## Sending Output Messages

Use the `output` method on `ComponentSender`:

```rust
sender.output(AppOutput::ValueChanged(new_value)).unwrap();
```

The `.unwrap()` is safe because output messages always have a destination (the parent component).

## Complete Example

Here's a child component that sends output messages:

```rust
use gtk::prelude::*;
use relm4::{ComponentParts, ComponentSender, RelmApp, SimpleComponent, Sender};

struct CounterModel {
    value: u32,
}

#[derive(Debug)]
enum CounterInput {
    Increment,
    Decrement,
    SetValue(u32),
}

#[derive(Debug)]
enum CounterOutput {
    ValueChanged(u32),
    Overflow,
}

struct CounterWidgets {
    label: gtk::Label,
}

impl SimpleComponent for CounterModel {
    type Init = ();
    type Input = CounterInput;
    type Output = CounterOutput;
    type Widgets = CounterWidgets;
    type Root = gtk::Window;

    fn init_root() -> Self::Root {
        gtk::Window::builder()
            .title("Counter")
            .default_width(200)
            .default_height(100)
            .build()
    }

    fn init(
        _: Self::Init,
        window: Self::Root,
        sender: ComponentSender<Self>,
    ) -> ComponentParts<Self> {
        let model = CounterModel { value: 0 };

        let vbox = gtk::Box::builder()
            .orientation(gtk::Orientation::Vertical)
            .spacing(10)
            .margin_all(20)
            .build();

        let label = gtk::Label::new(Some("0"));
        let inc_button = gtk::Button::with_label("+");
        let dec_button = gtk::Button::with_label("-");

        inc_button.connect_clicked(move |_| {
            sender.input(CounterInput::Increment);
        });

        dec_button.connect_clicked(move |_| {
            sender.input(CounterInput::Decrement);
        });

        window.set_child(Some(&vbox));
        vbox.append(&label);
        vbox.append(&inc_button);
        vbox.append(&dec_button);

        let widgets = CounterWidgets { label };

        ComponentParts { model, widgets }
    }

    fn update(&mut self, msg: Self::Input, sender: ComponentSender<Self>) {
        match msg {
            CounterInput::Increment => {
                let (new_value, overflow) = self.value.overflowing_add(1);
                self.value = new_value;
                sender.output(CounterOutput::ValueChanged(new_value)).unwrap();
                if overflow {
                    sender.output(CounterOutput::Overflow).unwrap();
                }
            }
            CounterInput::Decrement => {
                self.value = self.value.wrapping_sub(1);
                sender.output(CounterOutput::ValueChanged(self.value)).unwrap();
            }
            CounterInput::SetValue(val) => {
                self.value = val;
                sender.output(CounterOutput::ValueChanged(val)).unwrap();
            }
        }
    }

    fn update_view(&self, widgets: &mut Self::Widgets, _sender: ComponentSender<Self>) {
        widgets.label.set_label(&self.value.to_string());
    }
}
```

## Receiving Output Messages in Parent

The parent component receives output messages and can convert them to its own input messages:

```rust
struct AppModel {
    total: u32,
    child_controller: Controller<CounterModel>,
}

#[derive(Debug)]
enum AppMsg {
    ChildOutput(CounterOutput),
}

impl SimpleComponent for AppModel {
    type Init = ();
    type Input = AppMsg;
    type Output = ();
    type Widgets = AppWidgets;
    type Root = gtk::Window;

    fn init_root() -> Self::Root {
        gtk::Window::builder()
            .title("Parent App")
            .default_width(300)
            .default_height(200)
            .build()
    }

    fn init(
        _: Self::Init,
        window: Self::Root,
        sender: ComponentSender<Self>,
    ) -> ComponentParts<Self> {
        // Launch child component and forward its outputs
        let child = CounterModel::builder()
            .launch(())
            .forward(sender.input_sender(), AppMsg::ChildOutput);

        let model = AppModel {
            total: 0,
            child_controller: child,
        };

        let vbox = gtk::Box::builder()
            .orientation(gtk::Orientation::Vertical)
            .spacing(10)
            .margin_all(20)
            .build();

        let total_label = gtk::Label::new(Some("Total: 0"));

        window.set_child(Some(&vbox));
        vbox.append(&total_label);

        let widgets = AppWidgets { total_label };

        ComponentParts { model, widgets }
    }

    fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
        match msg {
            AppMsg::ChildOutput(output) => {
                match output {
                    CounterOutput::ValueChanged(val) => {
                        self.total += val;
                    }
                    CounterOutput::Overflow => {
                        println!("Child counter overflowed!");
                    }
                }
            }
        }
    }

    fn update_view(&self, widgets: &mut Self::Widgets, _sender: ComponentSender<Self>) {
        widgets.total_label.set_label(&format!("Total: {}", self.total));
    }
}
```

## Forwarding Messages

The `forward` method makes it easy to convert child outputs to parent inputs:

```rust
let child = CounterModel::builder()
    .launch(())
    .forward(sender.input_sender(), AppMsg::ChildOutput);
```

The first argument is where to send the messages, and the second is a function that transforms the output to input.

## Summary

- Output messages communicate from child to parent
- Use `sender.output(message).unwrap()` to send them
- The parent receives them via its own input message type
- Use `forward()` to easily bridge child outputs to parent inputs
- This maintains the unidirectional data flow pattern
