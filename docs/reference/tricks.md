# Tips and Tricks

A collection of helpful patterns and tricks for working with Relm4.

## UI Patterns

### Conditional Widget Visibility

```rust
struct AppModel {
    show_details: bool,
    details: Vec<String>,
}

fn update_view(&self, widgets: &mut AppWidgets, _sender: ComponentSender<Self>) {
    // Show/hide based on state
    widgets.details_container.set_visible(self.show_details);
    
    // Or use CSS classes
    if self.show_details {
        widgets.container.add_css_class("show-details");
    } else {
        widgets.container.remove_css_class("show-details");
    }
}
```

### Dynamic Layout

```rust
fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
    match msg {
        AppMsg::SetLayout(layout) => {
            self.current_layout = layout;
        }
    }
}

fn update_view(&self, widgets: &mut AppWidgets, _sender: ComponentSender<Self>) {
    // Remove all children
    widgets.container.unparent();
    
    // Add new layout based on state
    match self.current_layout {
        Layout::Grid => widgets.container.append(&self.grid),
        Layout::List => widgets.container.append(&self.list),
    }
}
```

### Loading States

```rust
struct AppModel {
    is_loading: bool,
    data: Option<Vec<String>>,
    error: Option<String>,
}

fn update_view(&self, widgets: &mut AppWidgets, _sender: ComponentSender<Self>) {
    widgets.spinner.set_visible(self.is_loading);
    widgets.content.set_visible(!self.is_loading && self.data.is_some());
    widgets.error.set_visible(self.error.is_some());
    
    if let Some(ref err) = self.error {
        widgets.error.set_label(err);
    }
    
    if let Some(ref data) = self.data {
        // Populate content
    }
}
```

## Async Patterns

### Debouncing

```rust
use std::time::Duration;

struct AppModel {
    search_text: String,
    debounce_timer: Option<glib::SourceId>,
}

fn update(&mut self, msg: Self::Input, sender: ComponentSender<Self>) {
    match msg {
        AppMsg::SearchTextChanged(text) => {
            // Cancel previous timer
            if let Some(timer) = self.debounce_timer.take() {
                glib::source::source_remove(timer);
            }
            
            self.search_text = text.clone();
            
            // Start new debounce timer
            let sender = sender.clone();
            self.debounce_timer = Some(
                glib::source::timeout_add_local(
                    Duration::from_millis(300),
                    move || {
                        sender.input(AppMsg::PerformSearch(text.clone()));
                        glib::Continue(false)
                    },
                )
            );
        }
    }
}
```

### Cancellation Token

```rust
use std::sync::atomic::{AtomicBool, Ordering};
use std::sync::Arc;

struct AppModel {
    cancel_flag: Arc<AtomicBool>,
}

fn update(&mut self, msg: Self::Input, sender: ComponentSender<Self>) {
    match msg {
        AppMsg::StartLongTask => {
            self.cancel_flag.store(false, Ordering::SeqCst);
            
            let cancel = self.cancel_flag.clone();
            let sender = sender.clone();
            
            std::thread::spawn(move || {
                for i in 0..100 {
                    if cancel.load(Ordering::SeqCst) {
                        sender.input(AppMsg::TaskCancelled).unwrap();
                        return;
                    }
                    std::thread::sleep(Duration::from_millis(100));
                }
                sender.input(AppMsg::TaskComplete).unwrap();
            });
        }
        AppMsg::CancelTask => {
            self.cancel_flag.store(true, Ordering::SeqCst);
        }
    }
}
```

## State Management

### Immutable Updates

```rust
#[derive(Clone)]
struct Document {
    pages: Vec<Page>,
    cursor: Position,
}

impl Document {
    fn with_page(mut self, page: Page) -> Self {
        self.pages.push(page);
        self
    }
    
    fn update_page(mut self, index: usize, page: Page) -> Self {
        self.pages[index] = page;
        self
    }
    
    fn remove_page(mut self, index: usize) -> Self {
        self.pages.remove(index);
        self
    }
}

// Usage
self.document = self.document.with_page(new_page);
```

### Undo/Redo

