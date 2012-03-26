---
layout: post
categories: [Parrot, Rosella, Net]
title: Rosella Net
---

I've started work on a new library for Rosella: Net. Net will be a networking
library for doing things like HTTP requests and related operations. For a
first draft I'm modelling it closely on the LWP::, Http::, and related modules
in Parrot's standard library. Many of these have been lovingly crafted and
maintained by Francois Perrad and others over the years. Basically I'm borrowing
the essential algorithms from the existing libraries, translating to Winxed
en passant, and using them as a basis for building a bigger library.

I'm not looking to replace the LWP or Http modules from Parrot. Those things
are required dependencies of **Distutils**, which is used by many projects for
good reason. Distutils is an amazing tool for quickly and easily setting up
build and test frameworks for Parrot-based projects. I wouldn't want to disrupt
that. In fact, I want to encourage it. Rosella uses Distutils for its build. PLA
uses it as well. It's a great tool and most projects on Parrot should consider
using it. What I do want to do is create an internet-access framework where
basic functionality is easy to get to and where new features can be easily
added. I also want to fill in what I see as being a pretty important
functionality gap in Rosella's collection of libraries.

The LWP and Http libraries in the Parrot standard runtime are loosly-based on
the Perl 5 modules of the same names. They started as something of a direct
translation of the necessary parts of the Perl5 libraries directly to PIR.
What I want to do is start with some of the ideas and algorithms from that
port, update the interfaces to make them more Rosella-esque, and then start
adding some of the features that the LWP and Http ports didn't include (and
maybe more beyond that).

Eventually I would like to add a nice wrapper interface around Socket and Select
PMCs the same way the FileSystem library adds nicer wrappers around FileHandle
and OS PMCs. I also want to add support for other protocols (FTP and IRC both
come to mind). SOAP, REST, and RPC calls could be interesting to add in the
future, though most of those would require a library for building and parsing
XML, which I don't have now and don't have any plans to add in the near future.
If I could add upload support for Smolder or other continuous integration
applications to my Harness library, that would be fun too.

As of today I can upload report archives to smolder which have been previously
assembled by distutils:

    var request = Rosella.Net.Http.create_request("http://smolder.parrot.org/app/projects/process_add_report/2");
    request.set_method("POST");
    request.add_header("Connection", "close");
    request.add_form_field("architecture", "x86-64");
    request.add_form_field("platform", "linux");
    request.add_form_field("revision", "0e75eb61d5b713b57b05e91a2d45251bb18d3e2e");
    request.add_form_field("tags", "test");
    request.add_form_field_filename("report_file", "/pla/report.tar.gz", "application/octet-stream");
    var response = Rosella.Net.Http.request(request);
    say(response.status_code());
    say(response.header().get_header_text());
    say(response.get_content());

This is a lower-level of interface than I expect most people to use when the
library is more mature. Eventually you'll create a UserAgent object which
provides nice pretty methods for making GET and POST requests. Like I said, this
is roughly based on Parrot's LWP library, which itself is based on Perl 5's
much-loved LWP module.

The library also supports local resource access with `file://` URIs:

    // Get a file:
    var request = Rosella.Net.Http.create_request("file:///home/andrew/data/foo.txt");
    var response = Rosella.Net.Http.request(request);
    say(response.get_content());

    // Create a new file:
    var request = Rosella.Net.Http.create_request("file:///home/andrew/data/bar.txt");
    request.set_method("PUT");
    request.set_content("Hello World!");
    var response = Rosella.Net.Http.request(request);
    say(response.status_code());

There isn't a lot to look at with this new library quite yet, but things are
moving along pretty quick. Now that I have some of the basic algorithms working,
I'll start rearranging the internals to more closely align with my long-term
goals. This has basically been a weekend project so far, and I need to get back
to my `remove_sub_flags` branch and a few other projects in Parrot core before I
take this too much further.


