# Async Components

Async components handle async/await natively, making it easy to write asynchronous UI logic without mixing threading concepts.

## When to Use Async Components

- Complex async workflows
- Multiple concurrent async operations
- Async streams
- When you're already using async/await extensively

## Basic Async Component

```rust
use gtk::prelude::*;
use relm4::{
    AsyncComponent, AsyncComponentParts, AsyncComponentSender, ComponentParts, 
    ComponentSender, RelmApp, SimpleComponent,
};
use std::time::Duration;

struct AppModel {
    data: String,
    is_loading: bool,
}

#[derive(Debug)]
enum AppMsg {
    FetchData,
    DataReceived(String),
}

#[derive(Debug)]
enum AppOutput {
    Status(String),
}

struct AppWidgets {
    label: gtk::Label,
    button: gtk::Button,
}

#[relm4::component]
impl AsyncComponent for AppModel {
    type Init = ();
    type Input = AppMsg;
    type Output = AppOutput;
    type Widgets = AppWidgets;
    type Root = gtk::Window;

    fn init_root() -> Self::Root {
        gtk::Window::builder()
            .title("Async Component Example")
            .default_width(300)
            .default_height(100)
            .build()
    }

    async fn init(
        _: Self::Init,
        window: Self::Root,
        sender: AsyncComponentSender<Self>,
    ) -> AsyncComponentParts<Self> {
        let model = AppModel {
            data: String::new(),
            is_loading: false,
        };

        let vbox = gtk::Box::builder()
            .orientation(gtk::Orientation::Vertical)
            .spacing(10)
            .margin_all(20)
            .build();

        let label = gtk::Label::new(Some("Ready"));
        let button = gtk::Button::with_label("Fetch");

        window.set_child(Some(&vbox));
        vbox.append(&label);
        vbox.append(&button);

        let widgets = AppWidgets { label, button };

        AsyncComponentParts { model, widgets }
    }

    async fn update(&mut self, msg: Self::Input, sender: AsyncComponentSender<Self>) {
        match msg {
            AppMsg::FetchData => {
                self.is_loading = true;
                sender.output(AppOutput::Status("Loading...".to_string())).unwrap();

                // Async work directly in update
                tokio::time::sleep(Duration::from_secs(2)).await;
                
                self.data = "Data loaded!".to_string();
                self.is_loading = false;
                sender.output(AppOutput::Status("Done".to_string())).unwrap();
            }
            AppMsg::DataReceived(data) => {
                self.data = data;
            }
        }
    }

    fn update_view(&self, widgets: &mut Self::Widgets, _sender: AsyncComponentSender<Self>) {
        widgets.label.set_label(&if self.is_loading {
            "Loading...".to_string()
        } else {
            self.data.clone()
        });
        widgets.button.set_sensitive(!self.is_loading);
    }
}

fn main() {
    let app = RelmApp::new("org.relm4.AsyncExample");
    app.run::<AppModel>(());
}
```

## Async vs Sync Components

| Feature | Sync | Async |
|---------|------|-------|
| `update()` | Synchronous | `async fn` |
| `sender` | `ComponentSender` | `AsyncComponentSender` |
| `spawn` | Manual | Built-in |
| Best for | Simple cases | Complex async |

## AsyncComponentSender

The async sender provides:

```rust
impl AsyncComponentSender<MyComponent> {
    // Send async command
    async fn spawn<F, Fut>(&self, future: F)
    where
        F: Future<Output = ()> + Send + 'static;
    
    // Send message immediately
    fn input(&self, msg: Self::Input);
    
    // Send output
    fn output(&self, msg: Self::Output) -> Result<(), ...>;
}
```

## Async Spawn

Run async tasks that can send messages when complete:

```rust
async fn update(&mut self, msg: Self::Input, sender: AsyncComponentSender<Self>) {
    match msg {
        AppMsg::StartTask => {
            let sender = sender.clone();
            
            // Spawn async task
            self.spawn(async move {
                let result = some_async_operation().await;
                sender.input(AppMsg::TaskComplete(result));
            });
        }
    }
}
```

## Complete Async Example

