# Message Brokers

Message brokers enable cross-component communication without direct dependencies. Components can subscribe to specific messages and react accordingly.

## When to Use Message Brokers

- Decouple components from each other
- Broadcast messages to multiple components
- Implement event-driven architectures
- Create plugin-like systems

## Basic Message Broker

```rust
use gtk::prelude::*;
use relm4::{
    ComponentParts, ComponentSender, RelmApp, SimpleComponent,
    MessageBroker,
};

#[derive(Debug, Clone)]
enum GlobalMsg {
    ThemeChanged(String),
    LanguageChanged(String),
    UserLoggedIn(String),
    UserLoggedOut,
}

static BROKER: MessageBroker<GlobalMsg> = MessageBroker::new();

struct ThemeComponent;

impl SimpleComponent for ThemeComponent {
    type Init = ();
    type Input = GlobalMsg;
    type Output = ();
    type Widgets = ();
    type Root = ();

    fn init_root() -> Self::Root {}

    fn init(
        _: Self::Init,
        _: Self::Root,
        sender: ComponentSender<Self>,
    ) -> ComponentParts<Self> {
        // Subscribe to theme messages
        BROKER.subscribe(sender.input_sender(), |msg| {
            matches!(msg, GlobalMsg::ThemeChanged(_))
        });

        let model = ThemeComponent;
        ComponentParts { model, widgets: () }
    }

    fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
        match msg {
            GlobalMsg::ThemeChanged(theme) => {
                println!("Theme component: Applying theme '{}'", theme);
                // Apply theme changes
            }
            _ => {}
        }
    }
}

struct NotificationComponent;

impl SimpleComponent for NotificationComponent {
    type Init = ();
    type Input = GlobalMsg;
    type Output = ();
    type Widgets = ();
    type Root = ();

    fn init_root() -> Self::Root {}

    fn init(
        _: Self::Init,
        _: Self::Root,
        sender: ComponentSender<Self>,
    ) -> ComponentParts<Self> {
        // Subscribe to all user-related messages
        BROKER.subscribe(sender.input_sender(), |msg| {
            matches!(
                msg,
                GlobalMsg::UserLoggedIn(_) | GlobalMsg::UserLoggedOut
            )
        });

        let model = NotificationComponent;
        ComponentParts { model, widgets: () }
    }

    fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
        match msg {
            GlobalMsg::UserLoggedIn(user) => {
                println!("Notification: User '{}' logged in", user);
            }
            GlobalMsg::UserLoggedOut => {
                println!("Notification: User logged out");
            }
            _ => {}
        }
    }
}
```

## Broadcasting Messages

Any component can broadcast messages:

```rust
struct AppModel {
    broker_sender: relm4::Sender<GlobalMsg>,
}

impl SimpleComponent for AppModel {
    // ...

    fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
        match msg {
            AppMsg::ChangeTheme(theme) => {
                // Broadcast to all subscribers
                self.broker_sender.send(GlobalMsg::ThemeChanged(theme)).unwrap();
            }
            AppMsg::Login(user) => {
                self.broker_sender.send(GlobalMsg::UserLoggedIn(user)).unwrap();
            }
            AppMsg::Logout => {
                self.broker_sender.send(GlobalMsg::UserLoggedOut).unwrap();
            }
        }
    }
}
```

## Complete Example

