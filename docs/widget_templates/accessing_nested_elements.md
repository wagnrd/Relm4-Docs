# Accessing Nested Template Elements

When you have deeply nested widget structures in templates, you can still access specific elements for updates and signal connections.

## The Problem

Templates can have deeply nested widget trees, but you often need to access specific widgets directly:

```
Root Box
└── Header Box
    ├── Title Label
    └── Subtitle Label
├── Body Box
│   └── Content Label
└── Footer Box
    ├── Cancel Button
    └── Confirm Button
```

## Accessing Nested Elements

### Direct Access

Each template child is directly accessible:

```rust
struct MyTemplate {
    #[template_child]
    title: gtk::Label,
    
    #[template_child]
    content: gtk::Label,
    
    #[template_child]
    confirm_button: gtk::Button,
}

// Direct access
template.title.set_label("New Title");
template.confirm_button.set_sensitive(false);
```

### Finding Nested Widgets

For deeper nesting, use GTK's widget lookup:

```rust
fn find_child<T: IsA<gtk::Widget>>(parent: &gtk::Widget, name: &str) -> Option<T> {
    if parent.name() == name {
        return parent.downcast().ok();
    }
    
    if let Some(container) = parent.downcast_ref::<gtk::Container>() {
        for child in container.children() {
            if let Some(found) = find_child::<T>(&child, name) {
                return Some(found);
            }
        }
    }
    None
}

// Usage
let button: gtk::Button = find_child(&root, "confirm-button").unwrap();
button.connect_clicked(|_| { /* ... */ });
```

### By Name

Set names on widgets to find them later:

```rust
let title = gtk::Label::new(Some("My Title"));
title.set_name("title-label");  // Set name for lookup

let button = gtk::Button::with_label("OK");
button.set_name("confirm-button");

// Find by name
let title = root.find_child::<gtk::Label>("title-label").unwrap();
let button = root.find_child::<gtk::Button>("confirm-button").unwrap();
```

## Complete Example

```rust
use gtk::prelude::*;
use relm4::{ComponentParts, ComponentSender, RelmApp, SimpleComponent};

struct DialogTemplate {
    root: gtk::Box,
    title: gtk::Label,
    message: gtk::Label,
    cancel_btn: gtk::Button,
    confirm_btn: gtk::Button,
}

impl DialogTemplate {
    fn new() -> Self {
        let root = gtk::Box::builder()
            .orientation(gtk::Orientation::Vertical)
            .spacing(10)
            .margin_all(20)
            .build();

        let header = gtk::Box::builder()
            .orientation(gtk::Orientation::Vertical)
            .spacing(4)
            .build();

        let title = gtk::Label::new(None);
        title.add_css_class("title");
        title.set_halign(gtk::Align::Start);
        title.set_name("dialog-title");

        let message = gtk::Label::new(None);
        message.set_halign(gtk::Align::Start);
        message.set_wrap(true);
        message.set_name("dialog-message");

        header.append(&title);
        header.append(&message);

        let button_box = gtk::Box::builder()
            .orientation(gtk::Orientation::Horizontal)
            .spacing(8)
            .halign(gtk::Align::End)
            .build();

        let cancel_btn = gtk::Button::with_label("Cancel");
        cancel_btn.set_name("cancel-button");
        cancel_btn.add_css_class("flat");

        let confirm_btn = gtk::Button::with_label("OK");
        confirm_btn.set_name("confirm-button");
        confirm_btn.add_css_class("suggested-action");

        button_box.append(&cancel_btn);
        button_box.append(&confirm_btn);

        root.append(&header);
        root.append(&button_box);

        DialogTemplate {
            root,
            title,
            message,
            cancel_btn,
            confirm_btn,
        }
    }

    fn set_title(&self, title: &str) {
        self.title.set_label(title);
    }

    fn set_message(&self, message: &str) {
        self.message.set_label(message);
    }

    fn set_confirm_label(&self, label: &str) {
        self.confirm_btn.set_label(label);
    }

    fn on_cancel<F>(&self, f: F)
    where
        F: Fn() + 'static,
    {
        self.cancel_btn.connect_clicked(move |_| f());
    }

    fn on_confirm<F>(&self, f: F)
    where
        F: Fn() + 'static,
    {
        self.confirm_btn.connect_clicked(move |_| f());
    }

    fn widget(&self) -> &gtk::Widget {
        self.root.as_ref()
    }
}

struct AppModel {
    dialog: DialogTemplate,
}

#[derive(Debug)]
enum AppMsg {
    ShowDialog,
    Confirm,
    Cancel,
}

struct AppWidgets {
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
            .title("Nested Elements Example")
            .default_width(400)
            .default_height(200)
            .build()
    }

    fn init(
        _: Self::Init,
        window: Self::Root,
        sender: ComponentSender<Self>,
    ) -> ComponentParts<Self> {
        let dialog = DialogTemplate::new();

        dialog.set_title("Confirm Action");
        dialog.set_message("Are you sure you want to proceed?");
        dialog.set_confirm_label("Yes, Proceed");

        dialog.on_cancel(move || {
            sender.input(AppMsg::Cancel);
        });

        dialog.on_confirm(move || {
            sender.input(AppMsg::Confirm);
        });

        let model = AppModel { dialog };

        let vbox = gtk::Box::builder()
            .orientation(gtk::Orientation::Vertical)
            .spacing(10)
            .margin_all(20)
            .build();

        let button = gtk::Button::with_label("Show Dialog");

        button.connect_clicked(move |_| {
            sender.input(AppMsg::ShowDialog);
        });

        // Initially hide the dialog
        model.dialog.widget().set_visible(false);

        window.set_child(Some(&vbox));
        vbox.append(&button);
        vbox.append(model.dialog.widget());

        let widgets = AppWidgets { button };

        ComponentParts { model, widgets }
    }

    fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
        match msg {
            AppMsg::ShowDialog => {
                self.dialog.widget().set_visible(true);
            }
            AppMsg::Confirm => {
                self.dialog.widget().set_visible(false);
                println!("Confirmed!");
            }
            AppMsg::Cancel => {
                self.dialog.widget().set_visible(false);
                println!("Cancelled");
            }
        }
    }
}

fn main() {
    let app = RelmApp::new("org.relm4.NestedElementsExample");
    app.run::<AppModel>(());
}
```

## Finding Elements by Type

You can also find elements by type when multiple widgets share the same type:

```rust
fn find_first<T: IsA<gtk::Widget> + Cast>(parent: &gtk::Widget) -> Option<T> {
    if parent.is::<T>() {
        return parent.downcast().ok();
    }
    
    if let Some(container) = parent.downcast_ref::<gtk::Container>() {
        for child in container.children() {
            if let Some(found) = find_first::<T>(&child) {
                return Some(found);
            }
        }
    }
    None
}

// Usage
let first_button = find_first::<gtk::Button>(&root).unwrap();
```

## Summary

- Template children are directly accessible as struct fields
- Use `set_name()` for widget identification
- GTK provides methods to find widgets by name or type
- For deep nesting, build helper functions to traverse the tree
- Connect signals directly to specific widgets for clean callbacks
