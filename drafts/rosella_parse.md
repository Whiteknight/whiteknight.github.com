As I mentioned a few days ago, I created simple XML and JSON parsing libraries
for Rosella. I tried using a different technique from what I have tried in the
past, which is using StringIterator and integer comparisons on characters
instead of substrings and creating string headers for each new character I
wanted to examine. This strategy is much more efficient in most cases, but isn't
always well supported by Parrot's current set of string-handling primitives.
There just isn't a lot of prior art.

While building these two libraries I had a lot of very similar logic copied
between them. A little bit of refactoring and a new library (and a few new
include files) was born. Rosella Parse is a simple library for implementing
some basic character-wise string parsing routines.

Here's an example of using the new parse library to get the parts of a `file://`
URI:

    string l = "file://./path/to/file.txt";
    :(l, var s, var b, int len) = Rosella.Parse.setup_parse(l);

    string file_host;
    string file_path;
    string protocol = Rosella.Parse.parse_alphanumeric(l, s, b, len);

    if (get_next(s, b) != ASCII_COLON ||
        get_next(s, b) != ASCII_SLASH ||
        get_next(s, b) != ASCII_SLASH)
        Rosella.Error.error("File urls should begin with 'file://'");

    :(string host, int marker) = Rosella.Parse.parse_until(l, s, b, len, ASCII_SLASH);
    if (marker == ASCII_NULL)
        Rosella.Error.error("Empty file:// uri");
    if (host != null && host != "")
        file_host = host;

    string path = Rosella.Parse.parse_remainder(l, s, b, len);
    file_path = path;

The variable `s` is a buffer for holding lookahead characters. Variable `b` is
the string iterator.

This library comes with two other components, header files with lots of
defined constants and inline functions (Winxed's version of macros). The file
`Ascii.winxed` is where the `ASCII_` constants are defined. The file
`Parse_builtins.winxed` contains a bunch of low-level inlines for working with
characters:

    // true if we have more chars left to read
    while(have_more_chars(s, b)) {

        // Get the next char from the iterator. Returns ASCII_NULL on empty
        int c = get_next(s, b);

        // Peek at the next char without removing it
        int d = peek_next(s, b);

        // Skip ahead until the next non-whitespace character
        eat_whitespace(s, b);

        // The current character position of the iterator
        int p = current_position(len, s, b);
