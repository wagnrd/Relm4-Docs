# Input Messages

Input messages are the primary way that external events reach your component. They are processed by the `update` function and can change your model's state.

## What are Input Messages?

Input messages are enum variants that represent events your component can handle:

```rust
#[derive(Debug)]
enum AppInput {
    Increment,
    Decrement,
    Reset,
}
```

## Processing Input Messages

The `update` function receives input messages and modifies the model accordingly:

```rust
impl SimpleComponent for AppModel {
    type Input = AppInput;
    
    fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
        match msg {
            AppInput::Increment => {
                self.counter = self.counter.wrapping_add(1);
            }
            AppInput::Decrement => {
                self.counter = self.counter.wrapping_sub(1);
            }
            AppInput::Reset => {
                self.counter = 0;
            }
        }
    }
}
```

## Sending Input Messages

### From Widget Signals

Connect GTK signals to send messages:

```rust
// Button click
let button = gtk::Button::with_label("Click me");
button.connect_clicked(move |_| {
    sender.input(AppInput::ButtonClicked);
});

// Entry text changed
let entry = gtk::Entry::new();
entry.connect_changed(move |entry| {
    let text = entry.text().to_string();
    sender.input(AppInput::TextChanged(text));
});

// Checkbox toggled
let check = gtk::CheckButton::with_label("Enable feature");
check.connect_toggled(move |btn| {
    let active = btn.is_active();
    sender.input(AppInput::FeatureToggled(active));
});
```

### From Keyboard Shortcuts

```rust
let window = gtk::Window::new();

window.connect_key_pressed(move |_, key, _, _| {
    use gtk::gdk::Key;
    match key {
        Key::q if true => sender.input(AppInput::Quit),  // Ctrl+Q
        Key::s if true => sender.input(AppInput::Save),  // Ctrl+S
        _ => gtk::Inhibit(false),
    }
});
```

### From Timers

```rust
use glib::source::timeout_add_seconds;

timeout_add_seconds(1, move || {
    sender.input(AppInput::TimerTick);
    glib::Continue(true)  // Continue the timer
});
```

## Message Handling Patterns

### Guard Conditions

Only process messages when certain conditions are met:

```rust
fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
    match msg {
        AppInput::SubmitForm => {
            if self.is_form_valid() {
                self.submit();
            }
        }
        AppInput::DeleteItem(index) => {
            if index < self.items.len() {
                self.items.remove(index);
            }
        }
    }
}
```

### Async Operations

For operations that take time, use the `update_cmd` function:

```rust
impl Component for AppModel {
    type CommandOutput = ();
    
    fn update(&mut self, msg: Self::Input, sender: ComponentSender<Self>, root: &Self::Root) {
        match msg {
            AppInput::StartLoading => {
                self.is_loading = true;
            }
            AppInput::LoadData => {
                // This is blocking - use commands instead for async
                self.load_from_disk();
            }
        }
    }
    
    fn update_cmd(&mut self, _: Self::CommandOutput, sender: ComponentSender<Self>, root: &Self::Root) {
        sender.input(AppInput::FinishLoading);
    }
}
```

## Summary

- Input messages are processed by the `update` function
- They typically originate from UI events (clicks, keypresses, etc.)
- Send messages using `sender.input(message)`
- Use pattern matching to handle different message types
