I've never been too happy with the random number generation capabilities of
Parrot. Parrot essentially provides a thin wrapper around the system `rand` and
`srand` capabilities, with an unimaginative interface. This is a fine system for
most purposes, but sometimes you need something a little bit different, or want
to substitute your own version without too much hassle.

One of the tasks I submitted to GCI was asking for an implementation of the
[Mersenne Twister][] algorithm, a good and well-known algorithm to generate
pseudo-random numbers. GCI student Yuki'N, who was also a prolific contributor
last year for the program, submitted a fine implementation for Rosella to use.
Shortly thereafter I wrote up a quick [Box-Muller][] implementation to get
normally-distributed numbers too. Now, Rosella has a pretty decent start of a
random number library.

For an example, I wrote up a short histogram program to display the output. Here
are the histograms for the Mersenne Twister and the Box-Muller generators:

    Histogram of 500 uniformly-distributed floats:
    0: ##########################
    1: ####################
    2: ##########################
    3: ##########################
    4: #############################
    5: #########################
    6: ###########################
    7: ########################
    8: ####################
    9: #######################
    10: ############################
    11: ##############
    12: ###############################
    13: #########################
    14: ##############
    15: ##########################
    16: ##########################
    17: ##############
    18: ##############################
    19: #######################

    Histogram of 500 normal-distributed floats:
    0: #
    1: #
    2: ####
    3: ######
    4: ########
    5: ########################
    6: #######################
    7: #########################################################
    8: ##########################################################################
    9: ###################################################################
    10: ################################################################
    11: ####################################################
    12: #######################################################
    13: #########################
    14: #################
    15: ########
    16: ##########
    17: #
    18:
    19: ##

These are the two distributions that I wanted to have most, but they are
certainly not the only one ones available.

And random number generation is not the only feature that this library will
have, either. The Rosella Query library currently has an implementation of a
[Fisher-Yates Shuffle][fischer_yates] algorithm for shuffling an array. I think
I'm going to move that to the new Random library where it will be more visible
and re-usable. I have a few more small features in mind that I might like to add
to this library too, and since it is so straight-forward and algorithmic, I
suspect it will become stable pretty quickly.
