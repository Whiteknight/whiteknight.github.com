---
layout: post
categories: [Parrot, Release, 4.0]
title: Parrot 4.0.0 "Hyperstasis" Released!
---

> At one extreme, it is possible to approach the subject on a high mathematical
> epsilon-delta level, which generally results in many undergraduate students not
> knowing what's going on. At the other extreme, it is possible to wave away all
> the subtleties until neither the student nor the teacher knows what's going on.
>
> -Stanley J. Farlow, Preface to Partial Differential Equations for Scientists and
> Engineers


On behalf of the Parrot team, I'm proud to announce Parrot 4.0.0, also known
as "Hyperstasis".  [Parrot](http://parrot.org/) is a virtual machine aimed
at running all dynamic languages.

Parrot 4.0.0 is available on [Parrot's FTP site]
(ftp://ftp.parrot.org/pub/parrot/releases/stable/4.0.0/), or by following the
download instructions at [http://parrot.org/download](http://parrot.org/download).
For those who would like to develop on Parrot, or help develop Parrot itself, we
recommend using Git to retrieve the source code to get the latest and best
Parrot code.

    Parrot 4.0.0 News:
        - Core
            + Several cleanups to the interp subsystem API
            + Cleanups and documentation additions for green threads and timers
            + Iterator PMC and family now implement the "iterator" role
            + A bug in Parrot_ext_try was fixed where it was not popping a context correctly
        - Documentation
            + Docs for all versions of Parrot ever released are now available
              at http://parrot.github.com
        - Tests
            + Timer PMC tests were converted from PASM to PIR


The SHA256 message digests for the downloadable tarballs are:

    a1e0bc3de509b247b2cea4863cc202cdceeaa329729416115d3c20a162a0dd88 parrot-4.0.0.tar.bz2
    a63d45f50f7dd8ba76395cd2af14108412398ac24b8d827db369221cdb37fada parrot-4.0.0.tar.gz

Many thanks to all our contributors for making this possible, and our sponsors
for supporting this project.  Our next scheduled release is 21 February 2012.

Enjoy!
