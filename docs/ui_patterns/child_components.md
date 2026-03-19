# Child Components

Child components are reusable components nested within a parent component. They enable modular, composable UI architectures.

## Why Use Child Components?

- **Reusability**: Write once, use in multiple places
- **Separation of Concerns**: Each component handles its own logic
- **Testability**: Test components in isolation
- **Maintainability**: Smaller, focused code files

## Basic Example: Alert Dialog

Here's a reusable alert dialog component:

```rust
use gtk::prelude::*;
use relm4::{ComponentParts, ComponentSender, RelmApp, SimpleComponent, Controller};

struct AlertModel {
    visible: bool,
    title: String,
    message: String,
    accept_label: String,
    cancel_label: String,
}

#[derive(Debug)]
enum AlertInput {
    Show,
    Hide,
    Accept,
    Cancel,
}

#[derive(Debug)]
enum AlertOutput {
    Confirmed,
    Cancelled,
}

struct AlertWidgets {
    dialog: gtk::MessageDialog,
}

impl SimpleComponent for AlertModel {
    type Init = AlertSettings;
    type Input = AlertInput;
    type Output = AlertOutput;
    type Widgets = AlertWidgets;
    type Root = gtk::Window;

    fn init_root() -> Self::Root {
        gtk::Window::builder()
            .title("Alert")
            .build()
    }

    fn init(
        settings: Self::Init,
        window: Self::Root,
        sender: ComponentSender<Self>,
    ) -> ComponentParts<Self> {
        let model = AlertModel {
            visible: false,
            title: settings.title,
            message: settings.message,
            accept_label: settings.accept_label,
            cancel_label: settings.cancel_label,
        };

        let dialog = gtk::MessageDialog::builder()
            .modal(true)
            .text(&model.title)
            .secondary_text(&model.message)
            .build();

        // Connect response signal
        let dialog_sender = sender.clone();
        dialog.connect_response(move |_, response| {
            match response {
                gtk::ResponseType::Accept => {
                    dialog_sender.input(AlertInput::Accept);
                }
                gtk::ResponseType::Cancel | gtk::ResponseType::DeleteEvent => {
                    dialog_sender.input(AlertInput::Cancel);
                }
                _ => {}
            }
        });

        let widgets = AlertWidgets { dialog };

        ComponentParts { model, widgets }
    }

    fn update(&mut self, msg: Self::Input, sender: ComponentSender<Self>) {
        match msg {
            AlertInput::Show => {
                self.visible = true;
                self.widgets.dialog.show();
            }
            AlertInput::Hide => {
                self.visible = false;
                self.widgets.dialog.hide();
            }
            AlertInput::Accept => {
                self.visible = false;
                self.widgets.dialog.hide();
                sender.output(AlertOutput::Confirmed).unwrap();
            }
            AlertInput::Cancel => {
                self.visible = false;
                self.widgets.dialog.hide();
                sender.output(AlertOutput::Cancelled).unwrap();
            }
        }
    }

    fn update_view(&self, widgets: &mut Self::Widgets, _sender: ComponentSender<Self>) {
        widgets.dialog.set_visible(self.visible);
    }
}

struct AlertSettings {
    title: String,
    message: String,
    accept_label: String,
    cancel_label: String,
}

// Re-export for convenience
type AlertController = Controller<AlertModel>;
```

## Using Child Components

### Launching Child Components

```rust
struct AppModel {
    counter: u8,
    alert: AlertController,
}

impl SimpleComponent for AppModel {
    type Init = ();
    type Input = AppMsg;
    type Output = ();
    type Widgets = AppWidgets;
    type Root = gtk::Window;

    fn init(
        _: Self::Init,
        window: Self::Root,
        sender: ComponentSender<Self>,
    ) -> ComponentParts<Self> {
        // Launch child component
        let alert = AlertModel::builder()
            .transient_for(&window)
            .launch(AlertSettings {
                title: "Confirm".to_string(),
                message: "Are you sure?".to_string(),
                accept_label: "Yes".to_string(),
                cancel_label: "No".to_string(),
            })
            .forward(sender.input_sender(), |output| match output {
                AlertOutput::Confirmed => AppMsg::Confirmed,
                AlertOutput::Cancelled => AppMsg::Cancelled,
            });

        let model = AppModel {
            counter: 0,
            alert,
        };

        // ... create widgets
        let widgets = AppWidgets { /* ... */ };

        ComponentParts { model, widgets }
    }
}
```

### Communicating with Child Components

```rust
#[derive(Debug)]
enum AppMsg {
    ShowAlert,
    Confirmed,
    Cancelled,
}

impl SimpleComponent for AppModel {
    fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
        match msg {
            AppMsg::ShowAlert => {
                // Show the alert dialog
                self.alert.emit(AlertInput::Show);
            }
            AppMsg::Confirmed => {
                self.counter = 0;
            }
            AppMsg::Cancelled => {
                // User cancelled, do nothing
            }
        }
    }
}
```

## Complete Example

Here's a counter app with an alert dialog:

