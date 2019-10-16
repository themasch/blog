+++
title = "OpenUI5 and JSX"
description = "Oh God What Have I Done?"
date = 2016-12-15
[taxonomies]
tags = ["web", "ui5", "dev"]
+++

Can we somewho bring together SAPs full-blown enterprise UI framework and facebooks monstrosity that is JSX?
<!-- more -->

For those of you who are not familiar with the technologies in the title, a short introduction:

OpenUI5 is the open-sourced version of SAPs new user interface framework SAPUI5. Its a “enterprise-ready” all-inclusive, batteries included UI framework that comes with full MVC, routing, UI controls, i18n support etc.
UI5 has a pretty strict separation of views and controllers. Every view defines the structure, used controls, data bindings, etc. and may declare a controller that should be used. Controllers should only be used to react to events fired by the view and update the models which, via the data binding, signal the view to refresh.

JSX (not to be confused with this JSX) on the other hand may be called a “syntax extension” to JavaScript introduced (as far as I know) by Facebook in their “React” framework. React itself does many things differently from other UI frameworks. Mainly, there is no differentiation between views and Controllers. There are just Components, and these are (in theory) pure functions. They receive an input (called “props”) and return an output (the resulting DOM).

In UI5, creating controls in the controller feels pretty verbose and hard to read:

```js
  myControllerFunction: function() {			
    var oContent = new sap.m.Input({
      value: this.getValue(),
      type: "Number",
      change: function(oEvent) {
        this.setValue(oEvent.getParameter("value"));
      }.bind(this)
    });

    return new sap.ui.layout.form.SimpleForm({
      editable: true,
      content: [
        new sap.m.Label({
          text: this.config.displayName
        }),
        oContent
      ]
    });
  }
```

(Yes, this can be done better by importing the needed classes in the local namespace, but I have yet to find a “good” solution)

While this is kind of okay in UI5, where you should write your views in XML, this would not work in React where you component is just a single JS function. That is what JSX is good for:

```js
function Item(props) {
  return <li>{props.message}</li>;
}

function TodoList() {
  const todos = ['finish doc', 'submit pr', 'nag dan to review'];
  return (
    <ul>
      {todos.map((message) => <Item key={message} message={message} />)}
    </ul>
  );
}
```

> Note: proper syntax highlighting for JSX is currently missing. sorry for that.

Yep, that’s right. We are mixing HTML and JS here. It may look scary at first but I promise you, its pretty awesome when you get used to it.

When I had walked my first steps with React and started to like JSX, I immediately missed it in UI5 whenever I had to create a control in controller code.
I dreamed about how awesome that could look, how clear it could be.

```js
  myControllerFunction: function() {			
    function onChange(oEvent) {
      this.setValue(oEvent.getParameter("value"));
    }
    return (
      <SimpleForm editable="true">
        <Label text={this.config.displayName} />
        <Input value={this.getValue()} type="Number" change={onChange.bind(this)} />
      </SimpleForm>
    )
  }
```

React relies on Babel to transform the JSX code to native JavaScript. Lets give it a try. There is babel-plugin-transform-jsx which solves this problem perfectly. JSX in, JS out.

To use it you need to install babel, the babel-cli and the plugin:

`$ npm install babel babel-cli babel-plugin-transform-jsx`

Additionally, I’d suggest adding a .babelrc file for configuration:

```json
{
  "plugins": [ "transform-jsx"]
}
```

This is just a JSON file containing the configuration for babel.

Now, you can just transpile the JSX source:

`$ babel source.js -d dist/`

Given the following input (I just added some necessary boilerplate)
```js
sap.ui.define(
  ["sap/ui/core/Controller", "sap/m/Label", "sap/m/Input", "sap/ui/layout/form/SimpleForm"],
  function(Controller, Label, Input, SimpleForm) {
    "use strict"

    return Controller.extend("example.Controller", {
      myControllerFunction: function() {
        function onChange(oEvent) {
          this.setValue(oEvent.getParameter("value"));
        }
        return (
          <SimpleForm editable="true">
            <Label text={this.config.displayName} />
            <Input value={this.getValue()} type="Number" change={onChange.bind(this)} />
          </SimpleForm>
        )
      }
    });
  }, 
  true
);
```

we receive the following output

```js
sap.ui.define([
  "sap/ui/core/Controller", "sap/m/Label", 
  "sap/m/Input", "sap/ui/layout/form/SimpleForm"
  ], 
  function (Controller, Label, Input, SimpleForm) {
  "use strict";

  return Controller.extend("example.Controller", {
    myControllerFunction: function () {
      function onChange(oEvent) {
        this.setValue(oEvent.getParameter("value"));
      }
      return {
        elementName: "SimpleForm",
        attributes: {
          editable: "true"
        },
        children: [{
          elementName: "Label",
          attributes: {
            text: this.config.displayName
          },
          children: null
        }, {
          elementName: "Input",
          attributes: {
            value: this.getValue(),
            type: "Number",
            change: onChange.bind(this)
          },
          children: null
        }]
      };
    }
  });
}, true);
```

Wohoo. That looks scary. But that’s okay, its compiled code. Have you ever looked at what gcc produces?

The real problem is, this does not work. It does something, but not what we intended. It just translates the XML to JS objects representing the same values. We somehow need to create UI5 controls from these objects.

There are two options to do that:

a) generate code that is as close as possible to the first example. Call constructors directly, passing the contents of attributes and children to the constructor.

