# Actions and Menus

Actions provide a declarative way to define and handle user interactions. They're integrated with GTK's action system and support keyboard shortcuts, menu items, and toolbars.

## Basic Actions

### Defining Actions

```rust
use gtk::prelude::*;
use relm4::{
    ComponentParts, ComponentSender, RelmApp, SimpleComponent,
    actions::{RelmAction, RelmActionGroup, AccelsPlus},
    new_action_group, new_stateless_action, new_stateful_action,
};

new_action_group!(WindowActionGroup, "win");

new_stateless_action!(ExampleAction, WindowActionGroup, "example");
new_stateful_action!(CounterAction, WindowActionGroup, "counter", i32, i32);

struct AppModel {
    counter: i32,
}

#[derive(Debug)]
enum AppMsg {
    Increment,
    Decrement,
}

struct AppWidgets {
    label: gtk::Label,
    menu_button: gtk::MenuButton,
}

impl SimpleComponent for AppModel {
    type Init = ();
    type Input = AppMsg;
    type Output = ();
    type Widgets = AppWidgets;
    type Root = gtk::Window;

    fn init_root() -> Self::Root {
        gtk::Window::builder()
            .title("Actions Example")
            .default_width(300)
            .default_height(200)
            .build()
    }

    fn init(
        _: Self::Init,
        window: Self::Root,
        sender: ComponentSender<Self>,
    ) -> ComponentParts<Self> {
        let model = AppModel { counter: 0 };

        let vbox = gtk::Box::builder()
            .orientation(gtk::Orientation::Vertical)
            .spacing(10)
            .margin_all(20)
            .build();

        let label = gtk::Label::new(Some("Counter: 0"));
        let menu_button = gtk::MenuButton::with_label("Menu");

        // Create menu
        let menu = gtk::Menu::new();
        menu.append(&gtk::MenuItem::with_label("Increment"));
        menu.append(&gtk::MenuItem::with_label("Decrement"));
        menu_button.set_menu_model(Some(&menu));

        window.set_child(Some(&vbox));
        vbox.append(&label);
        vbox.append(&menu_button);

        // Create action group
        let mut action_group = RelmActionGroup::<WindowActionGroup>::new();

        // Create actions
        let inc_action = RelmAction::new_stateless(move |_| {
            sender.input(AppMsg::Increment);
        });

        let dec_action = RelmAction::new_stateless(move |_| {
            sender.input(AppMsg::Decrement);
        });

        action_group.add_action(&inc_action);
        action_group.add_action(&dec_action);

        // Register action group
        action_group.register_for_widget(&window);

        let widgets = AppWidgets { label, menu_button };

        ComponentParts { model, widgets }
    }

    fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
        match msg {
            AppMsg::Increment => {
                self.counter += 1;
            }
            AppMsg::Decrement => {
                self.counter -= 1;
            }
        }
    }

    fn update_view(&self, widgets: &mut Self::Widgets, _sender: ComponentSender<Self>) {
        widgets.label.set_label(&format!("Counter: {}", self.counter));
    }
}

fn main() {
    let app = RelmApp::new("org.relm4.ActionsExample");
    app.run::<AppModel>(());
}
```

## Stateful Actions

Actions can maintain state:

```rust
new_stateful_action!(ToggleAction, WindowActionGroup, "toggle", bool, bool);

let toggle_action = RelmAction::new_stateful_with_target_value(
    &false,  // Initial state
    |_, state, _| {
        *state = !*state;
        println!("Toggle state: {}", *state);
    },
);

let action = RelmAction::new_stateful_with_target_value(
    &0i32,  // Initial value
    |_, state, value| {
        *state += value;
        println!("New state: {}", *state);
    },
);

// Trigger with specific value
action.set_parameter(&5i32);
action.activate();
```

## Action with Parameter Types

```rust
use relm4::actions::ActionablePlus;

new_action_group!(AppActionGroup, "app");

// Action that takes a u8 parameter
new_stateless_action!(SetValueAction, AppActionGroup, "set-value");

struct AppWidgets {
    increment_button: gtk::Button,
}

impl SimpleComponent for AppModel {
    fn init(...) -> ComponentParts<Self> {
        let increment_button = gtk::Button::with_label("Add 5");
        
        // Set action with parameter
        increment_button.set_action::<SetValueAction>(5u8);
        
        let action = RelmAction::<SetValueAction>::new_stateless(move |_| {
            sender.input(AppMsg::Add(5));
        });
        
        AppWidgets { increment_button }
    }
}
```

## Keyboard Shortcuts

Set keyboard accelerators for actions:

```rust
use gtk::accelerator_parse;

fn main() {
    let app = RelmApp::new("org.relm4.ActionsExample");
    
    // Get the application
    let application = relm4::main_application();
    
    // Set accelerators
    application.set_accelerators_for_action::<WindowActionGroup>("example", &["<Primary>q"]);
    application.set_accelerators_for_action::<WindowActionGroup>("counter", &["<Primary>plus"]);
    
    app.run::<AppModel>(());
}
```

## Menu Models

Use GMenu for declarative menus:

```rust
use gtk::gio;

fn create_menu() -> gio::Menu {
    let menu = gio::Menu::new();
    
    let section = gio::Menu::new();
    section.append("Increment", "win.example");
    section.append("Decrement", "win.example");
    
    menu.append_section(None, &section);
    
    let other = gio::Menu::new();
    other.append("Settings", "app.settings");
    other.append("Help", "app.help");
    
    menu.append_section(None, &other);
    
    menu
}

// Usage
let menu = create_menu();
menu_button.set_menu_model(Some(&menu));
```

## Complete Example with Actions

