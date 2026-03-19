# Commands

Commands let you run async tasks in the background and receive results when they complete. They're perfect for I/O-bound operations that can run in parallel.

## When to Use Commands

- Network requests (HTTP, WebSocket)
- File I/O operations
- Database queries
- Any async operation that can run in parallel

## Basic Command Example

Here's a component that fetches data from a URL:

```rust
use gtk::prelude::*;
use relm4::{Component, ComponentParts, ComponentSender, RelmApp, SimpleComponent};
use std::time::Duration;

struct AppModel {
    data: Option<String>,
    is_loading: bool,
    error: Option<String>,
}

#[derive(Debug)]
enum AppMsg {
    FetchData,
    DataReceived(Result<String, String>),
}

#[derive(Debug)]
enum AppOutput {
    Status(String),
}

impl Component for AppModel {
    type Init = ();
    type Input = AppMsg;
    type Output = AppOutput;
    type CommandOutput = AppMsg;
    type Widgets = AppWidgets;
    type Root = gtk::Window;

    fn init_root() -> Self::Root {
        gtk::Window::builder()
            .title("Command Example")
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
            data: None,
            is_loading: false,
            error: None,
        };

        let vbox = gtk::Box::builder()
            .orientation(gtk::Orientation::Vertical)
            .spacing(10)
            .margin_all(20)
            .build();

        let status_label = gtk::Label::new(Some("Click button to fetch"));
        let data_label = gtk::Label::new(Some("No data"));
        data_label.set_selectable(true);
        let fetch_button = gtk::Button::with_label("Fetch Data");
        let error_label = gtk::Label::new(None);
        error_label.add_css_class("error");

        window.set_child(Some(&vbox));
        vbox.append(&status_label);
        vbox.append(&data_label);
        vbox.append(&error_label);
        vbox.append(&fetch_button);

        fetch_button.connect_clicked(move |_| {
            sender.input(AppMsg::FetchData);
        });

        let widgets = AppWidgets {
            status_label,
            data_label,
            error_label,
        };

        ComponentParts { model, widgets }
    }

    fn update(&mut self, msg: Self::Input, sender: ComponentSender<Self>, _root: &Self::Root) {
        match msg {
            AppMsg::FetchData => {
                self.is_loading = true;
                self.error = None;
                sender.output(AppOutput::Status("Fetching...".to_string())).unwrap();

                // Start async command
                let sender_clone = sender.clone();
                tokio::spawn(async move {
                    // Simulate network delay
                    tokio::time::sleep(Duration::from_secs(2)).await;
                    
                    // Simulate successful fetch
                    let data = "Hello from the internet!".to_string();
                    sender_clone.command_output(AppMsg::DataReceived(Ok(data)));
                });
            }
            AppMsg::DataReceived(result) => {
                self.is_loading = false;
                match result {
                    Ok(data) => {
                        self.data = Some(data.clone());
                        sender.output(AppOutput::Status("Fetch complete".to_string())).unwrap();
                    }
                    Err(e) => {
                        self.error = Some(e.clone());
                        sender.output(AppOutput::Status(format!("Error: {}", e))).unwrap();
                    }
                }
            }
        }
    }

    fn update_cmd(
        &mut self,
        msg: Self::CommandOutput,
        sender: ComponentSender<Self>,
        _root: &Self::Root,
    ) {
        // This is called when commands complete
        match msg {
            AppMsg::DataReceived(result) => {
                // Handle the result - update_view will be called after this
            }
            _ => {}
        }
    }

    fn update_view(&self, widgets: &mut Self::Widgets, _sender: ComponentSender<Self>) {
        widgets.status_label.set_label(if self.is_loading {
            "Loading..."
        } else if let Some(ref data) = self.data {
            &format!("Data: {}", data)
        } else {
            "Click button to fetch"
        });

        if let Some(ref error) = self.error {
            widgets.error_label.set_label(error);
            widgets.error_label.set_visible(true);
        } else {
            widgets.error_label.set_visible(false);
        }
    }
}

fn main() {
    let app = RelmApp::new("org.relm4.CommandExample");
    app.run::<AppModel>(());
}
```

## Component vs CommandOutput

When using commands, you need a `Component` (not `SimpleComponent`) and define `CommandOutput`:

