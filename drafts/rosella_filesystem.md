This post is extremely late, but it's still worth mentioning. A few weeks ago
I did all the necessary work to get the Rosella FileSystem library listed as
stable. "Stable", as always, doesn't mean the library is perfect. Instead, it
means that the library has a stable design, a generally stable interface, and
it's usable right now by anybody who is interested to try it out.

Parrot offers some tools for working with files and directories. Mostly,
these tools involve use of the OS PMC, a dynamically-loaded PMC type that is
not built in to Parrot core. For simple IO operations on files Parrot does
provide the FileHandle PMC type, but for things like working with the
organization of files and directories you need to use OS. Also, the OS PMC
doesn't quite do everything either. There is also a dynamically-loaded ops
library that provides a `stat` wrapper op, in addition to some other
convenience ops for working with FileHandles (if the methods on FileHandle to
do the exact same things are inexplicably not what you needed).

What Rosella FileSystem library does is provide nicer, friendlier wrappers
around all these things. The library itself is pretty heavily inspired by
`System.IO` from the .NET standard library, and some items from Python's
`os` module too.

Files and directories are all instances of `Rosella.FileSystem.Entry`. There
are two subclasses: `Rosella.FileSystem.File` and
`Rosella.FileSystem.Directory`. You can create instances of these objects
using simple constructors with path strings. Here are some Winxed code
examples:

    var file = new Rosella.FileSystem.File("foo.txt");
    var dir = new Rosella.FileSystem.Directory("bar");

Or, if you want to create a file in a directory, you can do something like
this:

    var dir = new Rosella.FileSystem.Directory("bar");
    var file = new Rosella.FileSystem.File("foo.txt", dir);

All `Entry` objects have some common methods:

    entry.exists()      # 1 if it exists. 0 otherwise
    entry.delete()      # Delete (non-recursive delete for Directories)
    entry.rename("baz") # Rename it
    string n = entry.short_name()   # The short-name of the entry (no path)

Directories add in a few other features:

    dir.delete_recursive()          # Delete with all contents
    dir.create()                    # Create it, if it doesn't exist
    var files = dir.get_files()     # Get a list of all File objects in it
    var dirs = dir.get_subdirectories() # Get a list of all Directories in it
    var entries = dir.get_entries() # Get all entries, File and Directory
    var entry = dir["foo.txt"]      # Get the entry by name. File or Directory
    var entry = dir["baz"]          # Same
    dir.walk(visitor)               # Walk contents, using a Visitor
    dir.walk_func(func)             # Walk contents, using a Sub on each File
    exists dir["foo"]               # Determine if "foo" exists
    delete dir["foo"]               # Delete "foo", if it exists

Files likewise add in some features of their own:

    var fh = file.open_read()       # Get FileHandle, opened for reading
    var fh = file.open_write()      # Get FileHandle, opened for writing
    var fh = file.open_append()     # Get FileHandle, open for write/append
    string t = file.read_all_text() # Read all text into a single string
    var t = file.read_all_lines()   # Read all text, as an array of lines
    file.write_all_text(txt)        # Write all text to the file (delete existing contents)
    file.write_all_lines(t)         # Write an array of string lines to file
    file.append_text(t)             # Append some text
    file.copy(dest)                 # Copy a file to dest

Basically, it's a very easy object-oriented interface to common file system
operations. It's nothing fancy, and there are some features missing which
people might expect, but it's very useful and, I think, very usable. Also,
it's not a very thick library so while there is some performance overhead it
isn't too much considering the added convenience.
