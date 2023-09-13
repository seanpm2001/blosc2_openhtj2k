# Blosc2 OpenHTJ2K

This is a dynamic codec plugin for Blosc2 that allows to compress and decompress images
using the High Throughput JPEG 2000 standard. The HT version brings increased performance
when compared to the traditional JPEG 2000.  For details, check the
[HTJ2K whitepaper](https://ds.jpeg.org/whitepapers/jpeg-htj2k-whitepaper.pdf).

To provide this feature this plugin uses the
[OpenHTJ2K](https://github.com/osamu620/OpenHTJ2K) library.

# Install

The Blosc2 OpenHTJ2K plugin is distributed as a Python wheel:

    pip install blosc2-openhtj2k

There are wheels for Linux, macOS and Windows. As of now, only the x86-64 architecture is
supported.

# Usage from Python

The examples are not distributed with the wheel, but you can just clone the project:

    git clone https://github.com/Blosc/blosc2_openhtj2k.git
    cd blosc2_openhtj2k

In the examples folder there are the compress and decompress scripts, both take two
required arguments, for the input and output file:

- `compress.py` takes as first argument the path to an image, and the second argument the
  path to the output file, which should end by the `.b2nd` extension.

- `decompress.py` takes as first argument the path to the Blosc2 file generated by
  `compress.py`, and second argument the path to the output image.

To try out these scripts first install the required software:

    pip install blosc2-openhtj2k
    pip install Pillow

Then you can run the scripts, from the examples folder, for example:

    cd examples
    python compress.py kodim23.png /tmp/kodim23.b2nd
    python decompress.py /tmp/kodim23.b2nd /tmp/kodim23.png

Note that the examples cannot be run from the project's root, because it will fail to
import `blosc2_openhtj2k`, since there's a directory with that name.

For details on the arguments these commands accept call them with the `--help` option.

Below follows more detailed docs on how to accomplish the compression and decompression.

## Compression

### Load the image

To compress an image first we need to load it, and to transform it to a Numpy array, then
Blosc2 will compress that array.

For loading the image, and getting the Numpy array, we are going to use the Pillow
library:

    from PIL import Image
    im = Image.open(args.inputfile)
    np_array = np.asarray(im)

### Transform the Numpy array

Before feeding this array to Blosc2, we need to massage it a bit, because its structure
is different from what is expected by the OpenHTJ2K plugin. As can be seen in the
`compress.py` script, these are the transformations required, with comments:

    # Transpose the array so the channel (color) comes first
    # Change from (height, width, channel) to (channel, width, height)
    np_array = np.transpose(np_array, (2, 1, 0))

    # Make the array C-contiguous
    np_array = np_array.copy()

    # The library expects 4 bytes per color (xx 00 00 00), so change the type
    np_array = np_array.astype('uint32')

### Plugin options

It's possible to configure the OpenHTJ2K plugin with a number of options, this step is
optional. For example:

    import blosc2_openhtj2k
    blosc2_openhtj2k.set_params_defaults(
        transformation=0,   # 0:lossy 1:lossless (default is 1)
    )

Once the options above are set, these remain for all future calls, until they are changed
again.

### Blosc2 parameters

Note that:

- We must tell Blosc2 to use the OpenHTJ2K codec, passing its corresponding id `BLOSC_CODEC_OPENHTJ2K`.

- At this time the plugin does not support multithreading, so the number of threads must
  be explicitly defined to 1.

- OpenHTJ2K expects to work with images, so this plugin won't work well when combined
  with regular Blosc2 filters (`SHUFFLE`, `BITSHUFFLE`, `BYTEDELTA`...), nor with split mode,
  because they change the image completely; so these must be reset.

With that, we typically define the compression and decompression parameters like this:

    nthreads = 1
    cparams = {
        'codec': blosc2.Codec.OPENHTJ2K,
        'nthreads': nthreads,
        'filters': [],
        'splitmode': blosc2.SplitMode.NEVER_SPLIT,
    }
    dparams = {'nthreads': nthreads}

### Actual OpenHTJ2K compression

Now we can call the command that will compress the image using Blosc2 and the OpenHTJ2K
plugin:

    bl_array = blosc2.asarray(
        np_array,
        chunks=np_array.shape,
        blocks=np_array.shape,
        cparams=cparams,
        dparams=dparams,
    )

Note that:

- We set the chunk and block shape to match the image size, as this is well tested.

- If you pass the `urlpath` the Blosc2 array will be saved to the given path, for
  example `urlpath=/tmp/image.b2nd'.


## Decompression

If the Blosc2 array was saved to a file with a different program, we will need to read it
first:

    array = blosc2.open(args.inputfile)

Now decompressing it is easy, we will get a Numpy array:

    np_array = array[:]

But prior to obtain the image, we must undo the transformations done when compressing:

    # Get back 1 byte per color, change dtype from uint32 to uint8
    np_array = np_array.astype('uint8')

    # Get back the original shape: height, width, channel
    np_array = np.transpose(np_array, (2, 1, 0))

Now we can get the Pillow image from the Numpy array:

    im = Image.fromarray(np_array)

Which can be saved or displayed:

    im.save(...)
    im.show()
