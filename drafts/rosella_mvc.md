http://amix.dk/blog/post/19615#Model-View-Controller-History-theory-and-usage

I refactored the Rosella TAP Harness library to use an MVC architecture, and
that got me to thinking: Maybe I should provide an MVC framework library that
not only can be used by my TAP harness, but also would be easily adoptable by
other libraries and projects. After all, in many cases MVC is a pretty great
architecture to use, and supporting best practices in exactly the kind of
thing that Rosella aims to do.

However, this really raises some interesting questions. What does an MVC
look like at the level of abstraction where Rosella lives? Rosella doesn't
presume the existance of any particular user-interface, which is a necessary
component of any "View". Actually that isn't completely true, some parts of
Rosella do print information to the console right now (this is more an issue
of me being too lazy to properly implement output delegation like I should).
Rosella doesn't want to provide an MVC library which is console-only, but
most other GUI frameworks haven't been ported to Parrot yet and it's hard to
pretend to target something which doesn't exist.

What Rosella wants to do is enable best practices and provide building blocks,
not necessarily provide the final solutions. To that end we have to start
thinking and try to answer these questions: What, if anything, does an MVC
library look like, at this level of abstraction? How do we make an MVC, or
something which is pre-MVC, which has no particular knowledge of any
particular user interface technology? What can Rosella provide to help make
MVC easier to use for other programs?

Let's break it down. M is for "Model", the "business object" which implements
the particular problem domain logic. Models are completely agnostic, there are
no similarities between models in different programs. It wouldn't make sense
for Rosella to try to provide any functionality for models, they are all
different, have no inherent commonalities, and don't necessarily need to
even contain references to the View or the Controller. In fact, it's probably
best if the model doesn't reference either of those things. So, Rosella really
can't provide any foundation for making models.

V is for "View", which is typically the interface which the user interacts
with. The View presents information to the users and typically also takes
input to pass along to the controller. Rosella can't presuppose anything about
the actual interface toolkit that the View uses. However, one thing Rosella
can help to inforce is the 1:1 linkage between a View and its associated
Controller. Using Controller Injection we can have a ViewFactory which, when
asked for a new instance of a View, automatically instantiates and associates
a suitable Controller as well. Anothing thing Rosella could do is enforce a
contract on View types to require them to have certain methods and attributes
for working with a Controller and maybe a Model (or even a ViewModel of
sorts).

C is for "Controller", the logic element which controls the View and helps to
make data from the Model presentable on it. Similarly to the other elements,
the Controller is pretty nebulously defined and Rosella cannot make too many
assumptions about how it is used. Using a technique called View Injection,
Rosella could provide a ControllerFactory which automatically instantiates
and associates a suitable View when a new Controller is created. Rosella could
also enforce a basic contract on Controller types to ensure that they
satisfied a basic interface and always have references to a Model and a View.

Besides the basic contracts on View and Controller, Rosella can't do a whole
heck of lot of groundwork for directly supporting MVC. However, that doesn't
mean that Rosella can't do anything at all. As I mentioned above, ViewFactory
and ControllerFactory types could help to simplify the creation of
View/Controller pairs, help to keep them synchronized and initialized without
timing issues, and also help to associate them both with the same Model.

Here's a basic Winxed code example showing how we would create both in a
factory-like setting:

   function get_controller(var controller_type, var view_type, var model) {
       using Rosella.build;
       var controller = build(controller_type);
       var view = build(view_type);
       controller.view = view;
       controller.model = model;
       view.controller = controller;
       view.model = model;
       controller.initialize();
       return controller;
   }

This is trivial, of course, but it does show a basic sequence that would have
to be repeated many times by a programmer trying to use MVC without some kind
of preexisting framework.

In addition to basic factory types for creating Views and Controllers, Rosella
could also implement some basic infrastructure in the form of Modules. A
Module in this case would be a named collection of related Views and
Controllers. For instance, on a webpage we might have a module called
"Homepage", which contained classes HomepageView and HomepageController. If
we associate those things together in a Module, make the module available
through a type resolution container, and handle accesses through that module
instead of directly through the Controller and View, we do gain some benefits.

One thing that's worth mentioning is that MVC is really more about separation
of concerns than about specifying any particular way those components
communicate with each other. If we employed a Mediator-style approach, the
Controller becomes the single point of communication between the View and the
Model. That is, the Model does not directly talk to the View, and the View
does not directly feed back to the Model. All communications travel through
the controller, which in some sense is like a network router. This is the
style of MVC done in Cocoa, for instance. Some people might suggest that the
Mediator-based approach to MVC is really called something else like "MVP"
("Model-View-Presenter") instead. In this case, the Presenter is just an
implementation of a Controller which is also a Mediator. Other people may
suggest different or similar-but-nuanced definitions of "Presenter". I won't
concern myself with this difference here too much. If the definition of what
MVC is is kept sufficiently vague, we can just say MVP and MVVM (and any other
variant) are just different flavors of MVC. In that case, if Rosella provides
a library which is sufficiently flexibile, we can support all these things
with relative ease. For the purposes of this post, I will refer to all related
schemes as "MVC", and assume the reader can pick out cases that could possibly
be named something more specialized. But, I digress.

