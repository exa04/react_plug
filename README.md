# ![React-Plug](https://github.com/user-attachments/assets/99590d0e-68c7-4363-a21a-94e38cae60e1)

<div align="center">
  
[![Test](https://github.com/223230/react_plug/actions/workflows/test.yml/badge.svg)](https://github.com/223230/react_plug/actions/workflows/test.yml)
![Lines of Code](https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fapi.codetabs.com%2Fv1%2Floc%2F%3Fgithub%3D223230%2Freact_plug%26branch%3Dmain&query=%24%5B%3F(%40.language%3D%3D%22Rust%22)%5D.linesOfCode&label=Lines%20of%20Code&labelColor=gray&color=blue)
[![dependency status](https://deps.rs/repo/github/223230/react_plug/status.svg)](https://deps.rs/repo/github/223230/react_plug)
[![0.1.0 milestone counter](https://img.shields.io/github/milestones/progress-percent/223230/react_plug/1)](https://github.com/223230/react_plug/milestone/1)
</div>

---

> [!CAUTION]
> **Turn away before it's too late!**
> 
> You've stumbled upon this project at an extremely early stage! Re-visit it when
> it's more mature. Right now, I'm just trying things out and the whole project
> might end up going in a totally different direction. That, or I'll just lose
> interest.

> [!WARNING]
> This project and `nih-plug-webview` are both a very early stage. They are
> definitely **not production-ready**! If you want to want something more mature,
> give JUCE 8's [WebView UIs](https://juce.com/blog/juce-8-feature-overview-webview-uis/) a try.

React-Plug is a crate that allows you to build Rust audio plug-ins with React GUIs
using [nih-plug](https://github.com/robbert-vdh/nih-plug) and [nih-plug-webview](https://github.com/httnn/nih-plug-webview). It bundles and includes your
React GUI, automatically generates [TypeScript](https://typescriptlang.org) bindings for it, handles
plugin-to-GUI communication and more. It strives to be a batteries-included,
opinionated, easy-to-use framework. Here are some of its standout features:

  - [x] A macro for easy parameter generation
  - [x] Automatically generated TypeScript bindings
  - [x] Support for custom GUI messages
  - [ ] Hot-reloading (dev mode)

## 🎚️Parameters

Your plug-in communicates parameter changes to the GUI by deriving message types and
parameter type enums for which the GUI receives TypeScript bindings. The DX is
great, but this approach leads to a lot of boilerplate. To cut down on that, you
will be defining your parameters once using a DSL (domain specific language). The
`rp-params!` macro then handles all the fluff. Here's an example:

```rust
rp_params! {
    ExampleParams {
        gain: FloatParam {
            name: "Gain",
            value: util::db_to_gain(0.0),
            range: FloatRange::Skewed {
                min: util::db_to_gain(-30.0),
                max: util::db_to_gain(30.0),
                factor: FloatRange::gain_skew_factor(-30.0, 30.0),
            },
            smoother: SmoothingStyle::Logarithmic(50.0),
            unit: " dB",
            value_to_string: formatters::v2s_f32_gain_to_db(2),
            string_to_value: formatters::s2v_f32_gain_to_db(),
        },
        muted: BoolParam {
            name: "Muted",
            value: false
        },
    }
}
```

You can see that the parameters are declared along with their types, default values,
and more properties, such as the range, smoother, and formatters. You can specify as
many parameters as you like, as long as you include the necessary bits. Your
parameter config then gets turned into a parameter struct, in this case
`ExampleParams`. Some boilerplate implementations are generated for it, as well as
a `new` function. Out of the box, the parameters work with the GUI and the editor.

> [!NOTE]
> As nice as this is, one downside of it is that the `new` function of your
> parameters will be generated by React-Plug and you can really only define your
> parameter defaults inside the `rp_params!` macro. You also won't be able to add
> other fields to your params struct. I'd love for there to be a more flexible
> solution. If any proc macro wizard/witch/magician wants to help me out with this,
> file a pull request, create an issue, or cast a spell on my codebase 🧙

## 💬 Messages

The editor and GUI communicate parameter changes out of the box. For custom
messages, you can add more variants to your plugin's message enums. The attribute
macros `gui_message` and `plugin_message` handle the necessary boilerplate and add
the aforementioned built-in messages for you. Here's an example:

```rust
#[gui_message(params = ExampleParams)]
pub enum GuiMessage {
    Ping,
}

#[plugin_message(params = ExampleParams)]
pub enum PluginMessage {
    Pong,
}
```

Now, you can send the `Ping` message from the GUI.

[//]: # (TODO: Example for dispatching inside React)

Then, handle the `Ping` message inside React-Plug's `create_editor` function, by
responding with a `Pong` message - To do this, just modify the closure you pass to
`create_editor` like so:

```diff
    fn editor(&mut self, _async_executor: AsyncExecutor<Self>) -> Option<Box<dyn Editor>> {
        Some(Box::new(create_editor(
            ...
            |gui_message: GuiMessage, sender| {
+               // React to the `Ping` message by sending a `Pong` message.
+               if let GuiMessage::Ping = gui_message {
+                   sender.send(PluginMessage::Pong).unwrap();
+               }
            },
        )))
    }
```
