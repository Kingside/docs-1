---
layout: default
title: Toolkit kernel
---

The Toolkit _kernel_ provides a thin layer of code that expresses the Toolkit opinion, and provides the sugar that all components use. The kernel code is provided by a file named `g-component.html`. A web component that depends on the Toolkit kernel is called a _g-component_.

## Component declaration ##
---
A web component declaration look like the following:

{% highlight html %}
<element name="tag-name">
    <template>
        <!-- shadow DOM here -->
    </template>
    <script>
        // lifecycle setup here
    </script>
</element>
{% endhighlight html %}

To have this component  this component `component()` lifecycle initializer to the component's `<script/>` block, as shown below:

{% highlight html %}
<element name="tag-name">
  <template>
    <!-- shadow DOM here -->
  </template>
  <script>
    this.component();
  </script>
</element>
{% endhighlight %}  

Note the following:

* The `component()` initializer is all that's required to  prepare this component to use Toolkit [conventions and features](#features). 
* The "name" attribute of `<element/>` determines the name of the custom element you use to instantiate the component (`<tag-name/>`, for example).

### Component initializer

You can supply a single object-valued argument to `component()` to define object prototypes, and perform other setup tasks. Most properties and methods defined in the argument to `component()` are used directly in the component's prototype. In the following example the component initializer defines a property `helloWorld` and a method `ready`. 

{% highlight javascript %}
this.component({
  message: "Hallo!",
  ready: function() {
    // component is ready now, we can do stuff
  }
});
{% endhighlight %}

A component's `ready` method, if it exists, is called when the component is ready for it to be used.

## Protected and public API surfaces

G-components have _public_ and _protected_ aspects. The public aspect represents the API that is visible and accessible directly from a component (element) instance. The protected aspect contains the API of which component users shouldn't need to be aware, such as event handlers or internal methods.

### Protected properties ###

Properties and methods supplied to the `component` initializer  become properties on the component's *protected* interface. In the following example, the initializer declares two protected properties: `clickColor` and a method `clickHandler`. 

{% highlight javascript %}
this.component({
  clickColor: 'orange',
  clickHandler: function() {
    this.node.style.backgroundColor = this.clickColor;
  }
});
{% endhighlight %}

Note that the `clickHandler` method runs in the protected scope; it uses `this` which resolves to the component's protected scope.

Typically, the consumer of a component only needs to deal with the public scope. The exception to this rule is when we need to operate on our node itself, we do this using the `this.node` reference, as shown in the example.

### Public properties ###

To make a property or method public, you "publish" it by placing it inside a `publish` object block:

{% highlight javascript %}
this.component({
  clickColor: 'orange',
  clickHandler: function() {
    this.node.style.backgroundColor = this.clickColor;
  },
  publish: {
    this.clickColor = 'red',

  }
});
{% endhighlight %}

To make a `blueify` method that is callable on the node (public), we _publish_ the method by placing it inside a `publish` object:

For example, let's say our design for the _my-tag_ element calls for a method that can turn the element text blue, so a user could do like so:

{% highlight javascript %}
myTag = document.querySelector("my-tag");
myTag.blueify();
{% endhighlight %}

The `blueify` method is a part of _my-tag_'s public API. It must be available to end-users on the element instance.

Now, imagine _my-tag_ is also supposed to turn orange if clicked. As part of our set-up, we attach a `click` listener to a method called `clickHandler` which turns the element orange.

In this case, `clickHandler` is not intended to be called by end-users, it's only there to service an event. In this case, `clickHandler` should be part of the protected API. Then the method is not visible on the element instance and calling

{% highlight javascript %}
myTag.clickHandler(); // error: undefined function 
{% endhighlight %}

Note the following:

1. There can be only one `publish` block per definition.
2. Published properties are actually stored on the **protected** prototype, then they are forwarded to the public prototype. In other words, `blueColor` is different from `clickColor` only because there is a public getter/setter pair to access it.
3. Published methods still operate in protected scope: the properties you can access via `this` are no different from methods declared outside the publish block. 

Bottom line: when building components use `this` naturally and declare properties and methods as you like. Then, if you happen to create API you want to make public, you just move it into the `publish` block.

## G-component features ##

This section describes the features of g-components. 



## Attributes and properties

Another Toolkit convention is that public properties are settable by attribute. For example, we could instantiate the `name-tag` component and set its public properties with the following:

{% highlight html %}
<name-tag myname="Steve" nameColor="tomato"></name-tag>
{% endhighlight %}

When `name-tag` is created, or when its attributes change value, those new values are reflected into their matching component properties. 

Remember that only _public_ properties are settable via attribute.

### Declaring public properties as attributes

You can also declare public properties directly on an `<element/>` tag using its `attributes` attribute. For example:

{% highlight html %}
<element name="name-tag" attributes="myName nameColor">
  <template>
    Hello! My name is <span style="color:{{nameColor}}">{{myName}}</span>
  </template>
  <script>
    this.component({
      nameColor: "orange"
    });
  </script>
</element>
{% endhighlight %}
 
In this case, `name-tag` declares two attributes (`myName` and `nameColor`). This is semantically the same as declaring them in a publish block. Note that properties declared as attributes default to 'undefined' unless defaults are set in the prototype, as done for `nameColor` in the above example.

