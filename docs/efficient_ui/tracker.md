# Tracker

Trackers help you efficiently update only the widgets that actually changed. Instead of updating all widgets on every change, trackers let you know exactly which fields were modified.

## The Tracker Crate

Relm4 uses the [`tracker`](https://docs.rs/tracker/latest/tracker/) crate for change detection:

```toml
[dependencies]
tracker = "0.2"
```

## How Trackers Work

The `#[tracker::track]` macro generates:

- **Getters**: `get_field_name()` - Immutable access
- **Setters**: `set_field_name(value)` - Sets and marks as changed
- **Mutators**: `get_mut_field_name()` - Mutable access, marks as changed
- **Updaters**: `update_field_name(closure)` - Apply function, marks as changed
- **Change detection**: `changed(FieldStruct::field_name())` - Check if changed
- **Reset**: `reset()` - Clear all change flags

## Basic Example

```rust
use gtk::prelude::*;
use relm4::{ComponentParts, ComponentSender, RelmApp, SimpleComponent};
use tracker::track;

#[tracker::track]
struct AppModel {
    counter: u32,
    message: String,
    is_active: bool,
}

#[derive(Debug)]
enum AppMsg {
    Increment,
    SetMessage(String),
    Toggle,
}

struct AppWidgets {
    counter_label: gtk::Label,
    message_label: gtk::Label,
    status_label: gtk::Label,
}

impl SimpleComponent for AppModel {
    type Init = ();
    type Input = AppMsg;
    type Output = ();
    type Widgets = AppWidgets;
    type Root = gtk::Window;

    fn init_root() -> Self::Root {
        gtk::Window::builder()
            .title("Tracker Example")
            .default_width(300)
            .default_height(200)
            .build()
    }

    fn init(
        _: Self::Init,
        window: Self::Root,
        sender: ComponentSender<Self>,
    ) -> ComponentParts<Self> {
        let model = AppModel {
            counter: 0,
            message: String::from("Hello!"),
            is_active: true,
            tracker: 0,  // Required by the macro
        };

        let vbox = gtk::Box::builder()
            .orientation(gtk::Orientation::Vertical)
            .spacing(10)
            .margin_all(20)
            .build();

        let counter_label = gtk::Label::new(Some("Counter: 0"));
        let message_label = gtk::Label::new(Some("Hello!"));
        let status_label = gtk::Label::new(Some("Active: yes"));

        let inc_button = gtk::Button::with_label("Increment");
        let msg_button = gtk::Button::with_label("Change Message");
        let toggle_button = gtk::Button::with_label("Toggle Status");

        window.set_child(Some(&vbox));
        vbox.append(&counter_label);
        vbox.append(&message_label);
        vbox.append(&status_label);
        vbox.append(&inc_button);
        vbox.append(&msg_button);
        vbox.append(&toggle_button);

        inc_button.connect_clicked(move |_| {
            sender.input(AppMsg::Increment);
        });

        msg_button.connect_clicked(move |_| {
            sender.input(AppMsg::SetMessage("Changed!".to_string()));
        });

        toggle_button.connect_clicked(move |_| {
            sender.input(AppMsg::Toggle);
        });

        let widgets = AppWidgets {
            counter_label,
            message_label,
            status_label,
        };

        ComponentParts { model, widgets }
    }

    fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
        self.reset();  // Important: reset before making changes

        match msg {
            AppMsg::Increment => {
                self.set_counter(self.counter + 1);
            }
            AppMsg::SetMessage(msg) => {
                self.set_message(msg);
            }
            AppMsg::Toggle => {
                self.set_is_active(!self.is_active);
            }
        }
    }

    fn update_view(&self, widgets: &mut Self::Widgets, _sender: ComponentSender<Self>) {
        if self.changed(AppModel::counter()) {
            widgets.counter_label.set_label(&format!("Counter: {}", self.counter));
        }

        if self.changed(AppModel::message()) {
            widgets.message_label.set_label(&self.message);
        }

        if self.changed(AppModel::is_active()) {
            widgets.status_label.set_label(&format!("Active: {}", if self.is_active { "yes" } else { "no" }));
        }
    }
}

fn main() {
    let app = RelmApp::new("org.relm4.TrackerExample");
    app.run::<AppModel>(());
}
```

## Important Rules

### 1. Always Reset Before Making Changes

```rust
fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
    self.reset();  // Clear previous change flags

    match msg {
        // ... make changes using setters
    }
}
```

### 2. Use Setter Methods

```rust
// Use this:
self.set_counter(42);

// Not this:
self.counter = 42;

// Except when you intentionally don't want to track:
self.reset();
self.counter = 42;  // This won't be tracked
```

### 3. Check Changes in update_view

```rust
fn update_view(&self, widgets: &mut Self::Widgets, _sender: ComponentSender<Self>) {
    if self.changed(AppModel::counter()) {
        widgets.label.set_label(&self.counter.to_string());
    }
}
```

## Tracker Patterns

### Conditional Updates

```rust
fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
    self.reset();

    match msg {
        AppMsg::SubmitForm => {
            if self.validate_form() {
                self.set_status("Form valid!".to_string());
                self.set_can_submit(true);
            } else {
                self.set_status("Form invalid".to_string());
                self.set_can_submit(false);
            }
        }
    }
}

fn update_view(&self, widgets: &mut Self::Widgets, _sender: ComponentSender<Self>) {
    if self.changed(AppModel::status()) {
        widgets.status_label.set_label(&self.status);
        widgets.status_label.add_css_class("status-active");
    }

    widgets.submit_button.set_sensitive(self.can_submit);
}
```

### Multiple Related Fields

```rust
#[tracker::track]
struct UserProfile {
    first_name: String,
    last_name: String,
    full_name: String,  // Computed from first + last
}

impl UserProfile {
    fn update_names(&mut self, first: String, last: String) {
        self.set_first_name(first);
        self.set_last_name(last);
        self.set_full_name(format!("{} {}", self.first_name, self.last_name));
    }
}

fn update_view(&self, widgets: &mut Self::Widgets, _sender: ComponentSender<Self>) {
    // If either name changed, update the full name display
    if self.changed(AppModel::first_name()) || self.changed(AppModel::last_name()) {
        widgets.full_name_label.set_label(&self.full_name);
    }
}
```

### Nested Tracking

```rust
use tracker::track;

#[tracker::track]
struct Settings {
    #[track(ignore)]
    database_url: String,  // Won't be tracked

    theme: Theme,
}

#[tracker::track]
struct Theme {
    primary_color: String,
    secondary_color: String,
}

fn update_view(&self, widgets: &mut Self::Widgets, _sender: ComponentSender<Self>) {
    if self.changed(Settings::theme()) {
        if self.theme.changed(Theme::primary_color()) {
            widgets.set_primary_color(&self.theme.primary_color);
        }
        if self.theme.changed(Theme::secondary_color()) {
            widgets.set_secondary_color(&self.theme.secondary_color);
        }
    }
}
```

## Summary

- Trackers detect which specific fields changed
- Use `#[tracker::track]` macro on your model struct
- Always call `self.reset()` at the start of `update()`
- Use setter methods (`set_field(value)`) to mark changes
- Check changes in `update_view()` before updating widgets
- Use `#[track(ignore)]` for fields you don't want to track
