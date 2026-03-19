# Workers

Workers are components without widgets that run on a separate thread. They're perfect for CPU-bound tasks or operations that must execute sequentially.

## When to Use Workers

- Long-running computations
- Blocking I/O operations
- Tasks that must not block the UI
- Sequential processing requirements

## Basic Worker Example

Here's a worker that performs slow calculations:

```rust
use gtk::prelude::*;
use relm4::{
    Component, ComponentParts, ComponentSender, RelmApp, RelmWidgetExt, SimpleComponent, Worker,
    WorkerController,
};
use std::time::Duration;

struct Calculator;

#[derive(Debug)]
enum CalculatorMsg {
    Add(u32, u32),
    Subtract(u32, u32),
    SlowTask(u32),
}

#[derive(Debug)]
enum CalculatorOutput {
    Result(i64),
    TaskComplete,
}

impl Worker for Calculator {
    type Init = ();
    type Input = CalculatorMsg;
    type Output = CalculatorOutput;

    fn init(_init: Self::Init, _sender: ComponentSender<Self>) -> Self {
        println!("Worker initialized");
        Calculator
    }

    fn update(&mut self, msg: CalculatorMsg, sender: ComponentSender<Self>) {
        match msg {
            CalculatorMsg::Add(a, b) => {
                let result = (a as i64) + (b as i64);
                sender.output(CalculatorOutput::Result(result)).unwrap();
            }
            CalculatorMsg::Subtract(a, b) => {
                let result = (a as i64) - (b as i64);
                sender.output(CalculatorOutput::Result(result)).unwrap();
            }
            CalculatorMsg::SlowTask(n) => {
                // Simulate slow work
                std::thread::sleep(Duration::from_secs(2));
                let result = (n as i64) * 2;
                sender.output(CalculatorOutput::Result(result)).unwrap();
                sender.output(CalculatorOutput::TaskComplete).unwrap();
            }
        }
    }
}

struct AppModel {
    result: String,
    is_working: bool,
    calculator: WorkerController<Calculator>,
}

#[derive(Debug)]
enum AppMsg {
    Calculate(u32, u32),
    SlowCalculate(u32),
    ReceiveResult(i64),
    TaskDone,
}

impl SimpleComponent for AppModel {
    type Init = ();
    type Input = AppMsg;
    type Output = ();
    type Widgets = AppWidgets;
    type Root = gtk::Window;

    fn init_root() -> Self::Root {
        gtk::Window::builder()
            .title("Worker Example")
            .default_width(300)
            .default_height(200)
            .build()
    }

    fn init(
        _: Self::Init,
        window: Self::Root,
        sender: ComponentSender<Self>,
    ) -> ComponentParts<Self> {
        let calculator = Calculator::builder()
            .detach_worker(())
            .forward(sender.input_sender(), |output| match output {
                CalculatorOutput::Result(r) => AppMsg::ReceiveResult(r),
                CalculatorOutput::TaskComplete => AppMsg::TaskDone,
            });

        let model = AppModel {
            result: String::from("Ready"),
            is_working: false,
            calculator,
        };

        let vbox = gtk::Box::builder()
            .orientation(gtk::Orientation::Vertical)
            .spacing(10)
            .margin_all(20)
            .build();

        let result_label = gtk::Label::new(Some("Ready"));
        let add_button = gtk::Button::with_label("Add 5 + 3");
        let slow_button = gtk::Button::with_label("Slow Task");
        let status_label = gtk::Label::new(Some("Idle"));

        window.set_child(Some(&vbox));
        vbox.append(&result_label);
        vbox.append(&add_button);
        vbox.append(&slow_button);
        vbox.append(&status_label);

        add_button.connect_clicked(move |_| {
            sender.input(AppMsg::Calculate(5, 3));
        });

        slow_button.connect_clicked(move |_| {
            sender.input(AppMsg::SlowCalculate(10));
        });

        let widgets = AppWidgets {
            result_label,
            status_label,
        };

        ComponentParts { model, widgets }
    }

    fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
        match msg {
            AppMsg::Calculate(a, b) => {
                self.is_working = true;
                self.calculator.sender().send(CalculatorMsg::Add(a, b)).unwrap();
            }
            AppMsg::SlowCalculate(n) => {
                self.is_working = true;
                self.calculator.sender().send(CalculatorMsg::SlowTask(n)).unwrap();
            }
            AppMsg::ReceiveResult(r) => {
                self.result = format!("Result: {}", r);
                self.is_working = false;
            }
            AppMsg::TaskDone => {
                self.result = format!("{} - Task complete!", self.result);
                self.is_working = false;
            }
        }
    }

    fn update_view(&self, widgets: &mut Self::Widgets, _sender: ComponentSender<Self>) {
        widgets.result_label.set_label(&self.result);
        widgets.status_label.set_label(if self.is_working { "Working..." } else { "Idle" });
    }
}

fn main() {
    let app = RelmApp::new("org.relm4.WorkerExample");
    app.run::<AppModel>(());
}
```

