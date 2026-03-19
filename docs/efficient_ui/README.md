# Efficient UI Updates

Relm4 follows the Elm programming model where data and widgets are strictly separated. This raises an important question: when the model changes, how do we know which parts of the UI need updating?

## The Problem

Consider an app with 1000 counters. If you increment just the first counter, a naive approach would rebuild all 1000 UI elements. This is wasteful and slow.

## Relm4's Solutions

Relm4 provides two powerful mechanisms for efficient updates:

1. **Trackers** - Detect which specific fields changed
2. **Factories** - Efficiently manage collections of similar widgets

## Trackers

Trackers let you know exactly which fields changed, so you can update only the affected widgets:

```rust
use tracker::track;

#[tracker::track]
struct AppModel {
    counter: u8,
    name: String,
    is_active: bool,
}

impl SimpleComponent for AppModel {
    fn update(&mut self, msg: Self::Input, sender: ComponentSender<Self>) {
        self.reset();  // Clear previous changes
        
        match msg {
            AppInput::Increment => {
                self.set_counter(self.counter + 1);
            }
            AppInput::SetName(name) => {
                self.set_name(name);
            }
        }
    }
    
    fn update_view(&self, widgets: &mut Self::Widgets, sender: ComponentSender<Self>) {
        // Only update if counter changed
        if self.changed(AppModel::counter()) {
            widgets.counter_label.set_label(&self.counter.to_string());
        }
        
        // Only update if name changed
        if self.changed(AppModel::name()) {
            widgets.name_label.set_label(&self.name);
        }
    }
}
```

See the [Tracker](tracker.md) chapter for detailed information.

## Factories

Factories manage collections of widgets efficiently:

```rust
use relm4::factory::FactoryVecDeque;

struct AppModel {
    counters: FactoryVecDeque<CounterItem>,
}

#[relm4::factory]
impl FactoryComponent for CounterItem {
    type Init = u8;
    type Input = CounterMsg;
    type Output = AppMsg;
    type ParentWidget = gtk::Box;
    
    // ... factory implementation
}

impl SimpleComponent for AppModel {
    fn update(&mut self, msg: Self::Input, sender: ComponentSender<Self>) {
        let mut guard = self.counters.guard();
        match msg {
            AppMsg::AddCounter => {
                guard.push_back(self.next_value);
                self.next_value += 1;
            }
            AppMsg::RemoveCounter => {
                guard.pop_back();
            }
        }
    }
}
```

See the [Factories](factory.md) chapter for detailed information.

## Comparison

| Feature | Trackers | Factories |
|---------|----------|-----------|
| **Use case** | Track struct field changes | Manage widget collections |
| **Efficiency** | Update only changed fields | Minimal widget recreation |
| **Complexity** | Simple to use | More setup required |
| **Best for** | Single-instance widgets | Lists, grids, dynamic UI |

## When to Use Each

### Use Trackers When:

- You have multiple widgets that display different model fields
- You want to minimize unnecessary widget updates
- Your model has many optional or conditional fields

### Use Factories When:

- You have a dynamic list/grid of similar widgets
- Users can add/remove items
- You need efficient reordering or filtering

### Use Both When:

- You have a factory with items that have multiple tracked fields
- Complex UIs with both collections and individual elements

## Performance Tips

### Minimize Widget Updates

```rust
// Bad: Updates all widgets every time
fn update_view(&self, widgets: &mut Self::Widgets, _sender: ComponentSender<Self>) {
    widgets.header.set_label(&self.title);
    widgets.content.set_label(&self.body);
    widgets.footer.set_label(&self.footer);
}

// Good: Update only what changed (use trackers)
fn update_view(&self, widgets: &mut Self::Widgets, _sender: ComponentSender<Self>) {
    if self.changed(AppModel::title()) {
        widgets.header.set_label(&self.title);
    }
    if self.changed(AppModel::body()) {
        widgets.content.set_label(&self.body);
    }
    if self.changed(AppModel::footer()) {
        widgets.footer.set_label(&self.footer);
    }
}
```

### Batch Factory Operations

```rust
// Bad: Creates/destroys widgets for each operation
fn update(&mut self, msg: Self::Input, _: ComponentSender<Self>) {
    match msg {
        AppMsg::Add5 => {
            for _ in 0..5 {
                self.counters.guard().push_back(value);
            }
        }
    }
}

// Good: Use a single guard for multiple operations
fn update(&mut self, msg: Self::Input, _: ComponentSender<Self>) {
    match msg {
        AppMsg::Add5 => {
            let mut guard = self.counters.guard();
            for _ in 0..5 {
                guard.push_back(value);
            }
        }
    }
}
```

## Summary

- Relm4 separates data and UI for predictable updates
- Trackers detect which fields changed for efficient single-widget updates
- Factories efficiently manage collections of similar widgets
- Choose the right tool based on your UI structure
- Both mechanisms can be combined in complex applications