```rust
use gtk::prelude::*;
use relm4::{
    ComponentParts, ComponentSender, RelmApp, SimpleComponent,
    actions::{RelmAction, RelmActionGroup},
    new_action_group, new_stateless_action,
};

new_action_group!(AppActionGroup, "app");
new_stateless_action!(QuitAction, AppActionGroup, "quit");
new_stateless_action!(AboutAction, AppActionGroup, "about");

new_action_group!(WinActionGroup, "win");
new_stateless_action!(NewAction, WinActionGroup, "new");
new_stateless_action!(OpenAction, WinActionGroup, "open");
new_stateless_action!(SaveAction, WinActionGroup, "save");

struct AppModel {
    text: String,
    modified: bool,
}

#[derive(Debug)]
enum AppMsg {
    New,
    Open,
    Save,
    UpdateText(String),
    Quit,
    About,
}

struct AppWidgets {
    text_view: gtk::TextView,
}

impl SimpleComponent for AppModel {
    type Init = ();
    type Input = AppMsg;
    type Output = ();
    type Widgets = AppWidgets;
    type Root = gtk::ApplicationWindow;

    fn init_root() -> Self::Root {
        gtk::ApplicationWindow::builder()
            .title("Text Editor")
            .default_width(600)
            .default_height(400)
            .build()
    }

    fn init(
        _: Self::Init,
        window: Self::Root,
        sender: ComponentSender<Self>,
    ) -> ComponentParts<Self> {
        let model = AppModel {
            text: String::new(),
            modified: false,
        };

        // Create menu bar
        let menu_bar = gtk::MenuBar::new();

        // File menu
        let file_menu = gtk::Menu::new();
        let file_item = gtk::MenuItem::with_label("File");
        file_item.set_submenu(Some(&file_menu));

        let new_item = gtk::MenuItem::with_label("New");
        let open_item = gtk::MenuItem::with_label("Open");
        let save_item = gtk::MenuItem::with_label("Save");
        let quit_item = gtk::MenuItem::with_label("Quit");

        file_menu.append(&new_item);
        file_menu.append(&open_item);
        file_menu.append(&save_item);
        file_menu.append(&quit_item);

        // Help menu
        let help_menu = gtk::Menu::new();
        let help_item = gtk::MenuItem::with_label("Help");
        help_item.set_submenu(Some(&help_menu));

        let about_item = gtk::MenuItem::with_label("About");
        help_menu.append(&about_item);

        menu_bar.append(&file_item);
        menu_bar.append(&help_item);

        // Text view
        let scrolled = gtk::ScrolledWindow::new();
        let text_view = gtk::TextView::new();
        text_view.set_wrap_mode(gtk::WrapMode::Word);
        scrolled.set_child(Some(&text_view));

        // Layout
        let vbox = gtk::Box::builder()
            .orientation(gtk::Orientation::Vertical)
            .build();

        vbox.append(&menu_bar);
        vbox.append(&scrolled);

        window.set_child(Some(&vbox));

        // Create action groups
        let mut app_actions = RelmActionGroup::<AppActionGroup>::new();
        let mut win_actions = RelmActionGroup::<WinActionGroup>::new();

        // App actions
        let quit_action = RelmAction::new_stateless(move |_| {
            sender.input(AppMsg::Quit);
        });
        let about_action = RelmAction::new_stateless(move |_| {
            sender.input(AppMsg::About);
        });
        app_actions.add_action(&quit_action);
        app_actions.add_action(&about_action);
        app_actions.register_for_widget(&window);

        // Win actions
        let new_action = RelmAction::new_stateless(move |_| {
            sender.input(AppMsg::New);
        });
        let open_action = RelmAction::new_stateless(move |_| {
            sender.input(AppMsg::Open);
        });
        let save_action = RelmAction::new_stateless(move |_| {
            sender.input(AppMsg::Save);
        });
        win_actions.add_action(&new_action);
        win_actions.add_action(&open_action);
        win_actions.add_action(&save_action);
        win_actions.register_for_widget(&window);

        // Connect menu items to actions
        new_item.set_action_name(Some("win.new"));
        open_item.set_action_name(Some("win.open"));
        save_item.set_action_name(Some("win.save"));
        quit_item.set_action_name(Some("app.quit"));
        about_item.set_action_name(Some("app.about"));

        // Set keyboard shortcuts
        let app = relm4::main_application();
        app.set_accelerators_for_action::<WinActionGroup>("new", &["<Primary>n"]);
        app.set_accelerators_for_action::<WinActionGroup>("open", &["<Primary>o"]);
        app.set_accelerators_for_action::<WinActionGroup>("save", &["<Primary>s"]);
        app.set_accelerators_for_action::<AppActionGroup>("quit", &["<Primary>q"]);

        let widgets = AppWidgets { text_view };

        ComponentParts { model, widgets }
    }

    fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
        match msg {
            AppMsg::New => {
                self.text.clear();
                self.modified = false;
            }
            AppMsg::Open => {
                // Open file dialog
            }
            AppMsg::Save => {
                // Save file
                self.modified = false;
            }
            AppMsg::UpdateText(text) => {
                self.text = text;
                self.modified = true;
            }
            AppMsg::Quit => {
                relm4::main_application().quit();
            }
            AppMsg::About => {
                println!("About dialog");
            }
        }
    }
}

fn main() {
    let app = RelmApp::new("org.relm4.EditorExample");
    app.run::<AppModel>(());
}
```

## Summary

- Actions provide declarative interaction handling
- Use `new_action_group!` for action groups
- Use `new_stateless_action!` or `new_stateful_action!` for actions
- Actions integrate with GTK menus and keyboard shortcuts
- `RelmActionGroup` manages action registration
- Set accelerators with `set_accelerators_for_action()`
