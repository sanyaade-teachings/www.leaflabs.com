Title: Making libmaple compile faster
Date: 2012-08-01 15:02
Author: mbolivar
Category: Uncategorized
Slug: 2549

Recently, we've been playing around with ways to get  <a href="http://leaflabs.com/docs/libmaple.html">libmaple</a> to compile faster. We've been able to get good results (0.1  seconds for a full build); this post summarizes them. Results were obtained with  the <a href="http://leaflabs.com/docs/unix-toolchain.html">UNIX toolchain</a>, but it might be interesting to try to port them over to the <a href="http://leaflabs.com/docs/ide.html">IDE</a>. All timing results are for building <a href="https://github.com/leaflabs/libmaple/blob/0cc911c4aee2f8695f0ee93f2a189b5e0e259fb9/examples/blinky.cpp">blinky.cpp</a> on today's libmaple master (<a href="https://github.com/leaflabs/libmaple/tree/0cc911c4aee2f8695f0ee93f2a189b5e0e259fb9/">0cc911c</a>) using a quad-core hyper-threaded CPU (an Intel i7-3770 at 3.4 GHz).


TLDR:
<ol>
	<li>Use make's -j flag to parallelize the build</li>
	<li>Use ccache to save old build files</li>
</ol>
<!--more--><strong>Use make's <code>-j</code> flag for parallel builds:</strong> this one is a no-brainer.

Building with <code>make -jN</code> allows N separate jobs (e.g.  compilations) to proceed at the same time. The "right" value of N  depends on the computer you're using to build libmaple; we usually use  the common heuristic N = number of cores.

Without -j:

    :::text
    $ time make
    [...]
    real	0m3.570s
    user	0m2.664s
    sys	0m0.400s

That's 3.570 seconds of wall clock time. With -j8:

    :::text
    $ time make -j8
    [...]
    real	0m0.825s
    user	0m3.788s
    sys	0m0.476s

0.825 seconds, or a little <strong>more than a 4x speedup</strong>. (<code>make
-j4</code> usually results in ~1 second builds; increasing N beyond 8 seems to
have little effect.)

<strong>Use ccache to cache old build files:</strong> We recommend  this in general, but it really shines if you use more than one kind of  Maple board (regular Maple, Maple Mini, etc.) or have to <code>make clean</code> a lot for some other reason.


<a href="http://ccache.samba.org/">ccache</a> is a compiler cache. It  saves the results of old compilations, letting you reuse them even if  you've deleted the versions in the libmaple build directory. Since you  have to run <code>make clean</code> before rebuilding libmaple when  switching boards, having a cache of libmaple's object files for the  other board saves time when you switch back.


Results using ccache with an empty cache:

    :::text
    $ time make
    [...]
    real	0m4.489s
    user	0m3.056s
    sys	0m0.488s

Running it multiple times, it seems that this number (~4.5 seconds) is  typical. Note that that's a little under a second more than building  libmaple without ccache. The extra time is presumably spent warming  up the cache.


The results when rebuilding libmaple with a warm cache are pretty impressive:

    :::text
    $ make clean; time make
    real	0m0.352s
    user	0m0.064s
    sys	0m0.040s

That's <strong>more than a 10x speedup</strong> over rebuilding libmaple
without ccache.

If you'd like to try this for yourself, you can use the following libmaple
patch:

    :::diff
    $ diff --git a/support/make/build-rules.mk b/support/make/build-rules.mk
    index 3d541ba..1a2abbb 100644
    --- a/support/make/build-rules.mk
    +++ b/support/make/build-rules.mk
    @@ -1,9 +1,9 @@
    # Useful tools
    -CC       := arm-none-eabi-gcc
    -CXX      := arm-none-eabi-g++
    +CC       := ccache arm-none-eabi-gcc
    +CXX      := ccache arm-none-eabi-g++
    LD       := arm-none-eabi-ld -v
    AR       := arm-none-eabi-ar
    -AS       := arm-none-eabi-gcc
    +AS       := ccache arm-none-eabi-gcc
    OBJCOPY  := arm-none-eabi-objcopy
    DISAS    := arm-none-eabi-objdump
    OBJDUMP  := arm-none-eabi-objdump

(Save the patch to a file, then run <code>$ git apply the-file-name</code> from
within libmaple).

<strong>Parting shot</strong>: the combination of the two (make -j8 plus warm
cache):

    :::text
    $ make clean; time make -j8
    [...]
    real	0m0.132s
    user	0m0.064s
    sys	0m0.020s

About a <strong>27x speedup</strong> over  plain <code>make</code>. Which is
awesome, but  note that it's less than the product of the speedups of using -j8
(4x)  and ccache (10x). Presumably the fixed overhead is non-parallelizable,
non-cacheable build tasks like the final link.

