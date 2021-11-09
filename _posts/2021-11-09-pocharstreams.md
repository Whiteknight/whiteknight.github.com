---
layout: post
categories: [projects]
title: ParserObjects Character Streams
---

Let's talk streams of characters for a moment. [ParserObjects]() has an input abstraction called `ISequence<T>` which operates similarly to `IEnumerator<T>` but with a few extra features specific to parsing tasks. At any given point a sequence can:

1. Get the next item, advancing the input position ahead by 1 position. If an attempt is made to read past the end of the input, a special *end sentinel* value is returned instead.
2. Report the number of items which have been consumed so far
3. Report the current position in the input (for text this would include Line and Column numbers, etc)
4. Report whether the sequence is at the end
5. Take a checkpoint or continuation which can be used to rewind the sequence to that same position at any time

When talking about character sequences, which are the most common case for most parsing applications, ParserObjects provides two implementations: `StringCharacterSequence` and `StreamCharacterSequence`. The first of these is very straight-forward: The sequence contains a string and maintains an index value. When we read a character the character at the current index is returned and the pointer is incremented. The second of these is significantly more complicated. Reading characters from a `Stream` using an arbitrary encoding while also providing all the other features required is quite tricky indeed. Let's look at some of the problems.

## Problem 1: Character Encodings

