---
layout: post
title: Adding i18n Support to Parrot
categories: [Parrot, ParrotProjectIdeas]
---

We can talk about all the programming languages that either comprise Parrot,
or run on top of it. But when we are talking about spoken languages, Parrot
is completely monolingual. English is the language of Parrot. All it's method
names are English. Type names are English. Error messages are English. This is
just fine if you live in an English-speaking country or if you know English as
a second language, but it leaves everybody else out.

Luckily, there are standards out there that software programs can follow, one
of which is [i18n][]. This post details the project of making Parrot i18n
compliant, and discusses one possible design for an internationalization
system that adds i18n support to Parrot.

[i18n]: http://en.wikipedia.org/wiki/Internationalization_and_localization

In MediaWiki, where I've done a little bit of development on extensions and
the like, i18n support is handled in a pretty straight-forward way. Every
user-facing text string is represented in the code by a slug, or an identifier.
Elsewhere, in another code file specifically for messaging (typically named
`*.i18n.php`), we define a hash table that maps slugs to actual user messages,
depending on language. Here's an example slug table from my [EmbedVideo][]
extension:

[EmbedVideo]: https://github.com/Whiteknight/mediawiki-embedvideo

{% highlight php %}

$messages['en'] = array(
    'embedvideo-missing-params' =>
        'EmbedVideo is missing a required parameter.',
    'embedvideo-bad-params' =>
        'EmbedVideo received a bad parameter.',
    'embedvideo-unparsable-param-string' =>
        'EmbedVideo received the unparsable parameter string "<tt>$1</tt>".',
    'embedvideo-unrecognized-service' =>
        'EmbedVideo does not recognize the video service "<tt>$1</tt>".',
    'embedvideo-bad-id' =>
        'EmbedVideo received the bad id "$1" for the service "$2".',
    'embedvideo-illegal-width' =>
        'EmbedVideo received the illegal width parameter "$1".',
);

{% endhighlight %}

Elsewhere in that same file we can define the same messages for German
(`$messages['de']`), Spanish (`$messages['es']`), or whatever other language
we want. Finding whatever message I want is as easy as a lookup with the
current user's language preferences:

{% highlight php %}
$msg = $messages[$language][$slug];
{% endhighlight %}

This is the kind of system that I would like to duplicate for Parrot. Parrot
already has a hash type that we can use to map string slugs to complete
messages. We also have the ability to freeze those hashes to bytecode for
later retrieval.

In a nutshell, here is my idea for adding i18n compatibility to Parrot:

1. Create a series of hashes in individual files somewhere easy to locate,
   such as `runtime/i18n/*.pir`
2. During the build, we find all such files and serialize them to bytecode.
3. At runtime, we select and load the correct hash based on current settings.
   The default can be set at configure time, but we can have a command-line
   argument for selecting something else at runtime
4. Change most instances of the `CONST_STRING` macro to instead take a slug
   value and look that up in the current messages hash

We do run into a problem during that build in that we do not have access to
the error messages during the build while we are compiling the files of error
messages (or, anything prior to that). This is not an insurmountable problem,
but it could lead to some very confusing errors if something goes wrong at
this critical point.

One thing we gain is the ability to override the text of any system message or
error message at runtime. All we need to do is ask the interpreter for its
current messages hash, and either overwrite existing entries or add in new
entries. Using the second mechanism, any program that runs on top of Parrot
can immediately become i18n compliant: Add in your own slugs at runtime, and
then specify a message slug in exception objects that you throw. If there is
a translation library loaded, the slug will be looked up in that library
instead of default english. Everybody wins!
