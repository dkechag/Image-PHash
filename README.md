# NAME

Image::PHash - Fast perceptual image hashing (DCT-based pHash)

# SYNOPSIS

    use Image::PHash;

    # Load an image and prepare to hash.
    # Will try to find an image library to load and resize the image to 32x32, ready for the DCT step
    my $iph = Image::PHash->new($image_file);

    # Calculate the perceptual hash (top 8x8 of the DCT) - 64 bits, 16 hex chars
    my $p = $iph->pHash(); # implies settings geometry => '8x8', method => 'average'

    # Alternative, better performing, 64 bit hash (64 upper-left most DCT values)
    my $p = $iph->pHash(geometry => 64); # this is actually recommended over the default hash

    # Alternative method for creating the bitmask with lower false-negative rate
    my $p = $iph->pHash(geometry => 64, method => 'log');

    # Calculate a pHash with the upper left half of the upper-left-most 7x7 of the DCT - 27 bits, 7 hex chars
    # Use for indexing or reducing full phash false negatives - significant false negative rate by itself
    my $p7 = $iph->pHash(geometry => '7x7', reduce => 1);
    # or shortcut function
    $p7 = $iph->pHash7();

    # Calculate a pHash with the upper left half of the upper-left-most 6x6 of the DCT - 20 bits, 5 hex chars
    # Use for indexing or reducing full phash false negatives - very significant false negative rate by itself
    my $p6 = $iph->pHash(geometry => '6x6', reduce => 1, method => 'median');
    # or shortcut function
    $p6 = $iph->pHash6();

    # Calculate the difference (Hamming distance) between two hex hashes
    my $diff = Image::PHash::diff($p1, $p2);

# DESCRIPTION

Image::PHash allows you to calculate the (DCT-based) perceptual hash (pHash) of an image.