#### Binding and custom attributes

Toolkit makes it possible to bind references between components via attributes. Generally, attributes are only string-valued, so the binding engine interprets reference bindings specially (in particular, interrogating an attribute for a bound reference property will just return the binding expression (the double-mustache).

Let's modify our `name-tag` to take a record instead of individual properties.

{% highlight html %}
<element name="name-tag" attributes="person">
  <template>
    Hello! My name is <span style="color:{{"{{person.nameColor"}}}}">{{"{{person.name"}}}}</span>
  </template>
  <script>
    this.component({
      person: {
        name: "Scott",
        nameColor: "orange"
      }
    });
  </script>
</element>
{% endhighlight %}

Now, imagine we make a new component called 'visitor-creds' that uses `name-tag`:

{% highlight html %}
<element name="visitor-creds">
  <template>
    <name-tag person="{{"{{person"}}}}"></name-tag>
  </template>
  <script>
    this.component({
      person: {
        name: "Scott",
        nameColor: "orange"
      }
    });
  </script>
</element>
{% endhighlight %}

When I make an instance of `visitor-creds`, its `person` object is bound to the `name-tag` instance, so now both components are using the same `person` object.



### Declarative event mapping

Toolkit supports declarative binding of events to methods in the component. The toolkit uses special <code>on-<em>event</em></code> syntax to trigger this binding behavior.

{% highlight html %}
<element name="g-cool" on-keypress="keypress">
  <template>
    <button on-click="buttonClick"></button>
  </template>
  <script>
    this.component({
      keypress: function(event) {
      },
      buttonClick: function(event) {
      }
    });
  </script>
</element>
{% endhighlight html %}

In this example, the `on-keypress` declaration maps the standard DOM `"keypress"` event to the `keypress` method in the component. Within the component template, the `on-click` declaration maps a custom `buttonClick` event to the `buttonClick` method in the component. This is achieved again without the need for any glue code. 

Some things to notice:

* The value of an event handler attribute is the string name of a method on the component. Unlike traditional syntax, you cannot put executable code in the attribute.
* The event handler is passed the following arguments:
  * `inEvent` is the [standard event object](http://www.w3.org/TR/DOM-Level-3-Events/#interface-Event).
  * `inDetail`: A convenience form of `inEvent.detail`.
  * `inSender`: A reference to the node that declared the handler. This is often different from `inEvent.target` (the lowest node that received the event) and `inEvent.currentTarget` (the component processing the event), so the Toolkit provides it directly.



### Manageable API Surface

G-component API is not public by default. Only API declared in the `publish` block is part of the public surface. This way, users of a component need not contend with internal properties or event handlers.

<element name="g-cool">
  <script>
    this.component({
      better: 'better',
      publish: {
        makeBetterBest: function() {
          this.better = 'best';
        }
      }
    });
  </script>
</element>

In this example, the `g-cool` component has a single public method, 
`makeBetterBest`. The property _better _is not visible on the node, but a user could call `node.makeBetterBest` to set the internal property to the string value 'best'. 

This property hiding is not for security (non-public properties are technically still available, they are just not surfaced). We hide properties only to simplify the API surface.

The non-public API of a g-component is inherited by subclasses (extensions). For this reason, we call the non-public API _protected _(and not _private_).

## Advanced sugaring

In addition to the above features, which are focused around making the core functionality of components simple and easy to use, the toolkit provides syntactical sugar that makes more advanced component features easy to create.

### Change watchers

All properties on a g-component can be watched for changes by implementing a <code><em>propertyName</em>Changed</code> handler. When the value of a watched property changes, the appropriate chnage handler is automatically invoked. 

{% highlight html %}
<element name="g-cool" attributes="better">
  <script>
    this.component({
      plain: '',
      publish: {
        best: ''
      },
      betterChanged: function(inOldValue) {
      },
      bestChanged: function(inOldValue) {
      }
    });
  </script>
</element>
{% endhighlight html %}

In this example, there are two watched properties, `better` and `best`. The `betterChanged` and `bestChanged` function will be called whenever `better` or `best` are modified, respectively. 

### Automatic node finding

Another useful feature of Toolkit is node reference marshalling. Every node in a component's shadow DOM that is tagged with an `id` attribute is automatically referenced in components `this.$` hash. 

For example, the following defines a g-component whose template contains an `<input>` element whose `id` attribute is `nameInput`. The component can refer to the that element with the expression `this.$.nameInput`.

{% highlight html %}
<element name="x-form">
  <template>
    <input id="nameInput">
  </template>
  <script>
    this.component({
      logNameValue: function() {
        console.log(this.$.nameInput.value);
      }
    });
  </script>
</element>
{% endhighlight %}

### Calling inherited methods with $super

A g-component can extend a parent g-component by calling the parent's inherited methods. 

{% highlight html %}
<element name="g-cooler" extends="g-cool">
  <script>
    this.component({
      moarBetter: function() {
        this.$super();
        this.better += 'even more.';
      }
    });
  </script>
</element>
{% endhighlight html %}

In this example, `this.$super` returns a reference to the parent, which is a `g-cool` g-component. In `g-cooler` the value of `better` contains the value of `g-cool` is, plus the string 'even more'.