```rust
use std::collections::VecDeque;

struct AppModel {
    state_history: VecDeque<AppState>,
    history_index: usize,
    max_history: usize,
}

impl AppModel {
    fn push_state(&mut self, state: AppState) {
        // Remove any redo states
        while self.history_index > 0 {
            self.state_history.pop_front();
            self.history_index -= 1;
        }
        
        // Add new state
        self.state_history.push_back(state);
        
        // Limit history size
        while self.state_history.len() > self.max_history {
            self.state_history.pop_front();
        }
    }
    
    fn undo(&mut self) -> Option<&AppState> {
        if self.history_index > 0 {
            self.history_index -= 1;
            self.state_history.get(self.history_index)
        } else {
            None
        }
    }
    
    fn redo(&mut self) -> Option<&AppState> {
        if self.history_index < self.state_history.len() - 1 {
            self.history_index += 1;
            self.state_history.get(self.history_index)
        } else {
            None
        }
    }
}
```

## Performance Tips

### Reduce Allocations

```rust
// Bad: Creates new String on every update
fn update_view(&self, widgets: &mut AppWidgets, _sender: ComponentSender<Self>) {
    widgets.label.set_label(&format!("Count: {}", self.count));
}

// Good: Reuse buffer
struct AppWidgets {
    label: gtk::Label,
    buffer: String,
}

fn update_view(&self, widgets: &mut AppWidgets, _sender: ComponentSender<Self>) {
    widgets.buffer.clear();
    widgets.buffer.push_str("Count: ");
    widgets.buffer.push_str(&self.count.to_string());
    widgets.label.set_label(&widgets.buffer);
}
```

### Lazy Widget Creation

```rust
struct AppModel {
    show_details: bool,
    details_widget: Option<gtk::Expander>,
}

fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
    match msg {
        AppMsg::ToggleDetails => {
            self.show_details = !self.show_details;
            
            // Create widget only when needed
            if self.show_details && self.details_widget.is_none() {
                self.details_widget = Some(self.create_details_widget());
            }
        }
    }
}

fn update_view(&self, widgets: &mut AppWidgets, _sender: ComponentSender<Self>) {
    widgets.details_container.set_visible(self.show_details);
    
    if self.show_details {
        if let Some(ref details) = self.details_widget {
            widgets.details_container.append(details);
        }
    } else {
        widgets.details_container.unparent();
    }
}
```

## Debugging

### Logging

```rust
use tracing::{info, warn, error};

fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
    info!("Processing message: {:?}", msg);
    
    match msg {
        AppMsg::ImportantAction => {
            info!("Starting important action");
            // ...
        }
        AppMsg::RiskyOperation => {
            warn!("Performing risky operation");
            // ...
        }
    }
}
```

### Debug Output

```rust
fn update_view(&self, widgets: &mut AppWidgets, _sender: ComponentSender<Self>) {
    #[cfg(debug_assertions)]
    {
        widgets.debug_label.set_label(&format!(
            "State: {:?}\nModel: {:?}",
            self.state, self
        ));
        widgets.debug_label.set_visible(true);
    }
    
    // Normal updates...
}
```

## GTK Tips

### Focus Management

```rust
fn focus_next(widget: &gtk::Widget) {
    widget.child_focus(gtk::DirectionType::TabForward);
}

fn focus_previous(widget: &gtk::Widget) {
    widget.child_focus(gtk::DirectionType::TabBackward);
}
```

### Scroll to Bottom

```rust
use gtk::prelude::ScrolledWindowExt;

scrolled.set_vadjustment(
    &scrolled.vadjustment().unwrap()
);

if let Some(adj) = scrolled.vadjustment() {
    adj.set_value(adj.upper() - adj.page_size());
}
```

### Tooltips

```rust
widget.set_tooltip_text(Some("Help text here"));

// Or with markup
widget.set_tooltip_markup(Some("<b>Bold</b> tooltip"));
```

## Summary

- Use conditional visibility for dynamic layouts
- Implement debouncing for text input
- Consider immutable state patterns for complex data
- Profile and optimize hot paths
- Use debug features in development
