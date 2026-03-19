# Resource Bundles

Resource bundles let you package icons, images, and other assets with your application.

## When to Use Resources

- Application icons
- UI images and icons
- Embedded data files
- Themes and stylesheets

## Creating a Resource Bundle

### 1. Create a Resources XML File

Create `resources/myapp.gresource.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<gresources>
  <gresource prefix="/com/example/myapp">
    <!-- UI files -->
    <file>ui/main.ui</file>
    <file>ui/dialog.ui</file>
    
    <!-- Icons -->
    <file compressed="true">icons/app-icon.png</file>
    <file>icons/toolbar/new.svg</file>
    <file>icons/toolbar/open.svg</file>
    <file>icons/toolbar/save.svg</file>
    
    <!-- CSS -->
    <file>styles/main.css</file>
  </gresource>
</gresources>
```

### 2. Compile Resources

Use `glib-compile-resources` to compile the bundle:

```bash
# Compile resources
glib-compile-resources --target=resources/myapp.gresource \
    resources/myapp.gresource.xml

# With dependencies (images, etc.)
glib-compile-resources --target=resources/myapp.gresource \
    --sourcedir=resources \
    resources/myapp.gresource.xml
```

### 3. Include in Cargo.toml

```toml
[package]
# ...
build = "build.rs"

[build-dependencies]
relm4-icons-build = "0.10"  # Optional: for icon handling

[dependencies]
gio = { version = "0.10", package = "gtk4", features = ["vfs"] }
```

### 4. Create build.rs

```rust
// build.rs
fn main() {
    println!("cargo:rerun-if-changed=resources/myapp.gresource.xml");
    
    // Optionally regenerate resources
    // let _ = std::process::Command::new("glib-compile-resources")
    //     .args(&["--target=src/resources.rs", "resources/myapp.gresource.xml"])
    //     .status();
}
```

## Loading Resources

### Register the Resource

```rust
use gtk::gio;

fn main() {
    // Load and register resources
    let resource = gio::Resource::load("resources/myapp.gresource")
        .expect("Failed to load resources");
    gio::resources_register(&resource);
    
    // Or from embedded data
    // let bytes = include_bytes!("../resources/myapp.gresource");
    // let resource = gio::Resource::from_bytes(&bytes)
    //     .expect("Failed to load resources");
    // gio::resources_register(&resource);
    
    RelmApp::new("org.example.myapp").run::<AppModel>()
}
```

### Access Resources

```rust
use gtk::gio;

// Load an image from resources
let pixbuf = gtk::gdk_pixbuf::Pixbuf::from_resource(
    "/com/example/myapp/icons/app-icon.png"
).expect("Failed to load icon");

// Or use GdkTexture
let texture = gtk::gdk::Texture::from_resource(
    "/com/example/myapp/icons/app-icon.png"
).expect("Failed to load texture");

let image = gtk::Image::from_texture(&texture);
```

## Loading UI Files

GTK allows defining UI in XML:

```xml
<!-- ui/main_window.ui -->
<?xml version="1.0"?>
<interface>
  <object class="GtkWindow" id="main-window">
    <property name="title">My App</property>
    <property name="default-width">600</property>
    <property name="default-height">400</property>
    <child>
      <object class="GtkBox">
        <property name="orientation">vertical</property>
        <child>
          <object class="GtkLabel" id="title-label">
            <property name="label">Welcome</property>
          </object>
        </child>
      </object>
    </child>
  </object>
</interface>
```

### Load UI in Rust

```rust
use gtk::gio;

fn load_ui() -> gtk::Builder {
    let builder = gtk::Builder::new();
    
    builder.add_from_resource("/com/example/myapp/ui/main.ui")
        .expect("Failed to load UI");
    
    builder
}

// Use the builder
fn init_window() {
    let builder = load_ui();
    
    let window: gtk::Window = builder
        .object("main-window")
        .expect("Failed to get main-window")
        .downcast()
        .expect("main-window is not a window");
    
    let label: gtk::Label = builder
        .object("title-label")
        .expect("Failed to get title-label")
        .downcast()
        .expect("title-label is not a label");
    
    // Use window and widgets...
}
```