## Worker Trait

The `Worker` trait is simpler than `Component`:

```rust
pub trait Worker: Sized + 'static {
    type Init;
    type Input: Debug + 'static;
    type Output: Debug + 'static;

    fn init(init: Self::Init, sender: ComponentSender<Self>) -> Self;
    fn update(&mut self, msg: Self::Input, sender: ComponentSender<Self>);
}
```

### No Widgets or Root

Workers don't need:
- `Root` type
- `Widgets` type
- `init_root()` method
- `init()` method
- `update_view()` method

### init vs update

```rust
impl Worker for MyWorker {
    type Init = Config;
    type Input = WorkerMsg;
    type Output = WorkerOutput;

    fn init(config: Self::Init, _sender: ComponentSender<Self>) -> Self {
        // One-time setup (database connection, file handles, etc.)
        MyWorker { config }
    }

    fn update(&mut self, msg: Self::Input, sender: ComponentSender<Self>) {
        // Process messages
    }
}
```

## Launching Workers

### Detached Worker

```rust
let worker = MyWorker::builder()
    .detach_worker(())
    .forward(sender.input_sender(), transform);
```

The worker runs independently and forwards all output to the component.

### Attached Worker

```rust
let worker = MyWorker::builder()
    .launch(())
    .0;  // Returns (WorkerController<MyWorker>, ComponentParts)
```

Store the controller and use it to send messages.

## Sending Messages

```rust
// Send a message
worker.sender().send(MyWorkerMsg::DoSomething).unwrap();

// Or use emit
worker.emit(MyWorkerMsg::DoSomething);
```

## Worker Controller

The `WorkerController` provides:

```rust
let worker: WorkerController<MyWorker> = MyWorker::builder()
    .detach_worker(())
    .forward(...);

// Send messages
worker.sender().send(Msg::Hello).unwrap();

// Access the worker (for testing)
let _ = worker.widget();  // Returns ()
```

## Common Patterns

### Database Worker

```rust
struct DatabaseWorker {
    pool: SqlitePool,
}

#[derive(Debug)]
enum DbMsg {
    Query(String),
    Execute(String),
}

#[derive(Debug)]
enum DbOutput {
    QueryResult(Vec<Row>),
    Executed,
    Error(String),
}

impl Worker for DatabaseWorker {
    type Init = ConnectionString;
    type Input = DbMsg;
    type Output = DbOutput;

    fn init(conn_string: Self::Init, _sender: ComponentSender<Self>) -> Self {
        let pool = SqlitePool::connect(&conn_string).unwrap();
        DatabaseWorker { pool }
    }

    fn update(&mut self, msg: Self::Input, sender: ComponentSender<Self>) {
        match msg {
            DbMsg::Query(sql) => {
                match self.pool.query(&sql) {
                    Ok(rows) => sender.output(DbOutput::QueryResult(rows)).unwrap(),
                    Err(e) => sender.output(DbOutput::Error(e.to_string())).unwrap(),
                }
            }
            DbMsg::Execute(sql) => {
                self.pool.execute(&sql);
                sender.output(DbOutput::Executed).unwrap();
            }
        }
    }
}
```

### File Processing Worker

```rust
struct FileWorker {
    processed_count: usize,
}

#[derive(Debug)]
enum FileMsg {
    ProcessFile(PathBuf),
    ProcessBatch(Vec<PathBuf>),
}

#[derive(Debug)]
enum FileOutput {
    Processed(PathBuf),
    BatchComplete(usize),
}

impl Worker for FileWorker {
    type Init = ();
    type Input = FileMsg;
    type Output = FileOutput;

    fn init(_: Self::Init, _sender: ComponentSender<Self>) -> Self {
        FileWorker { processed_count: 0 }
    }

    fn update(&mut self, msg: Self::Input, sender: ComponentSender<Self>) {
        match msg {
            FileMsg::ProcessFile(path) => {
                process_file(&path);
                self.processed_count += 1;
                sender.output(FileOutput::Processed(path)).unwrap();
            }
            FileMsg::ProcessBatch(paths) => {
                for path in paths {
                    process_file(&path);
                    self.processed_count += 1;
                }
                sender.output(FileOutput::BatchComplete(self.processed_count)).unwrap();
            }
        }
    }
}
```

## Summary

- Workers are components without UI that run on a separate thread
- Use `Worker` trait for simple sequential processing
- Use `WorkerController` to send messages to workers
- Workers are ideal for CPU-bound or blocking tasks
- They maintain state between messages