```rust
use gtk::prelude::*;
use relm4::{ComponentParts, ComponentSender, RelmApp, SimpleComponent, Controller};

struct AlertModel {
    visible: bool,
    message: String,
}

#[derive(Debug)]
enum AlertInput {
    Show,
    Hide,
    Accept,
}

#[derive(Debug)]
enum AlertOutput {
    Confirmed,
}

struct AlertWidgets {
    dialog: gtk::MessageDialog,
}

impl SimpleComponent for AlertModel {
    type Init = String;
    type Input = AlertInput;
    type Output = AlertOutput;
    type Widgets = AlertWidgets;
    type Root = gtk::Window;

    fn init_root() -> Self::Root {
        gtk::Window::new(gtk::WindowType::Popup)
    }

    fn init(
        message: Self::Init,
        _window: Self::Root,
        sender: ComponentSender<Self>,
    ) -> ComponentParts<Self> {
        let model = AlertModel {
            visible: false,
            message,
        };

        let dialog = gtk::MessageDialog::builder()
            .modal(true)
            .text("Confirm Action")
            .secondary_text(&model.message)
            .add_button("OK", gtk::ResponseType::Accept)
            .add_button("Cancel", gtk::ResponseType::Cancel)
            .build();

        let dialog_sender = sender.clone();
        dialog.connect_response(move |_, response| {
            if response == gtk::ResponseType::Accept {
                dialog_sender.input(AlertInput::Accept);
            } else {
                dialog_sender.input(AlertInput::Hide);
            }
        });

        let widgets = AlertWidgets { dialog };

        ComponentParts { model, widgets }
    }

    fn update(&mut self, msg: Self::Input, sender: ComponentSender<Self>) {
        match msg {
            AlertInput::Show => {
                self.visible = true;
                self.widgets.dialog.show();
            }
            AlertInput::Hide => {
                self.visible = false;
                self.widgets.dialog.hide();
            }
            AlertInput::Accept => {
                self.visible = false;
                self.widgets.dialog.hide();
                sender.output(AlertOutput::Confirmed).unwrap();
            }
        }
    }

    fn update_view(&self, widgets: &mut Self::Widgets, _sender: ComponentSender<Self>) {
        widgets.dialog.set_visible(self.visible);
    }
}

type AlertController = Controller<AlertModel>;

struct AppModel {
    counter: u8,
    alert: AlertController,
}

#[derive(Debug)]
enum AppMsg {
    Increment,
    ShowResetAlert,
    ResetConfirmed,
}

struct AppWidgets {
    label: gtk::Label,
    button: gtk::Button,
}

impl SimpleComponent for AppModel {
    type Init = ();
    type Input = AppMsg;
    type Output = ();
    type Widgets = AppWidgets;
    type Root = gtk::Window;

    fn init_root() -> Self::Root {
        gtk::Window::builder()
            .title("Counter with Alert")
            .default_width(300)
            .default_height(150)
            .build()
    }

    fn init(
        _: Self::Init,
        window: Self::Root,
        sender: ComponentSender<Self>,
    ) -> ComponentParts<Self> {
        let alert = AlertModel::builder()
            .transient_for(&window)
            .launch("Reset counter to 0?".to_string())
            .forward(sender.input_sender(), |_| AppMsg::ResetConfirmed);

        let model = AppModel {
            counter: 0,
            alert,
        };

        let vbox = gtk::Box::builder()
            .orientation(gtk::Orientation::Vertical)
            .spacing(10)
            .margin_all(20)
            .build();

        let label = gtk::Label::new(Some("Counter: 0"));
        let button = gtk::Button::with_label("Reset");

        window.set_child(Some(&vbox));
        vbox.append(&label);
        vbox.append(&button);

        button.connect_clicked(move |_| {
            sender.input(AppMsg::ShowResetAlert);
        });

        let widgets = AppWidgets { label, button };

        ComponentParts { model, widgets }
    }

    fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
        match msg {
            AppMsg::Increment => {
                self.counter = self.counter.wrapping_add(1);
            }
            AppMsg::ShowResetAlert => {
                self.alert.emit(AlertInput::Show);
            }
            AppMsg::ResetConfirmed => {
                self.counter = 0;
            }
        }
    }

    fn update_view(&self, widgets: &mut Self::Widgets, _sender: ComponentSender<Self>) {
        widgets.label.set_label(&format!("Counter: {}", self.counter));
    }
}

fn main() {
    let app = RelmApp::new("org.relm4.ChildComponentExample");
    app.run::<AppModel>(());
}
```

## Key Concepts

### Controller

`Controller<T>` wraps a component and provides methods to communicate with it:

```rust
let alert: AlertController = AlertModel::builder()
    .launch(initial_value);

// Send input message
alert.emit(AlertInput::Show);

// Send input with callback
alert.emit_with_callback(AlertInput::Show, |_| {
    println!("Done");
});
```

### transient_for

Sets the parent window for dialogs:

```rust
AlertModel::builder()
    .transient_for(&window)  // Alert appears on top of window
    .launch(initial_value)
```

### forward

Automatically convert child outputs to parent inputs:

```rust
.child
    .forward(sender.input_sender(), |child_output| {
        match child_output {
            ChildOutput::Done => ParentMsg::ChildDone,
        }
    });
```

## Summary

- Child components are reusable UI modules
- Use `Controller<T>` to manage child components
- Launch children with the builder pattern
- Use `emit()` to send messages to children
- Use `forward()` to convert child outputs to parent inputs
- Set `transient_for()` for dialog positioning