The constructor and general structure is based on [Image::Hash](https://metacpan.org/pod/Image%3A%3AHash) - keeping usage quite
similar, but the pHash algorithm is rewritten from scratch as the [Image::Hash](https://metacpan.org/pod/Image%3A%3AHash)
implementation was flawed (and slow). Apart from fixes/tweaks for [GD](https://metacpan.org/pod/GD), [Imager](https://metacpan.org/pod/Imager), 
[ImageMagick](https://metacpan.org/pod/ImageMagick) resizing, [Image::Imlib2](https://metacpan.org/pod/Image%3A%3AImlib2) support is added along with some unique features
like reduced hashes for indexing, options to deal with image mirroring, alternative
bitmap methods.

A fast DCT XS module made specifically to serve this hashing module is used: [Math::DCT](https://metacpan.org/pod/Math%3A%3ADCT).
Depending on your setup, it should be 15x or so faster than pHash.org

# CONSTRUCTOR METHOD

## `new`

    my $iph = Image::PHash->new($image_file , $library?, \%settings?);
    

The first argument is an image filename when the Imlib2 library is used, but it
can also by a variable with the image data for the other libraries, or even an
image object for the supported libraries.

The second (optional) argument is the image library. Valid options are `Imlib2`,
`GD`, `Imager`, `ImageMagick` and if not specified the module will try to load
them in that order. Using a different library (or even library version) will most
likely result in different hashes being returned, so make sure you hash your entire
image set using the same image library. Also, the module will probably not work with
very old versions of the image libraries. See Notes for comparison of libraries.

The third (optional) argument can take a reference to a settings hash. Currently
supported settings:

- `resize` : Expects an integer to specify the size of the image resize before
applying the DCT transformation. By default it is `32`, which resizes to a `32x32` image.
You may want to explore different sizes which have different hashing behaviour e.g.
increasing resize to `64` seems to offer some benefit for the reduced/index hashes
(at a performance penalty of course).
- `magick_filter` : The `filter` parameter for [ImageMagick](https://metacpan.org/pod/ImageMagick) which controls
the scaling filter. The default is `Cubic` which is about as good as [ImageMagick](https://metacpan.org/pod/ImageMagick)'s
own default `Lanczos`, but significantly faster. You can look up the [ImageMagick](https://metacpan.org/pod/ImageMagick)
documentation for alternatives (e.g. `Lanczos`, `Mitchell`, `Triangle`, `Gaussian`...)
if you want a different balance of speed/quality.
- `imager_qtype` : The `qtype` parameter for [Imager](https://metacpan.org/pod/Imager) which defines the
quality of scaling performed. The type `mixing` is used by default as it seems to
behave well for most cases while being about twice as fast as [Imager](https://metacpan.org/pod/Imager)'s internal
default qtype of `normal`, but you can still manually specify that if you prefer.

    The constructor will `croak` if there is an error loading an image library or an
    incorrect/missing argument. It will only `carp` and return `undef` if an image
    library returns after failing to load an image.

# METHODS

## `pHash`

    my $p = $iph->pHash(
        geometry    => '8x8',     # Size of the matrix to keep from the DCT. 8x8 is the standard 64 bit pHash.
        reduce      => 0,         # If enabled removes lower right half or matrix and 0,0 position.
        method      => 'average', # Method with which to convert the dct to bitmask.
        mirror      => 0,         # If enabled will return the hash of the mirror image (horizontal flip).
        mirrorproof => 0,         # If enabled will return a hash type that resists image mirroring.
    );

    my @bits = $iph->pHash();

Generates a pHash, returns it as a hex string (or array of 1s and 0s in array
context).

The pHash process consists of resizing the image to 32x32 (unless different `resize`)
was specified with the constructor, converting color to luminance, DCT on resized image's
luminance, conversion to bit values based on the selected method and dropping the high
frequency part of the matrix, following the `geometry` and `reduce` settings.

The parameters to pass:

- `geometry` : A string with the desired square dimensions `NxN` of the bit
matrix taken from the upper left of the full (by default 32x32) processed DCT to be
used in the hash. By default it is `'8x8'`, which produces the typical 64bit pHash.

    Alternatively, specify a simple integer for the number of low frequency bits to take
    from the matrix going through the matrix in increasing diagonals.
    E.g. a value of `64` will return 64 bits just like the default pHash, but they will
    be taken from the upper left half of the 11x11 top-left part of the DCT matrix. The
    effect is that the resulting hash will have improved characteristics, especially
    in the average distance of non-similar images.

- `reduce` : The option will return only the bits that are on, or to the upper
left of the 0,N->N,0 diagonal of the selected NxN matrix, except the first bit (which
is always 1). This way you get the (N-1)\*(2+N)/2 most significant (recording the largest
changes) bits. 

    For example, a 8x8 reduced hash has 35 bits, which is less than a 6x6 matrix, and yet
    may outperform a full 7x7 49 bit matrix.

    The option only applies to square `geometry`. If you have specified bits instead,
    you already get an effect similar to `reduce`.

- `method` : Specifies which method to use for converting the DCT result to
a bitmask. By default it is `average`. Supported methods:
    - `average` : The default method which compares each DCT value with the arithmetic
    mean. It usually a bit better at recognising similarity than `median` (the average
    difference of similar images will be lower), albeit with an increase in false positives.
    - `median` : The median of the DCT values is used as the threshold. Some
    implementations prefer it over `average` due to lower false positives/collision rate,
    which can be good for reduced size hashes, but it increases false negatives so it is
    not the recommended method for many scenarios.
    - `average_x` : Applies only to reduced hashes - it will calculate the average using
    the entire NxN matrix for `geometry='NxN'`, or use the next X lowest frequency DCT coefficients
    (total 2\*X) to calculate the average for `geometry=X`. Almost as low collision rate as `log`,
    perhaps a bit better (lower) false negatives.
    - `log` : A special logarithmic average is calculated as the threshold, giving
    the lowest collision rate (a tie with &lt;median> or better). It will usually have a bit
    increased false negative chance compared to `average` or perhaps even `diff` - but
    should still be lower than `median`. Quite close to `average_x`, but also applies to
    non-reduced hashes.
    - `diff` : The difference between each DCT value is taken as the bitmask. It seems
    to fall between `average` and `median` in most tests, both in false negative/collision
    rate and at similarity recognition.
- `mirror` : Returns the pHash of the mirror image. Note that this is a function
applied after the DCT, so if you call `pHash` once with `mirror` and once without
you can get both hashes without a processing overhead. Not compatible with `mirrorproof`.
- `mirrorproof` : Will return a pHash that is impervious to mirroring (flipping
images horizontally). This means two mirrored images will have the same/similar pHash,
so it is good for declaring such images as "similar" with a single pHash comparison, but
if you want to know they were mirrored the `mirror` option is more appropriate.

    Caveat: The option sacrifices about 2 bits or so of entropy, so the resulting
    pHash is less effective. Thus, it should not be preferred if there is no specific
    reason, especially when the `mirror` option is available.

# INDEXING SHORTCUT METHODS

The special reduced hashes below, named **pHash6** and **pHash7** are specially chosen
to be useful in two scenarios:
\* Extra verification to reduce false positives that the full has produces. Depending
on the scenario, tests have shown each of the reduced hash being able to reduce false
positives by 60-90% with the right threshold (example 2, 3 for pHash6, pHash7 resp.),
and if both are used over 95% reduction of false positives is possible.
\* For scenarios where we want to use a simple index of a database (e.g. MySQL) and only
very simple manipulations are required to be matched (e.g. resize) or there is a higher
tolerance for false negatives, pHash6 & pHash7 at diff 0 can be used to retrieve most
matches. For example, storing pHash6, pHash7 (as indices) along with pHash in a MySQL db,
we can retrieve most matches with something like:

    SELECT *
    FROM hash_table
    WHERE (phash7 = @phash7
           OR phash6 = @phash6)
        AND BIT_COUNT(CAST(CONV(phash, 16, 10) AS UNSIGNED) ^ CAST(CONV(@phash, 16, 10) AS UNSIGNED)) < 8;

## `pHash7`

    my $p7 = $iph->pHash7();

Equivalent to `$iph->pHash(geometry => '7x7', reduce => 1)`. It is useful for
relational databases that don't support indexing with differences, so using the reduced
pHash will return many of the close full pHash matches. For simple resize/compression manipulations
expect matches in the region of 98% or so to be returned. It can also be used to verify
the match in some cases where a specific pattern might make different photos match in
the full phash but not the limited. It takes the same parameters as `pHash`.

## `pHash6`

    my $p6 = $iph->pHash6();

Equivalent to `$iph->pHash(geometry => '6x6', reduce => 1, method => 'median')`. Similar
to pHash7, but will produce more matches when used as an index, along with many more false
positives.

Note that neither reduced version is appropriate to use as an image comparison hash by itself
(too many false positives), and they are chosen to be complimentary, so when used in conjunction
for either indexing or verification, their performance increases considerably.

# HELPER METHODS

## `reducedimage`

    my $img = $iph->reducedimage();

Returns the reduced (rescaled) image that will be used for the DCT.

## `dctdump`

    my $dct = $iph->dctdump();

Will return the full 32x32 DCT as an arrayref of floats.

## `printbitmatrix`

    $iph->printbitmatrix(
       %phash_opt,        # Any pHash method option applies
       separator => '',   # Separator for horizontal values
       filler    => ' '   # For reduced results, filler for the missing positions
    );

Will return a print-friendly reduced size bitmask matrix as a string. Basically a
string with rows/columns of the 1s and 0s you would get from calling `$iph->pHash()`
with the same parameters.

# HELPER FUNCTIONS

## `b2h`

    my $hash = Image::PHash::b2h(join('', @bits));

Will convert a bit value string to a hex string.

## `diff`

    my $diff = Image::PHash::diff($phash1, $phash2);

Will calculate the bit difference of two hex string hashes (their Hamming distance
of their bit stream form). On 64 bit systems (checking `$Config{ivsize}`) it will
actually call `_diff64` which can calculate the difference of up to 64bit hashes in
a single operation (using `%064b`). You can call `_diff64` directly if you prefer
in that scenario.

# NOTES

## Performance

The hashing performance of the module is enough to make the actual pHash generation
from the final 32x32 mono image a trivial part of the process. For a general idea,
on a single core of a 2015 Macbook Pro, over 18000 hashes/sec can be processed thanks
in part to the fast [Math::DCT](https://metacpan.org/pod/Math%3A%3ADCT) XS module (developed specifically for Image::PHash).

So, most of the processing time is spent on loading the image, resizing, extracting
pixel values, removing color, all of which depend on the specific image module. On an
Apple M1, hashing 800x600 jpg images was measured at 131 h/s with [Image::Magick](https://metacpan.org/pod/Image%3A%3AMagick),
208 h/s with [Imager](https://metacpan.org/pod/Imager), 241 h/s with [GD](https://metacpan.org/pod/GD), 547 h/s with [Image::Imlib2](https://metacpan.org/pod/Image%3A%3AImlib2).
Higher resolutions make the process slower as you could expect. Since all images will
be resized to 32x32 in the end, the fastest hashing performance would be if you loaded
32x32 thumbnails. In that case, the performance of the libraries in the same order for
the resized imageset were: 659 h/s, 664 h/s, 1883 h/s, 2296 h/s. It is clear that
[Image::Imlib2](https://metacpan.org/pod/Image%3A%3AImlib2) should be preferred when hashing performance is desired, as it offers
dramatically better performance (unless you are hashing 32x32 images in which case [GD](https://metacpan.org/pod/GD)
also fast). It should be noted that the resulting hashes don't have exactly the same
behaviour/metrics, due to the different resizing algorithms used, but the differences
seem to be very small. You are encouraged to test on your own data set.

Remember, never mix image libraries (or settings), the hashes will most likely not
be compatible.

Finally, if you are curious about the performance of this module compared to the
C++ pHash.org implementation, pHash.org could achieve 33 h/s with the test setup as
above, making [Image::PHash](https://metacpan.org/pod/Image%3A%3APHash) over 16x faster with Imlib2. With pre-sized 32x32
images, pHash.org ran at 101 h/s (~23x slower than [Image::PHash](https://metacpan.org/pod/Image%3A%3APHash)/Imlin2).

## Compatibility of hashes

As already mentioned, if you produce hashes with different settings, different image
libraries etc, the hashes might not be compatible. It is advisable to even freeze
the version of this module and the image library in a production environment to
avoid any degraded performance.

## Calculation caching

Calculating pHashes with different dct/reduce/median/mirror arguments for the same
image is very fast (when the same object is used), as the resize and DCT transform
will only happen on the very first pHash calculation and are cached for any subsequent
call. You can essentially get the extra phash6/phash7/mirror etc "for free" after
the initial pHash calculation.

## [Image::PHash](https://metacpan.org/pod/Image%3A%3APHash) vs [Image::Hash](https://metacpan.org/pod/Image%3A%3AHash)

While [Image::Hash](https://metacpan.org/pod/Image%3A%3AHash) may still be useful for the aHash and dHash functionality, its
pHash implementation is seriously flawed. It does not actually do a full DCT, using
instead a shortcut that seems to result to hashes with lots of zeros and thus a high
rate of collisions (~2% chance for identical hash on dissimilar images making it useless
for my large data set), which is the reason the hashing was implemented from scratch.
Despite it not doing a full DCT it was really slow (over 80x slower than the XS [Math::DCT](https://metacpan.org/pod/Math%3A%3ADCT)),
so switching to [Image::PHash](https://metacpan.org/pod/Image%3A%3APHash) will give you "correct" hashes at a significant speed
increase, along with several extra features.

## [Image::PHash](https://metacpan.org/pod/Image%3A%3APHash) vs [pHash.org](https://metacpan.org/pod/pHash.org)

Apart from the significant speed advantage of Image::PHash noted above, there are a couple
of important differences, in that pHash.org will apply a 7x7 mean filter to the image
before the resize and the conversion to bits is always done with the median method. This
seems to keep false positives quite low, but its false negatives are higher. Since with
Image::PHash you can get even better hashes with, for example, `geometry=64` and you can
combine them with `method='average_x'` or `method='log'`, you will get even lower false
positive rate than pHash.org, but with less false negatives as well. Feel free to share
your own comparisons with the author if in doubt.

Note that the differences you are to use as a threshold for Image::PHash and pHash.org are
quite different - pHash.org will give about 50% greater diffs on average (e.g. where I would
use 7 for the former, 11 would be the equivalent for the latter).

## Selecting a `diff` threshold

The appropriate `diff` threshold for declaring images as "similar" is not a precise
art and will depend on the application (type of images, tolerance for false positives
etc.). The exact application is very important too, if you have 2 images and want to
check whether they are similar, a false positive rate of even over 1% is fine, in which
case the diff can be chosen to be probably over 10, whereas having a big collection of
photos in which you want to check whether a duplicate exists, requires a very low false
positive rate. Example diff ranges for a full pHash are 3-7 if you want to keep false
positives close to 0%. For the small pHash7 and pHash6 probably not more than 3 and 2
respectively are useful for lookups (and still with lots of false positives as noted above).

# ACKNOWLEDGEMENTS

Initially based on [Image::Hash](https://metacpan.org/pod/Image%3A%3AHash), so some code to do with loading images, pixels
etc has been kept/adapted.

# AUTHOR

Dimitrios Kechagias, `<dkechag at cpan.org>`

# BUGS

Please report any bugs or feature requests either on GitHub, or on RT (via the email
`bug-image-phash at rt.cpan.org` or web interface at [https://rt.cpan.org/NoAuth/ReportBug.html?Queue=Image-PHash](https://rt.cpan.org/NoAuth/ReportBug.html?Queue=Image-PHash)).

I will be notified, and then you'll be notified of progress on your bug as I make changes.

# GIT

[https://github.com/dkechag/Image-PHash](https://github.com/dkechag/Image-PHash)

# COPYRIGHT & LICENSE

Copyright (C) 2022, SpareRoom & Dimitrios Kechagias.

This program is free software; you can redistribute
it and/or modify it under the same terms as Perl itself.