## Loading CSS

```rust
use gtk::gdk;

fn load_css() {
    let provider = gtk::CssProvider::new();
    
    // Load from file
    // provider.load_from_path("styles/main.css");
    
    // Load from resources
    let display = gdk::Display::default().unwrap();
    gtk::StyleContext::add_provider_for_display(
        &display,
        &provider,
        gtk::STYLE_PROVIDER_PRIORITY_APPLICATION,
    );
}
```

## Complete Example

### Directory Structure

```
myapp/
├── Cargo.toml
├── build.rs
├── resources/
│   ├── myapp.gresource.xml
│   ├── icons/
│   │   ├── app-icon.png
│   │   └── toolbar/
│   │       ├── new.svg
│   │       └── save.svg
│   └── css/
│       └── style.css
└── src/
    └── main.rs
```

### resources/myapp.gresource.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<gresources>
  <gresource prefix="/com/example/myapp">
    <file>icons/app-icon.png</file>
    <file>icons/toolbar/new.svg</file>
    <file>icons/toolbar/save.svg</file>
    <file>css/style.css</file>
  </gresource>
</gresources>
```

### build.rs

```rust
fn main() {
    println!("cargo:rerun-if-changed=resources/myapp.gresource.xml");
}
```

### src/main.rs

```rust
use gtk::prelude::*;
use relm4::{ComponentParts, ComponentSender, RelmApp, SimpleComponent};

struct AppModel {
    icon_name: String,
}

#[derive(Debug)]
enum AppMsg {
    SetIcon(String),
}

impl SimpleComponent for AppModel {
    type Init = ();
    type Input = AppMsg;
    type Output = ();
    type Widgets = AppWidgets;
    type Root = gtk::Window;

    fn init_root() -> Self::Root {
        gtk::Window::builder()
            .title("Resource Bundle Example")
            .default_width(400)
            .default_height(300)
            .build()
    }

    fn init(
        _: Self::Init,
        window: Self::Root,
        _sender: ComponentSender<Self>,
    ) -> ComponentParts<Self> {
        // Load CSS from resources
        relm4::set_global_css_from_file("resources/css/style.css")
            .expect("Failed to load CSS");

        let model = AppModel {
            icon_name: "document-new".to_string(),
        };

        let vbox = gtk::Box::builder()
            .orientation(gtk::Orientation::Vertical)
            .spacing(20)
            .margin_all(20)
            .build();

        // Load icon from resources
        let icon = gtk::Image::from_icon_name("document-new");
        icon.set_icon_size(gtk::IconSize::Large);

        // Or load as texture
        if let Ok(texture) = gtk::gdk::Texture::from_resource(
            "/com/example/myapp/icons/app-icon.png"
        ) {
            let image = gtk::Image::from_texture(&texture);
            vbox.append(&image);
        }

        let button = gtk::Button::with_label("Change Icon");

        window.set_child(Some(&vbox));
        vbox.append(&icon);
        vbox.append(&button);

        let widgets = AppWidgets { icon };

        ComponentParts { model, widgets }
    }

    fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
        match msg {
            AppMsg::SetIcon(name) => {
                self.icon_name = name;
            }
        }
    }

    fn update_view(&self, widgets: &mut Self::Widgets, _sender: ComponentSender<Self>) {
        widgets.icon.set_icon_name(Some(&self.icon_name));
    }
}

fn main() {
    // Register resources before running app
    let resource = gtk::gio::Resource::load("resources/myapp.gresource")
        .expect("Failed to load resources");
    gtk::gio::resources_register(&resource);

    let app = RelmApp::new("org.example.myapp");
    app.run::<AppModel>(());
}
```

## Summary

- Resource bundles package assets with your application
- Use `gresource.xml` to define resources
- Compile with `glib-compile-resources`
- Register with `gio::resources_register()`
- Access with `gtk::gdk::Texture::from_resource()` or similar
- Works for icons, UI files, CSS, and any binary data
