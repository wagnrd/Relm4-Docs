# CLI Arguments

Relm4 applications can handle command-line arguments using the `clap` crate or GTK's built-in argument handling.

## GTK Command-Line Options

GTK provides standard options:

```bash
# Display help
myapp --help

# GTK options
myapp --gdk-debug=events
myapp --gtk-debug=interactive
myapp --sync

# X11 specific
myapp --display=:0
```

## Custom CLI Arguments

### Using Clap

Add to `Cargo.toml`:

```toml
[dependencies]
clap = { version = "4", features = ["derive"] }
```

### Basic Arguments

```rust
use clap::Parser;

#[derive(Parser, Debug)]
#[command(name = "myapp")]
struct Args {
    /// Enable verbose output
    #[arg(short, long)]
    verbose: bool,

    /// Set the config file
    #[arg(short, long, default_value = "config.json")]
    config: String,

    /// Input file
    input: Option<String>,
}

fn main() {
    let args = Args::parse();
    
    println!("Verbose: {}", args.verbose);
    println!("Config: {}", args.config);
    println!("Input: {:?}", args.input);
}
```

### Subcommands

```rust
use clap::{Parser, Subcommand};

#[derive(Parser, Debug)]
#[command(name = "myapp")]
struct Args {
    #[command(subcommand)]
    command: Option<Commands>,
}

#[derive(Subcommand, Debug)]
enum Commands {
    /// Run the application
    Run {
        /// Open in fullscreen
        #[arg(short, long)]
        fullscreen: bool,
    },
    /// Export data
    Export {
        /// Output file
        #[arg(short, long)]
        output: String,
        /// Format (json, csv)
        #[arg(short, long, default_value = "json")]
        format: String,
    },
    /// Show version
    Version,
}

fn main() {
    let args = Args::parse();
    
    match args.command {
        Some(Commands::Run { fullscreen }) => {
            run_app(fullscreen);
        }
        Some(Commands::Export { output, format }) => {
            export_data(&output, &format);
        }
        Some(Commands::Version) => {
            println!("myapp v{}", env!("CARGO_PKG_VERSION"));
        }
        None => {
            run_app(false);
        }
    }
}
```

## Integrating with Relm4

### Full Example

```rust
use clap::Parser;
use gtk::prelude::*;
use relm4::{ComponentParts, ComponentSender, RelmApp, SimpleComponent};

#[derive(Parser, Debug)]
#[command(name = "relm4-app")]
struct Args {
    /// Enable dark mode
    #[arg(short, long)]
    dark: bool,

    /// Set window title
    #[arg(short, long, default_value = "My App")]
    title: String,

    /// Window width
    #[arg(long, default_value = "800")]
    width: u32,

    /// Window height
    #[arg(long, default_value = "600")]
    height: u32,

    /// Verbose output
    #[arg(short, long)]
    verbose: bool,
}

struct AppModel {
    title: String,
    dark_mode: bool,
}

#[derive(Debug)]
enum AppMsg {
    ToggleTheme,
}

impl SimpleComponent for AppModel {
    type Init = Args;
    type Input = AppMsg;
    type Output = ();
    type Widgets = AppWidgets;
    type Root = gtk::Window;

    fn init_root() -> Self::Root {
        gtk::Window::builder()
            .build()
    }

    fn init(
        args: Self::Init,
        window: Self::Root,
        _sender: ComponentSender<Self>,
    ) -> ComponentParts<Self> {
        let model = AppModel {
            title: args.title,
            dark_mode: args.dark,
        };

        // Apply arguments to window
        window.set_title(&model.title);
        window.set_default_size(args.width as i32, args.height as i32);

        // Apply dark mode if requested
        if model.dark_mode {
            apply_dark_mode(&window, true);
        }

        let vbox = gtk::Box::builder()
            .orientation(gtk::Orientation::Vertical)
            .spacing(20)
            .margin_all(20)
            .build();

        let label = gtk::Label::new(Some(&format!(
            "Welcome to {}!\nDark mode: {}",
            model.title,
            if model.dark_mode { "on" } else { "off" }
        )));

        let toggle_btn = gtk::Button::with_label("Toggle Theme");

        window.set_child(Some(&vbox));
        vbox.append(&label);
        vbox.append(&toggle_btn);

        let widgets = AppWidgets { label, toggle_btn };

        ComponentParts { model, widgets }
    }

    fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
        match msg {
            AppMsg::ToggleTheme => {
                self.dark_mode = !self.dark_mode;
            }
        }
    }

    fn update_view(&self, widgets: &mut Self::Widgets, _sender: ComponentSender<Self>) {
        widgets.label.set_label(&format!(
            "Welcome to {}!\nDark mode: {}",
            self.title,
            if self.dark_mode { "on" } else { "off" }
        ));
    }
}

fn apply_dark_mode(window: &gtk::Window, dark: bool) {
    let display = window.display();
    if let Some(settings) = gtk::Settings::for_display(&display) {
        settings.set_gtk_application_prefer_dark_theme(dark);
    }
}

fn main() {
    let args = Args::parse();

    if args.verbose {
        eprintln!("Starting {}...", args.title);
        eprintln!("Window size: {}x{}", args.width, args.height);
        eprintln!("Dark mode: {}", args.dark);
    }

    let app = RelmApp::new("org.example.myapp");
    app.run::<AppModel>(args);
}
```

## GTK Settings from CLI

```rust
fn apply_settings_from_args(args: &Args) {
    // These affect the entire application
    if args.dark {
        // Set dark theme preference
        std::env::set_var("GTK_THEME", "Adwaita:dark");
    }
}
```

## Environment Variables

GTK respects several environment variables:

```bash
# Theme
GTK_THEME=Adwaita:dark
GTK_THEME=dark

# Icon theme
GTK_ICON_THEME=Adwaita

# Debug
GDK_DEBUG=events
GTK_DEBUG=interactive

# Backend
GDK_BACKEND=x11
GDK_BACKEND=wayland
GDK_BACKEND= Broadway (for HTML)
```

## Summary

- Use `clap` for custom command-line arguments
- Pass parsed arguments as the component's `Init` type
- Apply arguments in `init()` or `init_root()`
- GTK also respects environment variables for settings
- Combine with GTK's built-in options for flexibility