.NET Stores it's strings internally as UTF-16, though this is a relatively rare encoding in general use. Far more common, because it's significantly more compact in most cases, is UTF-8. UTF-8 stores a lot of common and familiar ASCII character values as a single byte (where "common and familiar" are from the American/English perspective), but then uses multi-byte sequences to store other values. UTF-8 may use up to 4 bytes (I seem to remember allowance to use up to 6 bytes, though these aren't needed yet. Forgive me if I am wrong) to store a single character. For example the letter `'a'` is stored simply as the byte `0x61` but something like the EMDASH &mdash; character which is relatively common among HTML folks, is stored in UTF-8 as `0xE28094` for three bytes total. 

Converting bytes to characters is quite tricky, but luckily the .NET framework provides things like `Encoding`, `Decoder` and `StreamReader` to help. Instead of operating directly on the stream of bytes and trying to decode ourselves, we can instead use `StreamReader` to read characters into buffers. Once in a buffer we can use a buffer pointer to keep track of the current character. To read we just return the current character in the buffer and then advance the pointer, just like in our string sequence. The only difference is that when we read past the end of the current buffer we have to refill the buffer.

## Problem 2: Random Seek

Using the above solution with a buffer and `StreamReader` is nice for all requirements *EXCEPT* the requirement to rewind the input to a previous position.  There are two sub-problems here:

1. `StreamReader` keeps it's own internal buffer which it uses to try and figure out where multi-byte characters begin, which means that the underlying `Stream` may be advanced beyond the point of the most recent character you've just read, so you can't get the current logical position of the input just by asking for `reader.BaseStream.Position`.
2. The `StreamReader` doesn't publicly report anything about it's internal buffer, which makes good sense when you want to have qualities like "encapsulation". However it makes a problem because you can't calculate the logical position of the stream by taking it's current position and subtracting out the amount of bytes read ahead by the `StreamReader`'s internal buffer.

If you *could* find a position in the `Stream` to seek to, you would do it with two lines:

```csharp
reader.BaseStream.Seek(position, SeekOrigin.Begin);
reader.DiscardBufferedData();
```

However if you assume variable-length encodings like UTF-8 or UTF-16, the only position you know for certain is valid and not in the middle of a multi-byte character is `0`, and that's not useful at all.

To get around these problems ParserObjects v2.0 and v3.0 used a linked-list of buffers. When the stream was read we allocated a new buffer, and created a linked list of buffers. A checkpoint object would reference the buffer and the index within the buffer. There was no pointer to the beginning of the linked list, so old nodes theoretically would be garbage collected when no more checkpoints were referencing them. If there were no checkpoints referencing a given buffer that means that we had no means of ever rewinding to it. 

## Problem 3: Large Files

See, the thing is, from the GC perspective the linked-list implementation makes good sense. Buffers that you still needed would be kept alive by references in the system and anything that was unreferenced would be collected. The problem is that many of the `IParser` implementations in ParserObjects took a checkpoint at the beginning so that they can rewind if their part or their children fail. If your top-level parser object is a `Rule()` for example, you will have a checkpoint pointing to the first buffer, which will keep it alive and all subsequent buffers as well (being a linked list and all). Again this means that rewinding is fast and then reading through the same input values a second time are even faster because there aren't any buffers to refill. But this performance comes at the cost of high memory usage especially if we're talking about a large input file. And why else would we be using a stream if we didn't have to, considering the performance penalties involved in filling buffers? 

The `Encoding` object includes a method to get the size in bytes of a particular character: `Encoding.GetByteCount()`. Using that I figured I could read a character out of a buffer, figure out how many bytes it must have been, and then use that to keep track of the stream position myself. Now as I read characters I can keep a running count of how many bytes I've read, so that when I take a checkpoint I can just use that calculated value as the position to reset the underlying stream to. This situation worked cleanly and I was almost happy except:

## Problem 4: Surrogate characters

I already mentioned that .NET uses UTF-16 for strings internally. This means that the `char` type is a 16-bit UTF-16 value. This works for most of what we want to do with strings, but not always. UTF-16 isn't a fixed-size encoding. Sometimes a code point may use more than two 16-bit characters to be represented. This is common with new fangled emoji characters that the kids use these days. Let's consider our friend the poop: &#57434;

The poop emoji is represented in UTF-16 as `0xD83D 0xDCA9`. The first char is the "high surrogate" (`char.IsHighSurrogate('\uD83D') == true`) and the second is the "low surrogate" (`char.IsLowSurrogate('\uDCA9') == true`). Now here's the new problem: Because poop can't fit into a single `char`, the `StreamReader` puts two characters into the output buffer: the high surrogate and then the low surrogate. But for whatever reason, when we ask the `UTF8Encoding.GetByteCount()` method how many bytes each of these is represented by, the answer comes back as `3` for each. In other words, the poop emoji as a single codepoint is encoded in UTF-8 as 4 bytes, but the byte `0xD83D` when treated like a single code point is encoded as 3 bytes. It's incorrect to ask the UTF-8 encoder how many bytes it would take to encode an incomplete codepoint.

The solution to this problem is that when we see a high surrogate character, we must immediately look for the corresponding low surrogate character, use both to calculate the size in bytes of the codepoint, and then advance the stream by that much. On the next read we find the low surrogate which we've already obtained and cached away safely, and do not advance the stream position at all.

## Problem 5: Mid-Surrogate Checkpoints

Consider this code example, borrowed from the unit test suite:

```csharp
var memoryStream = new MemoryStream();
var b = Encoding.UTF8.GetBytes("\U0001F4A9");
memoryStream.Write(b, 0, b.Length);
memoryStream.Seek(0, SeekOrigin.Begin);
var input = new StreamCharacterSequence(memoryStream, Encoding.UTF8);

target.GetNext().Should().Be('\uD83D');
target.Checkpoint();
target.GetNext().Should().Be('\uDCA9');
```

Notice especially the second line from the bottom, where we take the `.Checkpoint()` between the high and low surrogates of a single logical character. What do we expect to happen here? Remember that we've gone through some pretty great lengths so far to prevent the stream from being reset to an invalid location in the middle of a character. Setting a checkpoint in between surrogates of a single checkpoint would seem to be disregarding all that effort completely! Considering that we can't even calculate stream position between two surrogates because we need both to get the encoded size of the codepoint, this series of events seems nonsensical. In fact, it is. 

I generally don't like to throw exceptions from library code, especially not when a user has made a perfectly reasonable-looking method call with perfectly normal-looking data. It's better to try and design your interface to make nonsensical operations impossible to even express. However in this case I really couldn't find any other good solution to this problem and I opted to throw an exception.

In ParserObjects v3.1, if you are using a `StreamCharacterSequence` and try to set a checkpoint between a high and low surrogate of a single codepoint, it will throw an `InvalidOperationException`. This, in effect, is a breaking change from behavior in v3.0 and earlier. I'm not going to bump the major number for this, because I suspect it will affect relatively few users, but it's something that I wanted to make clear.

(The old behavior, of allowing you to set the checkpoint in the middle but then just like crapping all over the floor when you tried to rewind to it, is not behavior that I think is worth preserving)

## Recap: How Did We Get Here?

In v3.0 and earlier versions of ParserObjects the `StreamCharacterSequence` would keep a copy of all read data (for the most part) from the stream. This is bad because very large files or large streams will end up leading to huge amounts of memory consumption during your parse. I wanted to optimize this by using a single buffer with Stream seek, but to do that I needed to be able to calculate stream position because the built-in `StreamReader` type doesn't report it accurately. But, in order to calculate stream position I needed to make surrogate `char`s indivisible. Now the system should use significantly less memory when parsing large files or streams, but at the cost of throwing an exception if you try to set a checkpoint between surrogates.

Considering that this is technically a breaking change, I guess I could have saved this for a v4.0 release or maybe just gone straight to v4.0 instead of v3.1. Considering that this is a relatively small issue, there are lots of other features in this release which are more important and are also non-breaking, and I have a larger list of breaking changes planned to become a v4.0 in the future, I decided to just keep this release as v3.1. 



