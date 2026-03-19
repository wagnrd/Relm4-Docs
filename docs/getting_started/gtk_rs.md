# GTK-rs Overview

GTK-rs provides Rust bindings for GTK4. Understanding GTK-rs basics helps you work more effectively with Relm4.

## GTK4 Widget Hierarchy

GTK4 organizes widgets in a hierarchy:

```
GtkWidget (base)
├── GtkWindow
│   ├── GtkApplicationWindow
│   └── GtkDialog
│       ├── GtkMessageDialog
│       ├── GtkFileChooserNative
│       └── GtkPopover
├── GtkContainer
│   ├── GtkBox
│   ├── GtkGrid
│   ├── GtkStack
│   └── GtkScrolledWindow
├── GtkButton
│   ├── GtkToggleButton
│   ├── GtkCheckButton
│   └── GtkMenuButton
├── GtkEntry
│   └── GtkTextView
└── GtkDisplay
    └── GtkLabel
```

## Common Widgets

### Windows

```rust
use gtk::prelude::*;

// Create a window
let window = gtk::Window::builder()
    .title("My App")
    .default_width(400)
    .default_height(300)
    .build();

// Set child
window.set_child(Some(&box_widget));

// Connect close handler
window.connect_close_request(|_| {
    gtk::Inhibit(false)
});
```

### Boxes

```rust
use gtk::Orientation;

// Vertical box
let vbox = gtk::Box::builder()
    .orientation(Orientation::Vertical)
    .spacing(10)
    .margin_all(10)
    .build();

// Horizontal box
let hbox = gtk::Box::builder()
    .orientation(Orientation::Horizontal)
    .spacing(5)
    .build();

// Add children
vbox.append(&button);
vbox.prepend(&label);
hbox.append(&entry);
```

### Buttons

```rust
// Simple button
let button = gtk::Button::with_label("Click me");

// Button with callback
button.connect_clicked(|_| {
    println!("Clicked!");
});

// Toggle button
let toggle = gtk::ToggleButton::new();
toggle.connect_toggled(|btn| {
    println!("Active: {}", btn.is_active());
});

// Check button
let check = gtk::CheckButton::with_label("Accept terms");
check.connect_toggled(|btn| {
    if btn.is_active() {
        println!("Terms accepted!");
    }
});

// Menu button
let menu_btn = gtk::MenuButton::new();
menu_btn.set_menu_model(Some(&menu));
```

### Text Entry

```rust
let entry = gtk::Entry::builder()
    .placeholder_text("Enter text...")
    .max_length(50)
    .build();

// Connect signals
entry.connect_changed(|entry| {
    println!("Text: {}", entry.text());
});

entry.connect_activate(|entry| {
    println!("Submitted: {}", entry.text());
});

// Get/set text
entry.set_text("Hello");
let text = entry.text();
```

### Labels

```rust
let label = gtk::Label::new(Some("Hello!"));
label.set_label("New text");
label.set_selectable(true);
label.set_wrap(true);
label.set_wrap_mode(gtk::WrapMode::Word);
```

### Grids

```rust
let grid = gtk::Grid::builder()
    .row_spacing(5)
    .column_spacing(5)
    .margin_start(10)
    .margin_end(10)
    .build();

// Attach widgets
grid.attach(&label1, 0, 0, 1, 1);  // col, row, width, height
grid.attach(&label2, 1, 0, 1, 1);
grid.attach(&button, 0, 1, 2, 1);    // spans two columns
```

### Scrollable Windows

```rust
let scrolled = gtk::ScrolledWindow::builder()
    .hscrollbar_policy(gtk::PolicyType::Automatic)
    .vscrollbar_policy(gtk::PolicyType::Automatic)
    .min_content_width(300)
    .min_content_height(400)
    .build();

scrolled.set_child(Some(&text_view));
```

## Layout Containers

### Box

```rust
let box = gtk::Box::new(gtk::Orientation::Vertical, 10);
// or
let box = gtk::Box::builder()
    .orientation(gtk::Orientation::Vertical)
    .spacing(10)
    .build();
```

### Grid

```rust
let grid = gtk::Grid::new();
grid.attach(child, column, row, width, height);
```

### Stack

```rust
let stack = gtk::Stack::new();
stack.add_titled(&child1, "page1", "Page 1");
stack.add_titled(&child2, "page2", "Page 2");

let switcher = gtk::StackSwitcher::new();
switcher.set_stack(Some(&stack));
```

### Paned

```rust
let paned = gtk::Paned::new(gtk::Orientation::Horizontal);
paned.set_start_child(&sidebar);
paned.set_end_child(&content);
paned.set_position(200);  // Initial divider position
```

## Signal Handling

### Common Signals

```rust
// Button
button.connect_clicked(|_| { });

// Entry
entry.connect_changed(|_| { });
entry.connect_activate(|_| { });

// Toggle
toggle.connect_toggled(|_| { });

// Window
window.connect_close_request(|_| { });
window.connect_key_pressed(|_, key, _, _| { gtk::Inhibit(false) });

// ListBox
list_box.connect_row_activated(|_, row| { });
```

### Signal Callback Parameters

```rust
button.connect_clicked(|button| {
    // button is &gtk::Button
});

entry.connect_changed(|entry| {
    let text = entry.text();
});

window.connect_key_pressed(|window, key, modifier, _| {
    // window: &gtk::Window
    // key: gdk::Key
    // modifier: gdk::ModifierType
});
```

## Builder Pattern

GTK uses a fluent builder pattern:

```rust
let button = gtk::Button::builder()
    .label("Click me")           // Set property
    .halign(gtk::Align::Center)  // Set alignment
    .margin_top(10)               // Set margin
    .margin_bottom(10)
    .css_classes(&["suggested-action"])  // Add CSS class
    .build();                     // Create the widget
```

## CSS Styling

### Adding CSS Classes

```rust
widget.add_css_class("card");
widget.add_css_class("highlighted");

// Remove class
widget.remove_css_class("highlighted");

// Check class
if widget.has_css_class("card") { }
```

### Built-in Classes

- `.suggested-action` - Primary action buttons (blue)
- `.destructive-action` - Delete/danger buttons (red)
- `.flat` - Flat appearance
- `.linked` - Grouped buttons
- `.circular` - Circular shape

## Resources

- [GTK4 Documentation](https://docs.gtk.org/gtk4/)
- [gtk4-rs Book](https://gtk-rs.org/gtk4-rs/git/book/)
- [gtk4-rs Docs](https://gtk-rs.org/gtk4-rs/git/docs/gtk4/)