b) add some code that receives these objects at run time and creates the controls for us.

While (a) would produce a more readable output and the best performance, there is a problem which makes (b) the better option for me.

To understand the problem we need to introduce the concept of Aggregations. In UI5 an aggregation is a property of a component that can contain a list of other components. A List contains items, a Panel or a SimpleForm can contain content, a Table contains columns. A component may have multiple aggregations and may (or may not) have a single default aggregation. For List its items, for Panel its content. A Table contains columns and items, the default is items.

So if we decide to transform JSX into normal UI5 JS code, we would have to decide which aggregation to use for the children. But since it depends on the implementation, its impossible (or at least really really hard). So we could just define that children always go into, for example, content and you would need to use the attribute notation if you want another aggregation but that would feel awkward and non-intuitive.

On the other hand, if we decide to do this at run time, we can ask the class for its default aggregation and use that one.

The code for that could look somewhat like this:
```js
function createElement(definition) {
  var attributes = definition.attributes

  if (typeof definition.elementName !== 'string') {
    var Class = definition.elementName

    if (Class.getMetadata) {
      var oMetadata = Class.getMetadata()
      var sDefaultAggregation = oMetadata._sDefaultAggregation
      if (typeof sDefaultAggregation === 'string') {
        if (attributes[sDefaultAggregation] !== undefined 
        	&& definition.children.length === 1) {
          attributes[sDefaultAggregation].template = definition.children[0]
        } else {
          attributes[sDefaultAggregation] = definition.children
        }
      }
    } // else create a sap.ui.core.HTML?

    return new Class(attributes);
}
```

Its pretty straight forward: we expect the object (called definition here) to contain a set of attributes, an elementName that is a function or a string, and optionally a list of children. If the elementName is a function, we expect it to be a UI5 class and check its metadata for the default aggregation.

The transform-jsx plugin allows us to specify a module, that gets pulled in the namespaces when required, and returns this function OR to specify a function that should be used to construct elements. Since babel currently cannot deal with UI5s variance of AMD, we need to handle loading of this code by ourselves.

Lets create a createElement.js file next to our source.js and import it:
```js
sap.define([], 
  function() { 
    "use strict";
    
    return function createElement(definition) {
      var attributes = definition.attributes
  
      if (typeof definition.elementName !== 'string') {
        var Class = definition.elementName
  
        if (Class.getMetadata) {
          var oMetadata = Class.getMetadata()
          var sDefaultAggregation = oMetadata._sDefaultAggregation
          if (typeof sDefaultAggregation === 'string') {
            if (attributes[sDefaultAggregation] !== undefined 
          	  && definition.children.length === 1) {
              attributes[sDefaultAggregation].template = definition.children[0]
            } else {
              attributes[sDefaultAggregation] = definition.children
            }
          }
        } // else create a sap.ui.core.HTML?
  
        return new Class(attributes);
      }
    }
  },
  true
);
```

In the *source.js*, we just modify the first three lines:

```js
sap.ui.define(
  [
    "sap/ui/core/Controller", "sap/m/Label", "sap/m/Input", 
    "sap/ui/layout/form/SimpleForm", "./createElement.js"
  ], 
  function (Controller, Label, Input, SimpleForm, createElement) {
```
Now that `createElement` is known in our scope, we can change the babelrc to tell transform-jsx to use it.

```json
{ 
  "plugins": [  
    [ "transform-jsx", { function: "createElement", useVariables: true} ] 
  ] 
}
```

The `useVariables` part makes transform-jsx use element names that start with an upper case letter and are valid identifiers as identifiers. That way, when translating `<List />` we get `{ elementName: List, ... }` instead of `{ elementName: "List", ... }` which means that the actual constructor for a List will be contained in the object definition, not just the name.

Now *source.js* compiles to something like this:
```js
sap.ui.define(
  [
    "sap/ui/core/Controller", "sap/m/Label", "sap/m/Input", 
    "sap/ui/layout/form/SimpleForm", "./createElement.js"
  ], 
  function (Controller, Label, Input, SimpleForm, createElement) {
  "use strict";

  return Controller.extend("example.Controller", {
    myControllerFunction: function () {
      function onChange(oEvent) {
        this.setValue(oEvent.getParameter("value"));
      }
      return createElement({
        elementName: SimpleForm,
        attributes: {
          editable: "true"
        },
        children: [createElement({
          elementName: Label,
          attributes: {
            text: this.config.displayName
          },
          children: null
        }), createElement({
          elementName: Input,
          attributes: {
            value: this.getValue(),
            type: "Number",
            change: onChange.bind(this)
          },
          children: null
        })]
      });
    }
  });
}, true);
```

And this, indeed could work.

I created a tiny repository on GitHub which includes a working example of this and some additional features (did someone say ES6 arrow functions?).

[GitHub: themasch/openui5-jsx-example](https://github.com/themasch/openui5-jsx-example)

So how useful is it?

This depends on your workflow. If you do all you UI5 development in the SAP WebIDE you just can’t do it. If you just don’t want to deal with the change-compile-test loop at all you might don’t consider the benefits worth the additional steps. Although this could be reduced with something like `babel --watch`.

For my work it is of great use for projects that do not depend on the WebIDE. Not just the JSX parts, but also the whole glory of ES6 are worth an extra step in any case.

Next step: virtual-dom for the RenderManager? ;)