# Widget Templates

Widget templates define reusable widget hierarchies. They're perfect for creating consistent UI patterns across your application.

## When to Use Widget Templates

- Reusable button bars
- Card layouts
- Form fields
- Navigation elements
- Any repeated widget patterns

## Basic Template Example

Here's a simple card template:

```rust
use gtk::prelude::*;
use relm4::{ComponentParts, ComponentSender, RelmApp, SimpleComponent, WidgetTemplate};

#[derive(WidgetTemplate)]
struct CardTemplate {
    #[template_child]
    header: gtk::Label,
    #[template_child]
    content: gtk::Label,
}

impl CardTemplate {
    fn new(title: &str) -> Self {
        let box_widget = gtk::Box::builder()
            .orientation(gtk::Orientation::Vertical)
            .spacing(5)
            .margin_all(10)
            .css_classes(&["card"])
            .build();

        let header = gtk::Label::new(Some(title));
        header.add_css_class("card-header");

        let content = gtk::Label::new(Some("Content"));
        content.add_css_class("card-content");

        box_widget.append(&header);
        box_widget.append(&content);

        // Create template from the box widget
        let template = Self {
            header,
            content,
        };

        template
    }

    fn set_title(&self, title: &str) {
        self.header.set_label(title);
    }

    fn set_content(&self, content: &str) {
        self.content.set_label(content);
    }
}

struct AppModel {
    card1: gtk::Box,
    card2: gtk::Box,
}

#[derive(Debug)]
enum AppMsg {
    UpdateCard1,
    UpdateCard2,
}

struct AppWidgets {
    container: gtk::Box,
}

impl SimpleComponent for AppModel {
    type Init = ();
    type Input = AppMsg;
    type Output = ();
    type Widgets = AppWidgets;
    type Root = gtk::Window;

    fn init_root() -> Self::Root {
        gtk::Window::builder()
            .title("Widget Template Example")
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
            card1: gtk::Box::new(gtk::Orientation::Vertical, 10),
            card2: gtk::Box::new(gtk::Orientation::Vertical, 10),
        };

        let container = gtk::Box::builder()
            .orientation(gtk::Orientation::Vertical)
            .spacing(10)
            .margin_all(20)
            .build();

        let card1_title = gtk::Label::new(Some("Card 1"));
        card1_title.add_css_class("card-header");
        let card1_content = gtk::Label::new(Some("First card content"));
        card1_content.add_css_class("card-content");

        model.card1.append(&card1_title);
        model.card1.append(&card1_content);
        model.card1.add_css_class("card");

        let card2_title = gtk::Label::new(Some("Card 2"));
        card2_title.add_css_class("card-header");
        let card2_content = gtk::Label::new(Some("Second card content"));
        card2_content.add_css_class("card-content");

        model.card2.append(&card2_title);
        model.card2.append(&card2_content);
        model.card2.add_css_class("card");

        let update_btn1 = gtk::Button::with_label("Update Card 1");
        let update_btn2 = gtk::Button::with_label("Update Card 2");

        window.set_child(Some(&container));
        container.append(&model.card1);
        container.append(&model.card2);
        container.append(&update_btn1);
        container.append(&update_btn2);

        update_btn1.connect_clicked(move |_| {
            sender.input(AppMsg::UpdateCard1);
        });

        update_btn2.connect_clicked(move |_| {
            sender.input(AppMsg::UpdateCard2);
        });

        let widgets = AppWidgets { container };

        ComponentParts { model, widgets }
    }

    fn update(&mut self, msg: Self::Input, _sender: ComponentSender<Self>) {
        match msg {
            AppMsg::UpdateCard1 => {
                // Update card 1 content
            }
            AppMsg::UpdateCard2 => {
                // Update card 2 content
            }
        }
    }
}

fn main() {
    let app = RelmApp::new("org.relm4.WidgetTemplateExample");
    app.run::<AppModel>(());
}
```

## Template Child Pattern

The `#[template_child]` attribute marks fields as accessible from the template:

```rust
#[derive(WidgetTemplate)]
struct MyTemplate {
    #[template_child]
    pub header: gtk::Label,
    
    #[template_child]
    pub content: gtk::Box,
    
    // Private field (not accessible from outside)
    internal_button: gtk::Button,
}
```

## Template with Nested Structure

Templates can have deeply nested widget trees:

```rust
struct ComplexCard {
    root: gtk::Box,
    header_box: gtk::Box,
    title: gtk::Label,
    subtitle: gtk::Label,
    body: gtk::Box,
    footer: gtk::Button,
}

impl ComplexCard {
    fn new() -> Self {
        let root = gtk::Box::builder()
            .orientation(gtk::Orientation::Vertical)
            .spacing(0)
            .build();

        let header_box = gtk::Box::builder()
            .orientation(gtk::Orientation::Vertical)
            .spacing(4)
            .margin_top(12)
            .margin_start(12)
            .margin_end(12)
            .build();

        let title = gtk::Label::new(None);
        title.add_css_class("title");
        title.set_halign(gtk::Align::Start);

        let subtitle = gtk::Label::new(None);
        subtitle.add_css_class("subtitle");
        subtitle.set_halign(gtk::Align::Start);

        header_box.append(&title);
        header_box.append(&subtitle);

        let body = gtk::Box::builder()
            .orientation(gtk::Orientation::Vertical)
            .spacing(8)
            .margin_top(12)
            .margin_start(12)
            .margin_end(12)
            .margin_bottom(12)
            .build();

        let footer = gtk::Button::with_label("Action");
        footer.set_halign(gtk::Align::End);
        footer.set_margin_top(8);
        footer.set_margin_end(12);
        footer.set_margin_bottom(12);

        root.append(&header_box);
        root.append(&body);
        root.append(&footer);

        ComplexCard {
            root,
            header_box,
            title,
            subtitle,
            body,
            footer,
        }
    }

    fn set_title(&self, title: &str) {
        self.title.set_label(title);
    }

    fn set_subtitle(&self, subtitle: &str) {
        self.subtitle.set_label(subtitle);
    }

    fn add_to_body(&self, widget: &gtk::Widget) {
        self.body.append(widget);
    }

    fn on_action<F>(&self, callback: F)
    where
        F: Fn() + 'static,
    {
        self.footer.connect_clicked(move |_| callback());
    }
}
```

## Template with Dynamic Content

Templates can be used as containers for dynamic content:

```rust
struct FormField {
    root: gtk::Box,
    label: gtk::Label,
    input: gtk::Entry,
    error_label: gtk::Label,
}

impl FormField {
    fn new(field_label: &str) -> Self {
        let root = gtk::Box::builder()
            .orientation(gtk::Orientation::Vertical)
            .spacing(4)
            .build();

        let label = gtk::Label::new(Some(field_label));
        label.set_halign(gtk::Align::Start);

        let input = gtk::Entry::new();

        let error_label = gtk::Label::new(None);
        error_label.add_css_class("error");
        error_label.set_visible(false);

        root.append(&label);
        root.append(&input);
        root.append(&error_label);

        FormField {
            root,
            label,
            input,
            error_label,
        }
    }

    fn set_value(&self, value: &str) {
        self.input.set_text(value);
    }

    fn get_value(&self) -> String {
        self.input.text().to_string()
    }

    fn set_error(&self, error: Option<&str>) {
        if let Some(e) = error {
            self.error_label.set_label(e);
            self.error_label.set_visible(true);
            self.input.add_css_class("error");
        } else {
            self.error_label.set_visible(false);
            self.input.remove_css_class("error");
        }
    }

    fn connect_changed<F>(&self, f: F)
    where
        F: Fn(String) + 'static,
    {
        let entry = self.input.clone();
        self.input.connect_changed(move |_| {
            f(entry.text().to_string());
        });
    }
}
```

## Template Collections

Create multiple instances of a template:

```rust
struct AppModel {
    fields: Vec<FormField>,
}

impl SimpleComponent for AppModel {
    fn init(...) -> ComponentParts<Self> {
        let mut fields = Vec::new();
        
        // Create form fields
        let name_field = FormField::new("Name");
        let email_field = FormField::new("Email");
        let phone_field = FormField::new("Phone");
        
        fields.push(name_field);
        fields.push(email_field);
        fields.push(phone_field);
        
        // Add to form container
        for field in &fields {
            form_container.append(field.widget());
        }
        
        AppModel { fields }
    }
}
```

## Summary

- Widget templates define reusable widget hierarchies
- Use `#[template_child]` to mark accessible widgets
- Templates can be simple or deeply nested
- Templates support dynamic content via container widgets
- Create multiple instances for repeated patterns