```rust
use gtk::prelude::*;
use relm4::{
    AsyncComponent, AsyncComponentParts, AsyncComponentSender, RelmApp,
};
use std::time::Duration;

struct AppModel {
    progress: u8,
    status: String,
}

#[derive(Debug)]
enum AppMsg {
    StartDownload,
    Progress(u8),
    Complete,
    Error(String),
}

struct AppWidgets {
    progress_bar: gtk::ProgressBar,
    status_label: gtk::Label,
    button: gtk::Button,
}

#[relm4::component]
impl AsyncComponent for AppModel {
    type Init = ();
    type Input = AppMsg;
    type Output = ();
    type Widgets = AppWidgets;
    type Root = gtk::Window;

    fn init_root() -> Self::Root {
        gtk::Window::builder()
            .title("Async Download")
            .default_width(300)
            .default_height(150)
            .build()
    }

    async fn init(
        _: Self::Init,
        window: Self::Root,
        sender: AsyncComponentSender<Self>,
    ) -> AsyncComponentParts<Self> {
        let model = AppModel {
            progress: 0,
            status: "Ready".to_string(),
        };

        let vbox = gtk::Box::builder()
            .orientation(gtk::Orientation::Vertical)
            .spacing(10)
            .margin_all(20)
            .build();

        let progress_bar = gtk::ProgressBar::new();
        let status_label = gtk::Label::new(Some("Ready"));
        let button = gtk::Button::with_label("Start Download");

        window.set_child(Some(&vbox));
        vbox.append(&progress_bar);
        vbox.append(&status_label);
        vbox.append(&button);

        let widgets = AppWidgets {
            progress_bar,
            status_label,
            button,
        };

        AsyncComponentParts { model, widgets }
    }

    async fn update(&mut self, msg: Self::Input, sender: AsyncComponentSender<Self>) {
        match msg {
            AppMsg::StartDownload => {
                self.button.set_sensitive(false);
                
                // Simulate download with progress updates
                for i in 0..=100 {
                    tokio::time::sleep(Duration::from_millis(50)).await;
                    self.progress = i;
                    self.status = format!("Downloading... {}%", i);
                }
                
                self.status = "Download complete!".to_string();
                self.button.set_sensitive(true);
            }
            AppMsg::Progress(p) => {
                self.progress = p;
            }
            AppMsg::Complete => {
                self.progress = 100;
                self.status = "Complete!".to_string();
                self.button.set_sensitive(true);
            }
            AppMsg::Error(e) => {
                self.status = format!("Error: {}", e);
                self.button.set_sensitive(true);
            }
        }
    }

    fn update_view(&self, widgets: &mut Self::Widgets, _sender: AsyncComponentSender<Self>) {
        widgets.progress_bar.set_fraction(self.progress as f64 / 100.0);
        widgets.status_label.set_label(&self.status);
    }
}

fn main() {
    let app = RelmApp::new("org.relm4.AsyncDownload");
    app.run::<AppModel>(());
}
```

## Async Factories

Async factories work similarly for collections:

```rust
use relm4::{
    factory::{AsyncFactoryComponent, AsyncFactorySender, AsyncFactoryVecDeque, DynamicIndex},
    AsyncComponent, AsyncComponentParts, AsyncComponentSender,
};

struct AsyncListItem {
    data: String,
}

#[derive(Debug)]
enum AsyncItemInput {
    LoadData,
    DataLoaded(String),
}

#[relm4::factory]
impl AsyncFactoryComponent for AsyncListItem {
    type Init = ();
    type Input = AsyncItemInput;
    type Output = ();
    type CommandOutput = ();
    type ParentWidget = gtk::Box;

    async fn init_model(
        &mut self,
        _: Self::Init,
        _index: &DynamicIndex,
        _sender: AsyncFactorySender<Self>,
    ) {
        self.data = "Loading...".to_string();
    }

    async fn update(
        &mut self,
        msg: Self::Input,
        _sender: AsyncFactorySender<Self>,
    ) {
        match msg {
            AsyncItemInput::LoadData => {
                // Start async load
            }
            AsyncItemInput::DataLoaded(data) => {
                self.data = data;
            }
        }
    }
}
```

## Summary

- Async components use `async fn` for their update method
- Use `AsyncComponentSender` for sending messages
- Built-in `spawn` for running async tasks
- Handle async streams with async factories
- Great for complex async workflows
