# Data Binding

Data binding provides automatic synchronization between your model data and widget properties. When data changes, bound widgets update automatically.

## When to Use Data Binding

- Form inputs that should sync with model
- Real-time UI updates based on data
- Reducing manual `update_view()` code
- Complex widget property synchronization

## Basic Binding Example

```rust
use gtk::prelude::*;
use relm4::{
    ComponentParts, ComponentSender, RelmApp, SimpleComponent,
    binding::{Binding, StringBinding, BoolBinding, F64Binding},
};

struct AppModel {
    name: String,
    age: u32,
    is_active: bool,
}

#[derive(Debug)]
enum AppMsg {
    UpdateName(String),
    UpdateAge(u32),
    ToggleActive,
}

struct AppWidgets {
    name_entry: gtk::Entry,
    age_spin: gtk::SpinButton,
    active_switch: gtk::Switch,
}

impl SimpleComponent for AppModel {
    type Init = ();
    type Input = AppMsg;
    type Output = ();
    type Widgets = AppWidgets;
    type Root = gtk::Window;

    fn init_root() -> Self::Root {
        gtk::Window::builder()
            .title("Data Binding Example")
            .default_width(400)
            .default_height(300)
            .build()
    }

    fn init(
        _: Self::Init,
        window: Self::Root,
        sender: ComponentSender<Self>,
    ) -> ComponentParts<Self> {
        let model = AppModel {
            name: "John".to_string(),
            age: 30,
            is_active: true,
        };

        let vbox = gtk::Box::builder()
            .orientation(gtk::Orientation::Vertical)
            .spacing(10)
            .margin_all(20)
            .build();

        // Name field
        let name_label = gtk::Label::new(Some("Name:"));
        let name_entry = gtk::Entry::new();
        name_entry.set_text(&model.name);

        // Age field
        let age_label = gtk::Label::new(Some("Age:"));
        let age_adjustment = gtk::Adjustment::new(model.age as f64, 0.0, 150.0, 1.0, 0.0, 0.0);
        let age_spin = gtk::SpinButton::new(Some(&age_adjustment));
        age_spin.set_value(model.age as f64);

        // Active toggle
        let active_label = gtk::Label::new(Some("Active:"));
        let active_switch = gtk::Switch::new();
        active_switch.set_active(model.is_active);

        window.set_child(Some(&vbox));
        vbox.append(&name_label);
        vbox.append(&name_entry);
        vbox.append(&age_label);
        vbox.append(&age_spin);
        vbox.append(&active_label);
        vbox.append(&active_switch);

        let widgets = AppWidgets {
            name_entry,
            age_spin,
            active_switch,
        };

        ComponentParts { model, widgets }
    }

    fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
        match msg {
            AppMsg::UpdateName(name) => {
                self.name = name;
            }
            AppMsg::UpdateAge(age) => {
                self.age = age;
            }
            AppMsg::ToggleActive => {
                self.is_active = !self.is_active;
            }
        }
    }

    fn update_view(&self, widgets: &mut Self::Widgets, _sender: ComponentSender<Self>) {
        widgets.name_entry.set_text(&self.name);
        widgets.age_spin.set_value(self.age as f64);
        widgets.active_switch.set_active(self.is_active);
    }
}

fn main() {
    let app = RelmApp::new("org.relm4.DataBindingExample");
    app.run::<AppModel>(());
}
```

## Advanced Binding with Types

Relm4 provides typed bindings for common types:

```rust
use relm4::binding::{
    Binding, StringBinding, BoolBinding, F64Binding, I32Binding,
    ConnectBindingExt, RelmObjectExt,
};

struct FormModel {
    username: StringBinding,
    password: StringBinding,
    remember_me: BoolBinding,
    volume: F64Binding,
}

impl FormModel {
    fn new() -> Self {
        FormModel {
            username: StringBinding::new(""),
            password: StringBinding::new(""),
            remember_me: BoolBinding::default(),
            volume: F64Binding::new(50.0),
        }
    }
}

impl SimpleComponent for FormModel {
    // ...
    
    fn update(&mut self, msg: Self::Input, sender: ComponentSender<Self>) {
        match msg {
            FormMsg::Login => {
                let username = self.username.get();
                let password = self.password.get();
                let remember = *self.remember_me.guard();
                
                // Login logic
                println!("Login: {} (remember: {})", username, remember);
            }
        }
    }
}
```

## Binding Methods

### Create Bindings

```rust
// New binding with value
let binding = StringBinding::new("initial value");
let binding = BoolBinding::default();  // false
let binding = F64Binding::new(0.0);

// From existing value
let binding = StringBinding::from(&model.username);
```

### Get and Set Values

```rust
// Get value
let value = binding.get();

// Set value
binding.set("new value");

// With guard for mutable access
let mut guard = binding.guard();
*guard = "new value".to_string();
```

### Connect to Widgets

```rust
// Connect binding to entry
entry.connect_binding<StringBinding>(
    &model.username,
    |entry, value| entry.set_text(&value),
    |entry| entry.text().to_string(),
);

// Using the binding's built-in sync
model.username.bind_entry(&entry);
model.username.connect_changed(|value| {
    println!("Username changed: {}", value);
});
```

## Complete Binding Example

