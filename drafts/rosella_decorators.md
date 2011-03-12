In the world of staticly typed languages, like C#, the Decorator pattern is
pretty common. Decorators are used to add features to objects of a type
without needing to subclass. The canonical example is in a GUI widget toolset,
where a "Scrollable" decorator to add scrollbars to a widget is a decorator
which can be applied to a text box, an image, whatever. Using decorators we
don't need to subclass TextBox to TextBoxScrollable and Image to
ImageScrollable, we just create one ScrollableDecorator class and follow the
Decorator pattern.

In C#, let's say we have some of the following types:

{% highlight csharp %}
class Widget {
    public Size GetSize() { ... }
    public void Draw() { ... }
    public void AddChildren(Widget[] children) { ... }
}

class TextBox : Widget {
    public Size GetSize() { ... }
    public void Draw() { ... }
    public void AddChildren(Widget[] children) { ... }
}

class ScrollableDecorator : Widget {
    private Widget decorated;
    public ScrollableDecorator(Widget decorated)
    {
        this.decoratored = decorated;
    }
    public Size GetSize() { return this.decorated.GetSize() }
    public void Draw()
    {
        this.DrawScrollbars();
        this.decorated.Draw();
    }
    public void AddChildren(Widget[] children)
    {
        this.decorated.AddChildren(children);
    }
}
{% endhighlight %}

This example is kind of corney, but it gives the general idea of a decorator.
The decorator implements the little bit of logic that it needs, and passes
through all the rest of the methods to the decorated object. So far as the
code cares, ScrollableDecorator is just another widget and can be
transparently treated like any other widget.

What I've never liked about the decorator class is the fact that we need to
write simple pass-through methods for every method in the class or the
interface. That's a huge waste, and for even moderately-sized interfaces we
can end up with large swaths of boiler-plate Chain-of-Responsibility call
thunks.

In Parrot I think we can do better. In fact, with the new Rosella Proxy
library I *know* we can. With the basic proxy library, I can do this in
NQP with Rosella:

    INIT { pir::load_bytecode("rosella/proxy.pbc"); }

    my $f := Rosella::build(Rosella::Proxy::Factory, 'String', [
        Rosella::build(Rosella::Proxy::Builder::Passthrough),
        Rosella::build(Rosella::Proxy::Builder::AttributeIntercept)
    ]);

    my $target := pir::box__PS("Hello world!");
    my $null := pir::null__P();
    my $p := $f.create($null, $target);
    pir::say($p);           # "Hello world!"
    $p.replace("world", "andrew");
    pir::say($p);           # "Hello andrew!"

In this example above, I'm creating a string called `$target`, and then I
create a proxy object `$p` to proxy that string. The proxy is set up with
two builders: Passthrough, which causes all vtable accesses on the proxy to
automatically fallback to the target, and AttributeIntercept which allows us
to intercept get/set attribute accesses. The reason why we need
AttributeIntercept in the example above is because of an oddity in Parrots
current object metamodel that stores an instance of a low-level PMC type in
the "proxy" attribute of a subclass. It's a stupid system, but it hasn't
caused a lot of pain so far and a workaround is easy enough to come up with.

In the example above the variable `$p` is a proxy object from an anonymous
proxy class, but it looks and acts exactly like a String PMC. This is the
first step in making transparent decorator objects. This is almost exactly
what the Decorator library does. Here's an example of the Decorator library
in action:

    INIT { pir::load_bytecode("rosella/decorate.pbc"); }

    my %attr := {};
    my %methods := {};
    %methods{"test"} := method() { pir::say("In Decorator.Test!"); }

    my $f := Rosella::build(Rosella::Decorator::Factory, %methods, %attr);
    my $target := pir::box__PS("Hello world!");
    my $null := pir::null__P();
    my $p := $f.create($null, $target);
    pir::say($p);           # "Hello world!"
    $p.replace("world", "andrew");
    pir::say($p);           # "Hello andrew!"
    $p.test();              # "In Decorator.Test!"

In this example I'm using `Rosella::Decorator::Factory`, instead of
`Rosella::Proxy::Factory` like I did in the previous example. The decorator
library is a wrapper around the proxy library which hides a little bit of
messy logic. The end result is quite cool: I can create transparent decorator
objects given a target type, a hash of named decoration methods and a hash of
named decoration attributes. With this tool, I can add arbitrary methods
and attributes to objects of any type, on a per-instance basis, without
needing to explicitly subclass and without violating encapsulation. It's the
decorator pattern without all the messy boilerplate code required by static
typed languages.

Thinking about decorators, I was using proxies to add new functionality to
existing objects. However, there are times when we might want to *remove*
functionality from them instead. I turned my attention to another related
pattern, the Immutable pattern.

In the Immutable pattern, the object is not modifiable by user code after it
has been instantiated. The object may be modifiable internally, but externally
there is no interface for modifying the internal data. Parrot's string
primitives are a very relevant example of an immutable data type. I decided
that I wanted to create a proxy type to turn any arbitrary PMC into an
immutable PMC. Here's how to do that with Rosella today:

    INIT { pir::load_bytecode("rosella/proxy.pbc"); }

    my $f := Rosella::build(Rosella::Proxy::Factory, 'String', [
        Rosella::build(Rosella::Proxy::Builder::Passthrough),
        Rosella::build(Rosella::Proxy::Builder::AttributeIntercept),
        Rosella::build(Rosella::Proxy::Builder::Immutable)
    ]);

    my $target := pir::box__PS("Hello world!");
    my $null := pir::null__P();
    my $p := $f.create($null, $target);
    pir::say($p);           # "Hello world!"
    $p.replace("world", "andrew");
    pir::say($p);           # Exception!

If you run the example above, it will print out "Hello world!", and then when
we call the `replace` method (which is a built-in method defined on the
String PMC) it throws an exception. That's not too shabby, considering I don't
have to modify any code inside Parrot. I can create any object, wrap it up
in an immutability proxy, and that proxy transparently appears to be a
read-only version of the target PMC.

