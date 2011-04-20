---
layout: post
categories: [Parrot, Rosella]
title: Rosella Event Library
---

In previous posts I mentioned that the first release of Rosella would contain
8 individual libraries: Core, Action, Container, Proxy, Test, Harness,
MockObject and Winxed. A few days ago I made the decision to add in a ninth
library as well: Event. This post is going to be a short introduction to the
Rosella Event library, talk about what it does, why it is useful, and maybe
show some short examples of its use.

The Rosella Event library is basically an implementation of the
[publish/subscribe][] pattern. Publish/subscribe (or "pub/sub") is a pattern
of *decoupling* software components in large complex systems. Decoupling is
where we break hard links and dependencies between different types and
components to allow things to develop independently. Decoupling is typically
considered to have a beneficial effect in terms of improving long-term
software maintainability.

[publish/subscribe]:

Event works by exposing a `Rosella.Event` object. This Event object maintains
a list of subscribers which can be notified when the Event is raised. The big
benefit here is anonymity: The component raising the Event does not know
about who will be receiving it. The components receiving the Event do not know
where it was sent from. The code raising the Event doesn't need to know any of
the details about who is receiving the event, and doesn't even need to know
if anybody is listening at all!

Keeping track of all events in the `Rosella.EventManager` object. EventManager
keeps a list of events by name, and helps to manage all the necessary details.
Where Event implements much of the publish/subscribe logic, EventManager
provides the friendly and usable interface. You can use Event directly if you
want to, but EventManager is more likely the way to go.

Let's look at an example of an IRC client with plug-ins. We have a Listener
object reading in data from the server, and passing those commands to various
other components and widgets. Here's a partial implementation of an
IRCListener class which takes in command lines (probably from a socket reader
routine), parses them, and passes along the displayable data to the display
portion of the UI:

    class IRCListener {
        var text_display;

        method process_next_command(var cmd) {
            string from_username = get_username(cmd);
            string text = get_text(cmd);
            self.text_display.add_text(from_username, text);
        }
    }

This is an extremely simplistic implementation, of course. Now, we want to add
in a new plugin which reads in the text from the server to do some additional
analysis on it. For instance, a plugin which makes a little "bing" noise when
your nickname is mentioned, or which automatically replies with an away
message, or whatever. Let's add logic to our routine for sending the text
data to the plugin as well as the UI:

    class IRCListener {
        var text_display;
        var plugin;

        method process_next_command(var cmd) {
            string from_username = get_username(cmd);
            string text = get_text(cmd);
            self.text_display.add_text(from_username, text);
            self.pluggin.recv_text(text);
        }
    }

This is all well and good at first, but what do we do now if we want to add
a dozen plugins? A hundred of them? What if we want the list of operating
plugins to be variable? Well, we could store plugins in an array or a list of
some sort, and loop. But then what if we wanted to have multiple different
UIs, such as an IRC client that had multiple tabs, and the text would get
routed between them?

My point with all this is that the IRCListener class should really only be
worried with one flow of data: Receiving data from the sockets, breaking that
input data into the individual fields, and passing those fields along to
the parts of the program that use them. If we're adding in all sorts of logic
to manage lists of UI and plugin doodads and components, we've really violated
that single responsibility. Let somebody else manage the lists and keep track
of the recipients!

Now, let's rewrite this class to make use of the Rosella Event library:

    class IRCListener {
        var event_manager;

        method process_next_command(var cmd) {
            string from_username = get_username(cmd);
            string text = get_text(cmd);
            self.event_manager.raise_event("text_recv", from_username, text);
        }
    }

Problem solved. Now IRCListener doesn't have to give a damn about who is on
the other end of the line. We don't need to manage the list of recipients. We
also don't need the IRCListener class to even be available at the time
recipients are being assigned. All we need to do is register recipients with
the event manager objects, then publish events to it whenever we want.

On the other end of the equation, we have subscribers. In the general sense,
a subscriber is a tuple of an object and a method. Here's an example of a chat
display which outputs data to the console:

    class IRCTextDisplay {
        function BUILD(var event_manager) {
            event_manager.subscribe_object(self, "text_recv");
        }

        function text_recv(string username, string text) {
            say(
                sprintf("<%s> %s", [username, text])
            );
        }
    }

With this code, every time the "text_recv" event is raised, the
`.text_recv()` method on this instance of IRCTextDisplay will be invoked to
print the details out to the console. Keep in mind, we can have as many
objects subscribed to this event as we want, without needing to modify any
additional code, without needing to directly reference IRCListener or any of
its current instances (without even needing to have an instance of IRCListener
available at all), and without having to duplicate subscriber list logic in
every part of the system that might be publishing text_recv or similar events.

That's the quick overview of the Rosella Event library, the 9th library in
the project to be listed as "stable" and intended to be included in the
upcoming release. It is a very simple but powerful library with only two
types of objects and a scant handful of important methods. The benefits in
improving program maintainability can be quite profound, however, and should
not be disregarded because of how easy this library is to use.