```rust
use gtk::prelude::*;
use relm4::{
    ComponentParts, ComponentSender, RelmApp, SimpleComponent,
    binding::{StringBinding, BoolBinding, F64Binding},
};

struct SettingsModel {
    username: StringBinding,
    email: StringBinding,
    notifications_enabled: BoolBinding,
    volume: F64Binding,
    theme: StringBinding,
}

impl SettingsModel {
    fn new() -> Self {
        SettingsModel {
            username: StringBinding::new("user123"),
            email: StringBinding::new("user@example.com"),
            notifications_enabled: BoolBinding::new(true),
            volume: F64Binding::new(75.0),
            theme: StringBinding::new("dark"),
        }
    }
}

#[derive(Debug)]
enum SettingsMsg {
    Save,
    Reset,
}

struct SettingsWidgets {
    username_entry: gtk::Entry,
    email_entry: gtk::Entry,
    notifications_switch: gtk::Switch,
    volume_scale: gtk::Scale,
    theme_combo: gtk::ComboBoxText,
    save_button: gtk::Button,
    status_label: gtk::Label,
}

impl SimpleComponent for SettingsModel {
    type Init = ();
    type Input = SettingsMsg;
    type Output = ();
    type Widgets = SettingsWidgets;
    type Root = gtk::Window;

    fn init_root() -> Self::Root {
        gtk::Window::builder()
            .title("Settings with Binding")
            .default_width(400)
            .default_height(400)
            .build()
    }

    fn init(
        _: Self::Init,
        window: Self::Root,
        sender: ComponentSender<Self>,
    ) -> ComponentParts<Self> {
        let model = SettingsModel::new();

        let vbox = gtk::Box::builder()
            .orientation(gtk::Orientation::Vertical)
            .spacing(10)
            .margin_all(20)
            .build();

        // Username
        let username_entry = gtk::Entry::new();
        username_entry.set_text(&model.username.get());
        let username_label = gtk::Label::new(Some("Username:"));
        username_label.set_halign(gtk::Align::Start);

        // Email
        let email_entry = gtk::Entry::new();
        email_entry.set_text(&model.email.get());
        let email_label = gtk::Label::new(Some("Email:"));
        email_label.set_halign(gtk::Align::Start);

        // Notifications
        let notifications_switch = gtk::Switch::new();
        notifications_switch.set_active(model.notifications_enabled.get());
        let notif_label = gtk::Label::new(Some("Notifications:"));
        notif_label.set_halign(gtk::Align::Start);

        // Volume
        let volume_scale = gtk::Scale::with_range(
            gtk::Orientation::Horizontal,
            0.0,
            100.0,
            1.0,
        );
        volume_scale.set_value(model.volume.get());
        let volume_label = gtk::Label::new(Some("Volume:"));
        volume_label.set_halign(gtk::Align::Start);

        // Theme
        let theme_combo = gtk::ComboBoxText::new();
        theme_combo.append(Some("light"), Some("Light"));
        theme_combo.append(Some("dark"), Some("Dark"));
        theme_combo.append(Some("system"), Some("System"));
        theme_combo.set_active_id(Some(&model.theme.get()));
        let theme_label = gtk::Label::new(Some("Theme:"));
        theme_label.set_halign(gtk::Align::Start);

        // Buttons
        let save_button = gtk::Button::with_label("Save");
        let reset_button = gtk::Button::with_label("Reset");

        // Status
        let status_label = gtk::Label::new(Some(""));
        status_label.add_css_class("success");

        window.set_child(Some(&vbox));
        vbox.append(&username_label);
        vbox.append(&username_entry);
        vbox.append(&email_label);
        vbox.append(&email_entry);
        vbox.append(&notif_label);
        vbox.append(&notifications_switch);
        vbox.append(&volume_label);
        vbox.append(&volume_scale);
        vbox.append(&theme_label);
        vbox.append(&theme_combo);
        vbox.append(&save_button);
        vbox.append(&reset_button);
        vbox.append(&status_label);

        save_button.connect_clicked(move |_| {
            sender.input(SettingsMsg::Save);
        });

        reset_button.connect_clicked(move |_| {
            sender.input(SettingsMsg::Reset);
        });

        let widgets = SettingsWidgets {
            username_entry,
            email_entry,
            notifications_switch,
            volume_scale,
            theme_combo,
            save_button,
            status_label,
        };

        ComponentParts { model, widgets }
    }

    fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
        match msg {
            SettingsMsg::Save => {
                println!("Saving settings...");
                println!("  username: {}", self.username.get());
                println!("  email: {}", self.email.get());
                println!("  notifications: {}", self.notifications_enabled.get());
                println!("  volume: {}", self.volume.get());
                println!("  theme: {}", self.theme.get());
            }
            SettingsMsg::Reset => {
                self.username.set("user123".to_string());
                self.email.set("user@example.com".to_string());
                self.notifications_enabled.set(true);
                self.volume.set(75.0);
                self.theme.set("dark".to_string());
            }
        }
    }

    fn update_view(&self, widgets: &mut Self::Widgets, _sender: ComponentSender<Self>) {
        widgets.username_entry.set_text(&self.username.get());
        widgets.email_entry.set_text(&self.email.get());
        widgets.notifications_switch.set_active(self.notifications_enabled.get());
        widgets.volume_scale.set_value(self.volume.get());
        widgets.theme_combo.set_active_id(Some(&self.theme.get()));
    }
}

fn main() {
    let app = RelmApp::new("org.relm4.SettingsBindingExample");
    app.run::<SettingsModel>(());
}
```

## Summary

- Data binding automatically syncs model data with widgets
- Use typed bindings like `StringBinding`, `BoolBinding`, `F64Binding`
- Create bindings with `Binding::new()` or `Binding::default()`
- Get values with `get()`, set with `set()`
- Use guards for mutable access
- Reduces boilerplate in `update_view()`
