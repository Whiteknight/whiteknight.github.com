For anybody who hasn't heard of it, take some time out of your day today to
research Node.js. Node.js is a JavaScript engine and a special-purpose runtime
specifically designed for writing server-side web applications in JavaScript.
Of course, it has all the capabilities you would want for running normal
commandline programs written in JavaScript as well, but it is exploding onto
the scene in the world of server applications.

And why wouldn't it? Web developers have been using JavaScript on the client
side for years. Despite attempted intrusions by other languages such as
Microsoft's VBScript or a handful of other failed attempts, JavaScript is the
*de facto* and the only real language for bringing portable logic to the
client side of the web equation. Flash is nice but hardly ubiquitous. Java
applets or whatever they are called, and Microsoft's Silverlight are in the
same boat. No matter how nice any of these technologies are, we all know they
won't stand the test of time. Combine quick JavaScript interpreters, CSS3,
HTML5, and the constant forward march of progress, and all other web
development technologies will eventually be left behind. In my mind it's not
a question of "if", it's a question of *when*.

And this is what makes Node.js so attactive. Suddenly we can tell our web
developers that they can write JavaScript on the server just as easily and
pervasively as they can write it on the client. And on top of that, you can
bring to bear all the wild performance improvements that JS engines have seen
in the past. Name me one other programming language or language runtime
technology that has been pushed so aggressively to the bleeding edge of
performance by so many big players. Microsoft, Apple, Google, Mozilla, and
all sorts of teams researching for them are all pushing JavaScript performance
forward, and the competition is driving innovation much more than hindering
it. I'm not in the least bit ashamed to say that Parrot will never provide
a JavaScript environment with as much performance as these players will. This
is partially because there is no way that we can afford to specialize so
heavily as a single-language executor can afford to do, and there's no way
we can apply as many programmer resources to the problem of JavaScript alone
as these players can do.

That's not to say that Parrot shouldn't try to provide a working, compliant,
and pleasant environment for coding JavaScript in. If anything, the rise of
JavaScript is proof that we need to pursue this path as far as we possibly
can. People will be writing lots of software in JavaScript, and I suspect they
will want to be using JavaScript libraries from their Python programs, and
using Ruby libraries from their Node programs, and wanting to combine existing
PHP web server infrastructure, or Ruby on Rails applications, with their new
server-side JavaScript magics. This is all not to mention that Parrot provides
a lot more features than JavaScript needs or even wants, which makes it a very
interesting testbed for quickly and easily exploring extensions or
modifications to the language, which might not be as easy or even possible on
some of the heavily optimized alternatives.
