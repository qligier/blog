---
title: "Improving performance of the IG Publisher: I/O operations"
date: 2024-01-11
draft: false
tags: ['Development', 'FHIR', 'IDE']
description: "The blog post is about optimizing some I/O operations of the IG Publisher, a tool to build FHIR
implementation guides."
---

The [IG Publisher](https://github.com/HL7/fhir-ig-publisher) is the reference tool to build FHIR
implementation guides. It is a Java application built upon the FHIR Core library, that reads the implementation
guide sources and generates a FHIR package and a web documentation. It is a quite complex application, and new
versions are released quite regularly.

I recently had to build a large implementation guide, and it took the IG Publisher around 3 hours to complete. The
memory usage was quite high (around 20 {{< abbr GiB >}} at peak) and the result was a 2.57 {{< abbr GiB >}}
directory. While waiting for the completion, I was wondering if there was a way to reduce the completion time.

The first step to improve the performance of the IG Publisher is to understand what it does, and which operations
are the most time-consuming. To do that, I cloned the IG Publisher code from the
[GitHub repository](https://github.com/HL7/fhir-ig-publisher), quickly wrote a
`main(String[])` method, and ran it with the
[profiler included in Intellij IDEA](https://www.jetbrains.com/help/idea/profiler-intro.html) on
the [CH Core](https://fhir.ch/ig/ch-core/index.html) implementation guide. This IG has a good
size, not too big, not too small, and it is a good candidate for a performance analysis.

## Analysis of the flame graph

![Flame graph of the IG Publisher before optimization](flamegraph_before.png "Flame graph of the IG Publisher building an implementation guide, before I/O optimization")

My attention was immediately drawn to the `fileToString(String)` method, which take a significant
amount of time (around 7% of the total time). Opening the class confirmed it was used to read the
content of files, an operation that should usually be fast. The code looked like:

```java {hl_lines="3 7 16-17" title="org/hl7/fhir/utilities/TextFile.java"}
public class TextFile {
    public static String fileToString(String src) throws FileNotFoundException, IOException  {
        CSFile f = new CSFile(src);
        if (!f.exists()) {
            throw new IOException("File "+src+" not found");
        }
        FileInputStream fs = new FileInputStream(f);
        try {
            return streamToString(fs);
        } finally {
            fs.close();
        }
    }

    public static String streamToString(InputStream input) throws IOException  {
        InputStreamReader sr = new InputStreamReader(input, "UTF-8");
        StringBuilder b = new StringBuilder();
        int i = -1;
        while((i = sr.read()) > -1) {
            String s = Character.toString(i);
            b.append(s);
        }
        sr.close();

        return b.toString().replace("\uFEFF", "");
    }

    // [...]
}
```

To read a file, the class did:

1. create a custom `File` implementation;
2. open a `FileInputStream` to read the file byte content;
3. create a `InputStreamReader` to convert bytes to UTF-8 chars;
4. finally, create a `StringBuilder` to build a string from the read chars.

One issue with that approach is that we are reading the file byte by byte, which is really inefficient because it
produces a lot of disk accesses. A better approach would be to read the file in larger chunks, before converting
them to UTF-8 strings. Since we want to retrieve the whole file content as a string, we can also read the whole
file content in one go.

Another issue is that the `InputStreamReader` is creating a lot of temporary strings, that will have
to be merged later by the `StringBuilder`. Processing a byte at a time also prevents the code to
optimize compact strings: if a string is composed of only ISO-8859-1/Latin-1 characters, there is no need to
decode it as UTF-8, we can directly use the bytes as chars (thanks to
[JEP 254: Compact Strings](https://openjdk.org/jeps/254)). That is a fast path that the reader
can't take.

Other methods in that class look implemented in a suboptimal way. Let's check their
execution time in the generated profile:

![TextFile methods execution time before optimization](methods_before.png "`TextFile` methods execution time, before I/O optimization")

A few methods are taking a significant amount of time: 19 seconds for the `streamToString(InputStream)`
method, around 5 seconds for the `fileToString()` methods and 3.5 seconds for the
`bytesToString(byte[])` method. I was not expecting such long execution times, especially for the last
one. Let's work on optimizing that.

## Optimize the code and verify the results

To optimize the `fileToString(String)` method, the simplest solution is to use the
`java.nio.file.Files.readString(Path)`. But while testing the modified code, errors started to appear
when reading Excel spreadsheets. After some investigation, I found that the `readString(Path)` methods
failed while reading invalid UTF-8 byte sequences. An almost equivalent method that does not fail in that case is
`new String(Files.readAllBytes(Path), Charset)`. Both methods use the `readAllBytes(Path)`
method, but use different UTF-8 decoders.

For many methods, I avoided using `InputStream`, `OutputStream`,
`InputStreamReader` and `OutputStreamWriter` as much as possible. The Java NIO API is quite
helpful to write fewer lines of code, while directly getting a quite optimized implementation.

After those changes, it is time to run the profiler again, and compare the results:

![Flame graph of the IG Publisher after optimization](flamegraph_after.png "Flame graph of the IG Publisher building an implementation guide, after I/O optimization")

The `fileToString(String)` method is barely visible on the flame graph, and it represents now only
0.36% of the total time. It seems that the optimization was successful. To be sure, let's compare the execution
time of the methods in the `TextFile` class after the optimization:

![TextFile methods execution time after optimization](methods_after.png "`TextFile` methods execution time, after I/O optimization")

All methods are now under the 0.4 seconds threshold, which is a significant improvement. The
`streamToString(InputStream)` method is 51 times faster, the `fileToString(String)` method
18 times faster. Not bad for a small patch that also simplifies the code!

Switching from execution time to memory allocation view, the improvement is also significant: the
`streamToString(InputStream)` method allocates 300 {{< abbr MiB >}} instead of 23 {{< abbr GiB >}}, the
`fileToString(String)` method 545 {{< abbr MiB >}} instead of 6 {{< abbr GiB >}}. Avoiding the
creation of temporary objects prevents a lot of memory allocation.

I also created unit tests for that class, to ensure that the behavior is the same as before. The patch is now
ready to be submitted to the core library.

## Further optimizations

I was surprised to find code that manage UTF-8 byte order mark (BOM) in various methods that read or write files.
It is becoming really rare to encounter a UTF-8 BOM in the wild, because they are ultimately useless: the byte
order of UTF-8 is fixed, contrary to UTF-16 and UTF-32. While their use in UTF-8 is allowed, the specifications
explicitly discourage it. Removing that support would not really optimize the code, but it would surely make it
simpler.
