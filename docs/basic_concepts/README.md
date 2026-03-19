# Basic Concepts

Before we start building our app, we need to understand the basic concepts of Relm4. This section explains in detail how Relm4 works and how to use it.

After this section, you will be ready to build your first Relm4 application.

## Topics Covered

1. **[Model](model.md)** - The data structure that stores your application state
2. **[Messages](messages.md)** - Events that trigger state changes
   - [Input Messages](messages/input.md) - Messages received by components
   - [Output Messages](messages/output.md) - Messages sent to parent components
3. **[Widgets](widgets.md)** - GTK4 visual elements that make up your UI
4. **[Components](components.md)** - Self-contained pieces of UI logic

## The Elm Architecture

Relm4 follows the Elm architecture, which separates your application into three distinct parts:

```
┌─────────────────────────────────────────────────────────┐
│                     COMPONENT                           │
│                                                         │
│  ┌─────────┐      ┌──────────┐      ┌─────────────┐   │
│  │  Model  │ ←──→ │  Update  │ ←──→ │    View     │   │
│  │ (State) │      │ (Logic)  │      │  (Widgets) │   │
│  └─────────┘      └──────────┘      └─────────────┘   │
│       ↑                │                   │           │
│       │                │                   │           │
│       └────────────────┴───────────────────┘           │
│                      Messages                           │
└─────────────────────────────────────────────────────────┘
```

### How it works:

1. **User interacts** with the UI (clicks a button, types text, etc.)
2. A **message** is sent describing what happened
3. The **update** function processes the message and modifies the **model**
4. The **view** is updated to reflect the new model state

This architecture makes applications:
- **Predictable**: State changes always follow the same pattern
- **Testable**: Business logic is separated from UI
- **Maintainable**: Clear boundaries between components

## Next Steps

Continue to [Your First App](../getting_started/first_app.md) to build a practical example!
