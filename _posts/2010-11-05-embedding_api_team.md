---
layout: post
categories: [Parrot, ParrotProductManagement, Embedding]
title: Embedding API Team
---

I have mentioned on this blog previously that I have started a project to
overhaul Parrot's embedding API. Actually, it isn't a personal project so much
as I hope it is the first project for Parrot's Product Management team.
Several people have expressed some interest in participating in this work,
either in the role of a team member (getting their hands dirty in the code) or
being a "resource" (somebody who has used the embedding API and is
willing/able to answer questions or test patches). We need plenty of both
types of volunteers. I estimate that on the code side we will need at least
three people in order to write the code that needs to be written. In terms of
resource people, I don't think there is a fixed number. We just need all the
help we can possibly get, no matter how small the contribution of time or
energy can be made.

I say at least three developers for several reasons:

# Not everybody is super-familiar with all the embedding code we currently
have, nor the huge myriad of capabilities that Parrot should expose but
currently does not (or does, but in a lousy way).
# We don't just need help picking through the messiest parts of the code. We
would also like to increase the ever-important bus number for embedding tasks.
We aren't just looking for temporary help, we are looking to create new
experts.
# It's not just a series of wrapper functions to be written or rewriten over
existing mechanisms. Later stages of the API are going to necessitate some
non-trivial changes to the core behaviors of Parrot. We are *definitely* going
to need plenty of development and testing help when we get to those parts

This is an important project, and obviously I will slug through it myself if
nobody else volunteers. Likewise, I will be happy to organize a team of a
hundred people, if we were so lucky as to have that many people interested.
The more people we do have, the faster we can get the necessary changes in
place. But one way or another, it's going to happen.

I hope that some people are interested in helping. If that sounds like your
cup of tea, please let me know here on the blog, via email, or on IRC. If we
get a few interested developers together I think we can put together a great,
powerful, and elegant API, and I think we can do it in a relatively short
period of time.

Tomorrow I'm going to outline what the new Embedding API is going to look
like. Nothing is set in stone yet, but I have a few ideas laid out that should
serve as a good starting point.
