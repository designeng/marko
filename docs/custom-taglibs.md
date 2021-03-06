Custom Taglibs
==============

# Tag Renderer

Every tag should be mapped to an object with a `render(input, out)` function. The render function is just a function that takes two arguments: `input` and `out`. The `input` argument is an arbitrary object that contains the input data for the renderer. The `out` argument is an [asynchronous writer](https://github.com/marko-js/async-writer) that wraps an output stream. Output can be produced using `out.write(someString)` There is no class hierarchy or tie-ins to Marko when implementing a tag renderer. A simple tag renderer is shown below:

```javascript
exports.render = function(input, out) {
    out.write('Hello ' + input.name + '!');
}
```

If, and only if, a tag has nested content, then a special `renderBody` method will be added to the `input` object. If a renderer wants to render the nested body content then it must call the `renderBody` method. For example:

```javascript
exports.render = function(input, out) {
    out.write('BEFORE BODY');
    if (input.renderBody) {
        input.renderBody(out);
    }
    out.write('AFTER BODY');
}
```
For users of Marko Widgets: Invoking `input.renderBody` is equivalent to using the `w-body` attribute for tags (in conjunction with the `getInitialBody()` lifecycle method; see [getInitialBody()](https://github.com/marko-js/marko-widgets#getinitialbodyinput-out)).

A tag renderer should be mapped to a custom tag by creating a `marko-taglib.json` as shown in the next few sections.

# marko-taglib.json

## Sample Taglib

```json
{
    "tags": {
        "my-hello": {
            "renderer": "./hello-renderer",
            "attributes": {
                "name": "string"
            }
        }
    }
}
```

Marko also supports a short-hand for declaring tags and attributes. The following `marko-taglib.json` is equivalent to the `marko-taglib.json` above:

```json
{
    "<my-hello>": {
        "renderer": "./hello-renderer",
        "@name": "string"
    }
}
```

The short-hand will be used for the remaining of this documentation.

# Defining Tags

Tags can be defined by adding `"<tag_name>": <tag_def>` properties to your `marko-taglib.json`:

```json
{
    "<my-hello>": {
        "renderer": "./hello-renderer",
        "@name": "string"
    },
    "<my-foo>": {
        "renderer": "./foo-renderer",
        "@*": "string"
    },
    "<my-bar>": "./path/to/my-bar/marko-tag.json",
    "<my-baz>": {
        "template": "./baz-template.marko"
    },
}
```

Every tag should be associated with a renderer or a template. When a custom tag is used in a template, the renderer (or template) will be invoked at render time to produce the HTML/output. If a `String` path to a `marko-tag.json` for a custom tag then the target `marko-tag.json` is loaded to define the tag.

# Defining Attributes

If you provide attributes then the Marko compiler will do validation to make sure only the supported attributes are provided. A wildcard attribute (`"@*"`) allows any attribute to be passed in. Below are sample attribute definitions:

_Multiple attributes:_

```javascript
{
    "@message": "string",     // String
    "@my-data": "expression", // JavaScript expression
    "@*": "string"            // Everything else will be added to a special "*" property
}
```

# Scanning for Tags

Marko supports a directory scanner to make it easier to maintain a taglib by introducing a few conventions:

* The name of the tag directory will be the name of the tag
* One tag per directory
* All tag directories should be direct children of a parent directory
* Every tag directory must contain a `renderer.js` that is used as the tag renderer or, alternatively, a `template.marko`
* Each tag directory may contain a `marko-tag.json` file or the tag definition can be embedded into `renderer.js`

With this approach, `marko-taglib.json` will be much simpler:

```json
{
    "tags-dir": "./components"
}
```
Given the following directory structure:

* __components/__
    * __my-hello/__
        * renderer.js
    * __my-foo/__
        * template.marko
    * __my-bar/__
        * renderer.js
        * marko-tag.json
* marko-taglib.json

The following three tags will be exported:

* `<my-hello>`
* `<my-foo>`
* `<my-bar>`

Directory scanning only supports one tag per directory and it will only look at directories one level deep. The tag definition can be embedded into the `renderer.js` file or it can be put into a separate `marko-tag.json`. For example:

_In `renderer.js`:_

```javascript
exports.tag = {
    "@name": "string"
}
```

_In `marko-tag.json`:_

```javascript
{
    "@name": "string"
}
```

_NOTE: It is not necessary to declare the `renderer` since the scanner will automatically use `renderer.js` as the renderer._

`tags-dir` also accepts an array if you have taglibs organized in multiple folders.

```json
{
    "tags-dir": ["./components", "./modules"]
}
```

# Nested Tags

It is often necessary for tags to have a parent/child or ancestor/descendent relationship. For example:

```xml
<ui-tabs orientation="horizontal">
    <ui-tabs.tab title="Home">
        Content for Home
    </ui-tabs.tab>
    <ui-tabs.tab title="Profile">
        Content for Profile
    </ui-tabs.tab>
    <ui-tabs.tab title="Messages">
        Content for Messages
    </ui-tabs.tab>
</ui-tabs>
```

Nested tags can be declared in the parent tag's `marko-tag.json` as shown below:

___ui-tabs/marko-tag.json___

```json
{
    "@orientation": "string",
    "@tabs <tab>[]": {
        "@title": "string"
    }
}
```

This allows a `tabs` to be provided using nested `<ui-tabs.tab>` tags or the tabs can be provided as a `tabs` attribute (e.g. `<ui-tabs tabs="[tab1, tab2, tab3]"`). The nested `<ui-tabs.tab>` tags will be made available to the renderer as part of the `tabs` property for the parent `<ui-tabs>`. Because of the `[]` suffix on `<tab>[]` the tabs property will be of type `Array` and not a single object. That is, the `[]` suffix is used to declare that a nested tag can be repeated. The sample renderer that accesses the nested tabs is shown below:

___ui-tabs/renderer.js___

```javascript
var template = require('./template.marko');

exports.renderer = function(input, out) {
    var tabs = input.tabs;

    // Tabs will be in the following form:
    // [
    //     {
    //         title: 'Home',
    //         renderBody: function(out) { ... }
    //     },
    //     {
    //         title: 'Profile',
    //         renderBody: function(out) { ... }
    //     },
    //     {
    //         title: 'Messages',
    //         renderBody: function(out) { ... }
    //     }
    // ]
    console.log(tabs.length); // Output: 3

    template.render({
        tabs: tabs
    }, out);

};
```

Finally, the template to render the `<ui-tabs>` component will be similar to the following:

___ui-tabs/template.marko___

```xml
<div class="tabs">
    <ul class="nav nav-tabs">
        <li class="tab" for="tab in data.tabs">
            <a href="#${tab.title}">
                ${tab.title}
            </a>
        </li>
    </ul>
    <div class="tab-content">
        <div class="tab-pane" for="tab in data.tabs">
            <invoke function="tab.renderBody(out)"/>
        </div>
    </div>
</div>
```

Below is an example of using nested tags that are not repeated:

```xml
<ui-overlay>
    <ui-overlay.header class="my-header">
        Header content
    </ui-overlay.header>

    <ui-overlay.body class="my-body">
        Body content
    </ui-overlay.body>

    <ui-overlay.footer class="my-footer">
        Footer content
    </ui-overlay.footer>
</ui-overlay>
```

The `marko-tag.json` for the `<ui-overlay>` tag will be similar to the following:

___ui-overlay/marko-tag.json___

```json
{
    "@header <header>": {
        "@class": "string"
    },
    "@body <body>": {
        "@class": "string"
    },
    "@footer <footer>": {
        "@class": "string"
    }
}
```

The renderer for the `<ui-overlay>` tag will be similar to the following:

```javascript
var template = require('./template.marko');

exports.renderer = function(input, out) {
    var header = input.header;
    var body = input.body;
    var footer = input.footer;

    // NOTE: header, body and footer will be of the following form:
    //
    // {
    //     'class': 'my-header',
    //     renderBody: function(out) { ... }
    // }

    template.render({
        header: header,
        body: body,
        footer: footer
    }, out);

};
```

Finally, the sample template to render the `<ui-overlay>` tag is shown below:

```xml
<div class="overlay">
    <!-- Header -->
    <div class="overlay-header ${data.header['class']}" if="data.header">
        <invoke function="data.header.renderBody(out)"/>
    </div>

    <!-- Body -->
    <div class="overlay-body ${data.body['class']}" if="data.body">
        <invoke function="data.body.renderBody(out)"/>
    </div>

    <!-- Footer -->
    <div class="overlay-footer ${data.footer['class']}" if="data.footer">
        <invoke function="data.footer.renderBody(out)"/>
    </div>
</div>
```

# Taglib Discovery

Given a template file, the `marko` module will automatically discover all taglibs by searching relative to the template file. The taglib discoverer will search up and also look into `node_modules` to discover applicable taglibs.

As an example, given a template at path `/my-project/src/pages/login/template.marko`, the search path will be the following:

1. `/my-project/src/pages/login/marko-taglib.json`
2. `/my-project/src/pages/login/node_modules/*/marko-taglib.json`
3. `/my-project/src/pages/marko-taglib.json`
4. `/my-project/src/pages/node_modules/*/marko-taglib.json`
5. `/my-project/src/marko-taglib.json`
6. `/my-project/src/node_modules/*/marko-taglib.json`
7. `/my-project/marko-taglib.json`
8. `/my-project/node_modules/*/marko-taglib.json`