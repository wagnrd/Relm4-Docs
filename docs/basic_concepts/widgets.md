# Widgets

GTK4 provides widgets as the building blocks for your user interface. Widgets can display information and respond to user interactions.

## GTK Widget Basics

Widgets in GTK4 are organized in a parent-child hierarchy. Common patterns include:

- **Containers**: Widgets that can hold other widgets (Box, Grid, etc.)
- **Controls**: Interactive widgets (Button, Entry, etc.)
- **Display**: Widgets that show information (Label, Image, etc.)

## Widget Behavior

GTK widgets behave similarly to `Rc<T>` in Rust:

1. **Cloning doesn't create new instances** - It just increases the reference count
2. **Widgets are kept alive automatically** - GTK manages their lifetime
3. **Widgets are not thread-safe** - They can only be used on the main thread

## Common GTK Widgets

### Labels

Display text that users cannot edit:

```rust
let label = gtk::Label::new(Some("Hello, World!"));
label.set_label("New text");
label.set_selectable(true);
```

### Buttons

```rust
// Simple button with label
let button = gtk::Button::with_label("Click me");

// Button with callback
button.connect_clicked(|_| {
    println!("Button was clicked!");
});
```

### Text Entry

```rust
let entry = gtk::Entry::new();
entry.set_placeholder_text(Some("Enter text..."));
entry.set_text("Initial text");

// Connect to text changes
entry.connect_changed(|entry| {
    println!("Text changed: {}", entry.text());
});

// Get text on activation (Enter key)
entry.connect_activate(|entry| {
    println!("Submitted: {}", entry.text());
});
```

### Box (Container)

A box arranges widgets either horizontally or vertically:

```rust
// Vertical box
let vbox = gtk::Box::builder()
    .orientation(gtk::Orientation::Vertical)
    .spacing(10)
    .margin_all(10)
    .build();

// Horizontal box
let hbox = gtk::Box::builder()
    .orientation(gtk::Orientation::Horizontal)
    .spacing(5)
    .build();

// Add widgets to box
vbox.append(&button);
vbox.append(&label);
vbox.prepend(&another_widget);  // Add at beginning
```

### Grid

A grid arranges widgets in rows and columns:

```rust
let grid = gtk::Grid::builder()
    .row_spacing(5)
    .column_spacing(5)
    .margin_start(10)
    .margin_end(10)
    .build();

// Attach widgets to grid
grid.attach(&button1, 0, 0, 1, 1);  // column, row, width, height
grid.attach(&button2, 1, 0, 1, 1);
grid.attach(&label, 0, 1, 2, 1);     // spans two columns
```

## Widget Properties

GTK widgets use a builder pattern for setting properties:

```rust
let button = gtk::Button::builder()
    .label("Click me")
    .halign(gtk::Align::Center)
    .valign(gtk::Align::Center)
    .margin_top(10)
    .margin_bottom(10)
    .margin_start(10)
    .margin_end(10)
    .css_classes(&["suggested-action"])
    .build();
```

## Common Widget Signals

### Button Signals

```rust
button.connect_clicked(|_| { /* handle click */ });
button.connect_toggled(|btn| { /* handle toggle */ });
```

### Entry Signals

```rust
entry.connect_changed(|_| { /* text changed */ });
entry.connect_activate(|_| { /* enter pressed */ });
entry.connect_focus_in_event(|_, _| { /* gained focus */ });
entry.connect_focus_out_event(|_, _| { /* lost focus */ });
```

### Window Signals

```rust
use gtk::gdk::Event;

window.connect_close_request(move |_| {
    // Window is about to close
    // Return gtk::Propagation::Stop to prevent closing
    gtk::Propagation::Proceed
});

window.connect_key_pressed(|_, key, _, _| {
    use gtk::gdk::Key;
    match key {
        Key::Escape => { /* handle escape */ }
        _ => gtk::Inhibit(false),
    }
});
```

## Widget Hierarchy

Every widget has exactly one parent (except the root window):

```rust
let window = gtk::Window::new();

// Set the window's child
window.set_child(Some(&vbox));

// Or attach to another container
vbox.append(&button);
another_box.prepend(&label);
grid.attach(&widget, col, row, 1, 1);
```

## Widget Visibility

Widgets can be shown or hidden:

```rust
widget.set_visible(true);
widget.set_visible(false);

// Or use show/hide methods
widget.show();
widget.hide();
```

## Widget Styling

### CSS Classes

```rust
widget.add_css_class("suggested-action");  // Blue button
widget.add_css_class("destructive-action"); // Red button
widget.remove_css_class("my-class");
widget.set_css_classes(&["class1", "class2"]);
```

### Built-in Classes

GTK provides several CSS classes:
- `.suggested-action` - Primary action buttons
- `.destructive-action` - Delete/dangerous actions
- `.flat` - Flat appearance (no border)
- `.linked` - Buttons grouped together
- `.circular` - Circular shape

## Widget References in Relm4

In Relm4, widgets are stored in a separate struct for manual implementation:

```rust
struct AppWidgets {
    label: gtk::Label,
    button: gtk::Button,
    entry: gtk::Entry,
}

struct AppModel {
    counter: u8,
}

impl SimpleComponent for AppModel {
    type Widgets = AppWidgets;
    
    fn init(...) -> ComponentParts<Self> {
        // Create widgets
        let label = gtk::Label::new(Some("0"));
        let button = gtk::Button::with_label("+");
        
        // Connect signals
        button.connect_clicked(move |_| {
            sender.input(AppMsg::Increment);
        });
        
        // Store widgets
        let widgets = AppWidgets { label, button };
        
        ComponentParts { model, widgets }
    }
    
    fn update_view(&self, widgets: &mut Self::Widgets, _sender: ComponentSender<Self>) {
        widgets.label.set_label(&self.counter.to_string());
    }
}
```

## Summary

- GTK widgets are the visual building blocks of your UI
- Widgets behave like `Rc<T>` - cloning increases reference count
- Use the builder pattern to configure widget properties
- Connect signals to handle user interactions
- Store widgets in a separate struct in Relm4
- Update widgets in `update_view()` to reflect model changes
