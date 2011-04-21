Yesterday I put out Parrot 3.3 "Fire in the Sky". Besides the fact that it
happened much later in the day than we have become accustomed to in recent
months, it was a pretty uneventful release. Almost all releases are pretty
uneventful, one of the hallmarks of the regular monthly release system.

One thing I noticed again, which I notice every time I do a release, is that
the process is convoluted and needlessly complicated. Normally I don't put a
second thought towards it, once the release is out we can forget the pain and
continue on our merry way. However, I can't help but feel like eventually we
need to start making changes to make the release not only as uneventful and
painless as possible, but also to make them as easy and simple as possible.
The fewer steps we have, the faster a release can be cut and the less
expertise is required to cut one.

The release is composed from five different general phases: Updating the
release metadata, building and testing the release tarball, making the tarball
and documentation public, updating the website, and sending out all sorts of
announcements about the release. I'm going to describe these five phases here,
along with some ways that I think we should try to minimize and simplify them.

## Updating Release Metadata

The first part of the release is the easiest and most straight-forward. The
release manager needs to update several metadata files to include information
about the new release and the subsequent release. These files are VERSION,
ChangeLog, NEWS, MANIFEST.generated, release.json, release_manager_guide.pod
and parrothist.pod. This phase of the release is the most straight-forward as
I mentioned and is one of the most amenable to automation, though we haven't
yet put much effort into automations.

There is also some duplication of effort here. ChangeLog is pretty much a
worthless file now. Since the 0.4.14 release back in July 2007, the ChangeLog
file has contained exactly the same note for every single release: "See NEWS
for details". That's a complete waste of effort, and I suggest we remove
ChangeLog from the release procedure and maybe freeze or delete the file from
the repo entirely. dukeleto suggested that maybe some Linux distros like
Debian might require a ChangeLog file, to which I reply it's extremely easy
to generate this file automatically. A short string literal inside a short
for-loop would do the trick, and we could generate the file on demand instead
of having to update it every month with repetitive information is a waste.

The release_manager_guide.pod file, in addition to containing all the
instructions for cutting a release, also contains a schedule for upcoming
releases. parrothist.pod contains a list of previous releases. During each
release, the current release is removed from release_manager_guide.pod and
an entry is added to parrothist.pod. I suggest those two lists can be combined
into a single file.

release.json contains a lot of information including the name and date of the
current release, the data of the subsequent release, the date of the upcoming
"bug day" (which I'm not sure we've ever honored) and a little bit more
information. Right now we update all this information by hand each release,
but it should be extremely easy to automate it.

If we had a program that asked the user for the current release number and the
current release name, all these files I mentioned could be and should be
updated automatically.

## Building and Testing the Tarball

This is the longest part of the release, but it's very easy to do. In essence
you build and test Parrot, create the release tarball, unpack that tarball
somewhere else, and then build and test Parrot again. Running fulltest twice
takes the lions share of the time in a release, especially if the release
manager isn't running on a super-powerful computer.

Where we throw a monkey wrench into the system is that the release build needs
to be bootstrapped. So you need to have an old version of Parrot available to
update the core op library, then build parrot, test, make release, build and
test. That bootstrapping step is a little bit confusing, and does add a little
bit of complexity. Luckily, assuming all tests pass, this process is extremely
easy to automate. In fact, here's the general script to do it:

    perl Configure.pl ...
    make
    < update VERSION file to new version number >
    ./ops2c --core
    make realclean
    perl Configure.pl --test ...
    make world html
    make fulltest
    make release
    make release_check

It would be trivial to write a short program to update VERSION, which is
something from the previous phase which can all be easily automated. A single
release script could easily bring us this far with a single command and a
prompt for a release name. A decent perl hacker could put it together in less
than an hour. Where there are errors or test failures the release manager can
drop out of the script and do things manually, but when things go well (and
they usually do for the release) this level of automation would be an extreme
boon.

## Uploading the Tarball and Docs

The Open Source Lab at OSU hosts the Parrot web infrastructure. This includes
an FTP site for hosting the release tarball, the docs.parrot.org website for
hosting generated HTML documentation, the parrot.org website, and the
trac.parrot.org issue tracker software. This is the part of the release that
most often goes awry. We need to update the tarball on the FTP server which
requires one set of credentials, and then we need to update the docs on the
docs.parrot.org website which requires a second set of credentials. This is
kind of a hassle and until this release even *I* didn't have everything I
needed for a complete release. Combine that with the fact that I lost my last
SSH key when my computer crashed and I had to reformat it, and then had to
update that on the FTP server so I could log in again. Maybe other people
aren't as stupid and incompetent as I am with my SSH keys, buit can still be
a hassle if everything isn't working right (and there's no reason to check if
it is working right until the day of the release anyway).

This part of the release is a big hassle and is the hardest to automate.

## Updating the Website

The parrot.org website is a Drupal instance, which I can't say I really enjoy
too much. My personal opinions aside, we need to update the website to include
information about the release on the website. We need to write up a release
announcement which always seems to be harder than it needs to be (I'll discuss
this in the next section below). We then need to update a series of URL
redirects to point to the new release tarballs. This part of the release is
relatively short, but requires yet another set of credentials. The parrot.org
username isn't the same as your trac.parrot.org username, and not just any
user can publish to the front page or update the URL redirects. You need to
create the account then get somebody to grant you all the necessary
permissions to update it.

The drupal information is all stored in a database, and I can't think of any
real reason why we couldn't use a simple update script to update the entries.
This could be some kind of a script that operates as a cron job, or a script
that was manually triggered somehow. Either way, it doesn't need to be big and
fancy, just enough SQL to update two table entries.

In trac we have milestones set up for each release, and supposedly we should
be using those milestones to organize tickets. The problem is that we haven't
been using this feature of the software since we started redoin the way we
handle our roadmaps. For the release, the release manager is supposed to go
into trac and close out the corresponding milestone. However, when I cut the
3.3 release, the 3.0, 3.1, and 3.2 release milestones were still open in trac.
The previous three release managers hadn't bothered to close any of these
milestones, and none of them had any tickets associated with them. My
suggestion here is to do away with these milestones entirely, we shouldn't
need to update anything on trac for the release.

## Announce the Release

The final stage of the release. It doesn't require a heck of a lot of thought,
but it is tedious time-consuming, and open-ended. once the tarballs are on the
public FTP site and the URL redirects are set up on the website is to start
creating release announcements. The program `crow.pir` automatically generates
text-based and html-based release announcements, filling in some information
from NEWS and release.json from earlier in the process. Once the release
announcements are made, the release manager needs to proof-read them, add in
the sha256 checksums of the release tarballs, maybe add in a meaningful quote
of some sort, and then start sending them out.

One copy of the release announcement goes on the parrot.org website as I
mentioned above. Another copy gets mailed out to a variety of mailing lists.
More get sent out to various news-y websites or other interested recipients.

We also need to do some unrelated things, like updating the channel topic on
IRC to include the current version number, updating the Wikipedia entry for
Parrot to include the current release number and a link to the most recent
release announcement, and a few other things.