```rust
use gtk::prelude::*;
use relm4::{
    ComponentParts, ComponentSender, RelmApp, SimpleComponent,
    MessageBroker,
};

#[derive(Debug, Clone)]
enum AppEvent {
    ThemeChanged(String),
    LanguageChanged(String),
    DataRefreshed(Vec<String>),
}

static GLOBAL_BROKER: MessageBroker<AppEvent> = MessageBroker::new();

struct ThemeManager;

impl SimpleComponent for ThemeManager {
    type Init = ();
    type Input = AppEvent;
    type Output = ();
    type Widgets = ();
    type Root = ();

    fn init_root() -> Self::Root {}

    fn init(
        _: Self::Init,
        _: Self::Root,
        sender: ComponentSender<Self>,
    ) -> ComponentParts<Self> {
        BROKER.subscribe(sender.input_sender(), |msg| {
            matches!(msg, AppEvent::ThemeChanged(_))
        });

        ComponentParts {
            model: ThemeManager,
            widgets: (),
        }
    }

    fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
        if let AppEvent::ThemeChanged(theme) = msg {
            println!("[ThemeManager] Applying theme: {}", theme);
        }
    }
}

struct LanguageManager;

impl SimpleComponent for LanguageManager {
    type Init = ();
    type Input = AppEvent;
    type Output = ();
    type Widgets = ();
    type Root = ();

    fn init_root() -> Self::Root {}

    fn init(
        _: Self::Init,
        _: Self::Root,
        sender: ComponentSender<Self>,
    ) -> ComponentParts<Self> {
        BROKER.subscribe(sender.input_sender(), |msg| {
            matches!(msg, AppEvent::LanguageChanged(_))
        });

        ComponentParts {
            model: LanguageManager,
            widgets: (),
        }
    }

    fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
        if let AppEvent::LanguageChanged(lang) = msg {
            println!("[LanguageManager] Changing language to: {}", lang);
        }
    }
}

struct DataSyncManager;

impl SimpleComponent for DataSyncManager {
    type Init = ();
    type Input = AppEvent;
    type Output = ();
    type Widgets = ();
    type Root = ();

    fn init_root() -> Self::Root {}

    fn init(
        _: Self::Init,
        _: Self::Root,
        sender: ComponentSender<Self>,
    ) -> ComponentParts<Self> {
        BROKER.subscribe(sender.input_sender(), |msg| {
            matches!(msg, AppEvent::DataRefreshed(_))
        });

        ComponentParts {
            model: DataSyncManager,
            widgets: (),
        }
    }

    fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
        if let AppEvent::DataRefreshed(items) = msg {
            println!("[DataSyncManager] Received {} items", items.len());
        }
    }
}

struct AppModel {
    broker_sender: relm4::Sender<AppEvent>,
}

#[derive(Debug)]
enum AppMsg {
    SetTheme(String),
    SetLanguage(String),
    RefreshData,
}

struct AppWidgets {
    theme_combo: gtk::ComboBoxText,
    lang_combo: gtk::ComboBoxText,
    refresh_button: gtk::Button,
    log_view: gtk::TextView,
}

impl SimpleComponent for AppModel {
    type Init = ();
    type Input = AppMsg;
    type Output = ();
    type Widgets = AppWidgets;
    type Root = gtk::Window;

    fn init_root() -> Self::Root {
        gtk::Window::builder()
            .title("Message Broker Example")
            .default_width(500)
            .default_height(400)
            .build()
    }

    fn init(
        _: Self::Init,
        window: Self::Root,
        sender: ComponentSender<Self>,
    ) -> ComponentParts<Self> {
        let (broker_sender, _) = relm4::channel::<AppEvent>(|_| {});

        let model = AppModel {
            broker_sender: broker_sender.clone(),
        };

        // Launch subscriber components (headless)
        ThemeManager::builder()
            .launch(())
            .detach();

        LanguageManager::builder()
            .launch(())
            .detach();

        DataSyncManager::builder()
            .launch(())
            .detach();

        let vbox = gtk::Box::builder()
            .orientation(gtk::Orientation::Vertical)
            .spacing(10)
            .margin_all(20)
            .build();

        // Theme selector
        let theme_label = gtk::Label::new(Some("Theme:"));
        let theme_combo = gtk::ComboBoxText::new();
        theme_combo.append(None, "Select...");
        theme_combo.append(Some("light"), "Light");
        theme_combo.append(Some("dark"), "Dark");
        theme_combo.append(Some("blue"), "Blue");

        // Language selector
        let lang_label = gtk::Label::new(Some("Language:"));
        let lang_combo = gtk::ComboBoxText::new();
        lang_combo.append(None, "Select...");
        lang_combo.append(Some("en"), "English");
        lang_combo.append(Some("de"), "German");
        lang_combo.append(Some("fr"), "French");

        // Refresh button
        let refresh_button = gtk::Button::with_label("Refresh Data");

        // Log view
        let scrolled = gtk::ScrolledWindow::new();
        let log_view = gtk::TextView::new();
        log_view.set_editable(false);
        log_view.set_wrap_mode(gtk::WrapMode::Word);
        scrolled.set_child(Some(&log_view));
        scrolled.set_min_content_height(200);

        window.set_child(Some(&vbox));
        vbox.append(&theme_label);
        vbox.append(&theme_combo);
        vbox.append(&lang_label);
        vbox.append(&lang_combo);
        vbox.append(&refresh_button);
        vbox.append(&scrolled);

        let theme_combo_clone = theme_combo.clone();
        theme_combo.connect_changed(move |combo| {
            if let Some(theme) = combo.active_id() {
                if theme != "" {
                    broker_sender.send(AppEvent::ThemeChanged(theme.to_string())).unwrap();
                    theme_combo_clone.set_active_id(None);
                }
            }
        });

        let lang_combo_clone = lang_combo.clone();
        lang_combo.connect_changed(move |combo| {
            if let Some(lang) = combo.active_id() {
                if lang != "" {
                    broker_sender.send(AppEvent::LanguageChanged(lang.to_string())).unwrap();
                    lang_combo_clone.set_active_id(None);
                }
            }
        });

        let broker_sender_clone = broker_sender;
        refresh_button.connect_clicked(move |_| {
            let data = vec!["Item 1".to_string(), "Item 2".to_string()];
            broker_sender_clone.send(AppEvent::DataRefreshed(data)).unwrap();
        });

        let widgets = AppWidgets {
            theme_combo,
            lang_combo,
            refresh_button,
            log_view,
        };

        ComponentParts { model, widgets }
    }

    fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
        match msg {
            AppMsg::SetTheme(theme) => {
                self.broker_sender.send(AppEvent::ThemeChanged(theme)).unwrap();
            }
            AppMsg::SetLanguage(lang) => {
                self.broker_sender.send(AppEvent::LanguageChanged(lang)).unwrap();
            }
            AppMsg::RefreshData => {
                let data = vec!["Data 1".to_string(), "Data 2".to_string(), "Data 3".to_string()];
                self.broker_sender.send(AppEvent::DataRefreshed(data)).unwrap();
            }
        }
    }
}

fn main() {
    let app = RelmApp::new("org.relm4.MessageBrokerExample");
    app.run::<AppModel>(());
}
```

## Key Concepts

### Static Broker

```rust
static BROKER: MessageBroker<GlobalMsg> = MessageBroker::new();
```

A static message broker can be accessed from anywhere in your application.

### Subscribe

```rust
BROKER.subscribe(sender.input_sender(), filter);
```

Subscribe a component to messages. The filter function determines which messages are received.

### Broadcast

```rust
broker_sender.send(message).unwrap();
```

Send a message to all subscribers.

### Filter Function

```rust
BROKER.subscribe(sender.input_sender(), |msg| {
    matches!(msg, AppEvent::ThemeChanged(_))
});
```

Only receive `ThemeChanged` messages.

## Summary

- Message brokers enable decoupled communication
- Components subscribe to specific message types
- Messages are broadcast to all subscribers
- Use filters to receive only relevant messages
- Perfect for global state and cross-cutting concerns