```rust
impl Component for AppModel {
    type Init = ();
    type Input = AppMsg;           // Messages from UI
    type Output = AppOutput;       // Messages to parent
    type CommandOutput = AppMsg;   // Messages from commands
    
    // ...
    
    fn update_cmd(
        &mut self,
        output: Self::CommandOutput,
        sender: ComponentSender<Self>,
        root: &Self::Root,
    ) {
        // Handle command results
        match output {
            AppMsg::DataReceived(result) => {
                // Process result
            }
            _ => {}
        }
    }
}
```

## Sending Commands

### Using tokio::spawn

```rust
fn update(&mut self, msg: Self::Input, sender: ComponentSender<Self>, root: &Self::Root) {
    match msg {
        AppMsg::StartTask => {
            let sender = sender.clone();
            std::thread::spawn(move || {
                // Do work...
                let result = do_computation();
                // Send result back
                sender.command_output(AppMsg::TaskComplete(result));
            });
        }
    }
}
```

### Using tokio::spawn with async

```rust
fn update(&mut self, msg: Self::Input, sender: ComponentSender<Self>, root: &Self::Root) {
    match msg {
        AppMsg::StartAsyncTask => {
            let sender = sender.clone();
            tokio::spawn(async move {
                let result = fetch_data().await;
                sender.command_output(AppMsg::AsyncTaskComplete(result));
            });
        }
    }
}
```

## Command Patterns

### Parallel Commands

```rust
fn update(&mut self, msg: Self::Input, sender: ComponentSender<Self>, root: &Self::Root) {
    match msg {
        AppMsg::FetchAll => {
            let urls = vec!["https://api.example.com/1", "https://api.example.com/2"];
            
            for url in urls {
                let sender = sender.clone();
                let url = url.clone();
                tokio::spawn(async move {
                    let data = fetch(&url).await;
                    sender.command_output(AppMsg::DataFetched(url, data));
                });
            }
        }
    }
}

fn update_cmd(&mut self, msg: Self::CommandOutput, _: ComponentSender<Self>, _: &Self::Root) {
    match msg {
        AppMsg::DataFetched(url, data) => {
            self.results.insert(url, data);
        }
        _ => {}
    }
}
```

### Timeout Commands

```rust
use tokio::time::{timeout, Duration};

fn update(&mut self, msg: Self::Input, sender: ComponentSender<Self>, root: &Self::Root) {
    match msg {
        AppMsg::FetchWithTimeout => {
            let sender = sender.clone();
            tokio::spawn(async move {
                let result = timeout(
                    Duration::from_secs(5),
                    fetch_data()
                ).await;
                
                match result {
                    Ok(Ok(data)) => sender.command_output(AppMsg::Success(data)),
                    Ok(Err(e)) => sender.command_output(AppMsg::NetworkError(e.to_string())),
                    Err(_) => sender.command_output(AppMsg::Timeout),
                }
            });
        }
    }
}
```

### Cancelable Commands

```rust
use tokio::sync::oneshot;

struct AppModel {
    cancel_token: Option<oneshot::Sender<()>>,
}

fn update(&mut self, msg: Self::Input, sender: ComponentSender<Self>, root: &Self::Root) {
    match msg {
        AppMsg::StartLongTask => {
            // Cancel any existing task
            if let Some(cancel) = self.cancel_token.take() {
                let _ = cancel.send(());
            }
            
            let (cancel_tx, mut cancel_rx) = oneshot::channel();
            self.cancel_token = Some(cancel_tx);
            
            let sender = sender.clone();
            tokio::spawn(async move {
                tokio::select! {
                    _ = &mut cancel_rx => {
                        sender.command_output(AppMsg::Cancelled);
                    }
                    result = long_task() => {
                        sender.command_output(AppMsg::TaskResult(result));
                    }
                }
            });
        }
        AppMsg::Cancel => {
            if let Some(cancel) = self.cancel_token.take() {
                let _ = cancel.send(());
            }
        }
    }
}
```

## Summary

- Commands run async tasks on a thread pool
- Use `Component` trait with `CommandOutput` type
- Handle results in `update_cmd()` method
- Multiple commands can run in parallel
- Perfect for I/O-bound operations like network requests