Another model for MVC, which is slightly more complex but also more
"traditional" is the system used by the original Smalltalk implementations.
The controller is primarily concerned with talking to the View, to keep track
of state and providing logic handlers for user actions. The View can directly
ask the Model for data, and the Model can respond by passing data (and data
update notifications) to the View.

In some frameworks, notably ASP.NET MVC, the system creates a Model, passes
it to the Controller, and the Controller instantiates and uses that Model to
create and return a View as a response. The View uses the Model as its "Data
Context", and so can query values directly from it like in the Smalltalk
implementation.

Ruby on Rails is tinted by the separation between client and server. This
creates a huge mandatory separation between the Model (the "live" version of
which undoubtably lives on the server) and the View (which undoubtably lives
on the client). Changes to the Model does not cause a change to the View
without a browser refresh. Changes to the View would get communicated back to
the Model (via the Controller) only during an HTTP request or through an
external mechanism like AJAX.

I could post a few more examples (and, display my fundamental misunderstanding
of them as well) but I won't. I think this is enough to start brainstorming
possible ideas for Rosella. I really like the simplicity of the system used
by Cocoa and Rails. Keeping the Model and the View separate really does help
conceptually, and that seems like the most reasonable architecture for
Rosella to implement and even enforce. I don't necessarily like the idea of
"enforcement". Expect Rosella's default implementation to have certain rules
but with the ability for the user to override them if necessary.

Let's say I have a Mediator-based MVC system. Let's look at what the flow
would be to create and control such a system.

To start out, somewhere along the line I would define a Module. The Module
would be a named mapping between related View and Controller types. When I say
"Give me a 'Foo'", the Module would automatically create and associate a
FooController and a FooView, and return one or the other (or both) to me.
If I say "Give me a 'Foo' for this Model", the Module would be able to use
rules to inspect the Model and return the most appropriate View and Controller
to use. In the context of a web application, if I passed in a NormalUserData
object, I might get back a normal FooView and FooController. If, however, I
passed in an AdminUserData Model object, the Module might respond to me with
a FooAdminView and a FooAdminController.

It's worth mentioning here that if we descended Module from the Rosella
dependency injection Container class, we could optionally declare any
View/Controller pair to be either View-injected or Controller-injected. The
user could specify the dependency relation at the time the pair were
registered with the Module. It's easy enough to give the user both options
if we're using Container to implement it.

Controller would define a handful of standard methods for sending an update
notification to the View, sending an update with new data to the Model,
receiving user actions from the View and being able to route those requests
to user-specified handlers, etc. A View would have a standard `update` method,
and Models would be free-form. Because Models are free-form and have no
expectations placed on them, I may want to provide a ViewModel or "Data
Context" object which can be used internally by the library to pass Model
data around in a consistent and safe way. For instance, we may want the
ability to guarantee that an untrusted View cannot make or cause to be made
modifications to a trusted Model. A protected ViewModel and a Controller which
does not provide any mechanism for modifying the Model (at least not in a way
that would surive massive API change on the part of the Model) would be very
helpful in many cases.

There's also the idea of a dumb or passive View. That is, the View has display
information but does not implement *any* logic, except the bare minimum needed
to redirect accesses to the Controller. In this scheme, the Controller
implements all logic for the View, and the View is a "slave" of the Controller
"master". This type of scheme is very important for testability, because the
Controller becomes completely testable without a View, and the Controller
also becomes more independent of any View. Testing and testability is very
important for Rosella, but this also isn't the type of requirement that can be
enforced. In a web application where the View is just a template of HTML, this
is easy. The HTML syntax would only admit basic tags and content, and would
have some hooks for inserting data from the Model (or ViewModel/DataContext).

Where the difficulty in this scheme becomes apparent is when we try to change
to a different underlying View technology. Imagine if we have a model and a
Controller with a View that provides HTML for a webpage. If we want to reuse
the logic core of our application in a desktop program with Qt widgets,
suddenly our entire View infrastructure has to get tossed out the window and
we can't make guarantees that our View is going to be dumb.

If Rosella wants to specialize on something like HTML webserving, we could
make a pretty robust infrastructure for templating and serving text using
dumb Views. If Rosella wants to be more agnostic, we can't make any of these
kinds of guarantees, and can't assume that the View is going to be a simple
text template. The View can really be any mechanism for presenting data to the
outside world, including a mechanism that is only read by computers (web
service, library API, etc).

So there is my long-winded brain-dump about MVC on Rosella. The fact is that
I really do want to create some kind of MVC library for Rosella, but I am
trying to find a good design for it that is both flexible enough to support
the widest variety of schemes, but isn't so flexible that it's no more useful
than a blank file and your favorite text editor. If I can find a solution
which both provides powerful defaults and admits useful modifications, I will
certainly start putting finger to keyboard and develop it.

