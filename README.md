# vmbsrc
This project contains the official GStreamer plugin to make cameras supported by Allied Visions
Vimba X API available as GStreamer sources.

GStreamer is a multimedia framework which assembles pipelines from multiple elements. Using the
`vmbsrc` element it is possible to record images with industrial cameras supported by Vimba X and
pass them directly into these pipelines. This enables a wide variety of uses such as live displays
of the image data or encoding them to a video format.

## Building
A `CMakeLists.txt` file is provided that helps build the plugin. For convenience this repository
also contains `CMakePresets.json` and `CMakeUserPresets.json.TEMPLATE`, which define configuration
presets for different platforms. This is an easy way to provide paths to external build dependencies
like Vimba X.

To use these presets, create a copy of `CMakeUserPresets.json.TEMPLATE` and name it
`CMakeUserPresets.json`. In this file adjust the value for `PATH_TO_VimbaX` (on Windows also adjust
`PATH_TO_GSTREAMER_INSTALLATION`).

To use the preset for configuration, pass the name as parameter to cmake by calling for example
`cmake --preset linux64`. This will create a `build-linux64` directory. To build the `vmbsrc` binary
from this, execute `cmake --build build-linux64`. On different platforms, different preset names and
directory names must be used (e.g. `arm64` or `win64`).

### Docker build environment (Linux only)
To simplify the setup of a reproducible build environment, a `Dockerfile` based on an Ubuntu 18.04
base image is provided, which when build includes all necessary dependencies, except the Vimba X
version against which `vmbsrc` is linked. This is added when the compile command is run by mounting
a Vimba X installation into the Docker container as a volume.

#### Building the docker image
In order to build the docker image from the `Dockerfile`, run the following command inside the
directory containing it:
```
docker build -t gst-vmbsrc:18.04 .
```

#### Compiling vmbsrc using the Docker image
After running the build command described above, a Docker image with the tag `gst-vmbsrc:18.04` will
be created. This can be used to run the build process of the plugin.

Building the plugin with this image is simply a matter of mounting the source code directory and the
desired Vimba X installation directory into the image at appropriate paths. The expected paths into
which to mount these directories are:
- **/gst-vmbsrc**: Path inside the Docker container to mount the gst-vmbsrc project
- **/vimbax**: Path inside the Docker container to mount the desired Vimba X installation

The full build command to be executed on the host would be as follows:
```
docker run --rm -it --volume /path/to/gst-vmbsrc:/gst-vmbsrc --volume /path/to/VimbaX:/vimbax gst-vmbsrc:18.04
```
The image will run the required `cmake` commands to compile the `vmbsrc` element.

## Installation
GStreamer plugins become available for use in pipelines when GStreamer is able to load the shared
library containing the desired element. GStreamer typically searches the directories defined in
`GST_PLUGIN_SYSTEM_PATH`. By setting this variable to a directory and placing the shared library
file in it, GStreamer will pick up the `vmbsrc` element for use.

The `vmbsrc` element uses VmbC. In order to be usable the VmbC shared library therefore needs to
also be loadable when the element is started. The VmbC library is provided as part of Vimba X.

More detailed installation instructions for Linux and Windows can be found in the `INSTALLING.md`
file in this repository.

## Usage
**Please keep the [Known issues and limitations](##Known-issues-and-limitations) in mind when
specifying your GStreamer pipelines and check the [Troubleshooting](##Troubleshooting) section if
you encounter any issues**

`vmbsrc` is intended for use in GStreamer pipelines. The element can be used to forward recorded
frames from a Vimba X compatible camera into subsequent GStreamer elements.

The following pipeline can for example be used to display the recorded camera image. The
`camera=<CAMERA-ID>` parameter needs to be adjusted to use the correct camera ID.
```
gst-launch-1.0 vmbsrc camera=DEV_1AB22D01BBB8 ! videoscale ! videoconvert ! queue ! autovideosink
```

For further usage, also take a look at the included `EXAMPLES.md` file

### Setting camera features
To adjust the image acquisition process of the camera, access to settings like the exposure time are
necessary. The `vmbsrc` element provides access to these camera features two ways.
1. If given, an XML file defining camera features and their corresponding values is parsed and all
   features contained are applied to the camera (see [Using an XML file](####Using-an-XML-file))
2. Otherwise selected camera features can be set via properties of the `vmbsrc` element (see
   [Supported via GStreamer properties](####Supported-via-GStreamer-properties))

The first approach allows the user to freely modify all features the used camera supports. The
second one only gives access to a small selection of camera features that are supported by many, but
not all camera models. The feature names (and in case of enum features their values) follow the
Standard Feature Naming Convention (SFNC) for GenICam devices. For cameras not implementing the
SFNC, this may lead to errors in setting some camera features. For these devices the feature setting
via a provided XML file is recommended.

#### Using an XML file
Providing an XML file containing the desired feature values allows access to all supported camera
features. A convenient way to creating such an XML file is configuring your camera as desired with
the help of the Viewer that is part of Vimba X and saving the current configuration as an XML file
from there. The path to this file may then be passed to `vmbsrc` via the `settingsfile` property as
shown below
```
gst-launch-1.0 vmbsrc camera=DEV_1AB22D01BBB8 settingsfile=path_to_settings.xml ! videoscale ! videoconvert ! queue ! autovideosink
```

**If a settings file is used no other parameters passed as element properties are applied as feature
values.** This is done to prevent accidental overwriting of previously set features. One exception
from this rule is the format of the recorded image data. For details on this particular feature see
[Supported pixel formats](###Supported-pixel-formats).

#### Supported via GStreamer properties
A list of supported camera features can be found by using the `gst-inspect` tool on the `vmbsrc`
element. This displays a list of available "Element Properties", which include the available camera
features. **Note that these properties are only applied to their corresponding feature, if no XML
settings file is passed!**

For some of the exposed features camera specific restrictions in the allowed values may apply. For
example the `Width`, `Height`, `OffsetX` and `OffsetY` features may only accept integer values
between a minimum and a maximum value in a certain interval. In cases where the provided value could
not be applied a logging message with level `WARNING` is printed (make sure that an appropriate
logging level is set: e.g. `GST_DEBUG=vmbsrc:WARNING` or higher) and image acquisition will proceed
with the feature values that were initially set on the camera.

In addition to the camera features listed by `gst-inspect`, the pixel format the camera uses to
record images can be influenced. For details on this see [Supported pixel
formats](###Supported-pixel-formats).

### Supported pixel formats
As the pixel format has direct impact on the layout of the image data that is moving down the
GStreamer pipeline, it is necessary to ensure, that linked elements are able to correctly interpret
the received data. This is done by negotiating the exchange format of two elements by finding a
common data layout they both support. Supported formats are reported as `caps` of an elements pad.
In order to support this standard negotiation procedure, the pixel format is therefore set depending
on the negotiated data exchange format, instead of as a general element property like the other
camera features.

Selecting the desired format can be achieved by defining it in a GStreamer
[capsfilter](https://gstreamer.freedesktop.org/documentation/coreelements/capsfilter.html) element
(e.g. `video/x-raw,format=GRAY8`). A full example pipeline setting the GStreamer GRAY8 format is
shown below. Selecting the GStreamer GRAY8 format will set the Mono8 Vimba X Format in the used
camera to record images.
```
gst-launch-1.0 vmbsrc camera=DEV_1AB22D01BBB8 ! video/x-raw,format=GRAY8 ! videoscale ! videoconvert ! queue ! autovideosink
```

Not all Vimba X pixel formats can be mapped to compatible GStreamer video formats. This is
especially true for the "packed" formats. The following tables provide a mapping where possible.

#### GStreamer video/x-raw Formats
| Vimba X Format      | GStreamer video/x-raw Format | Comment                                                                     |
|---------------------|------------------------------|-----------------------------------------------------------------------------|
| Mono8               | GRAY8                        |                                                                             |
| Mono10              | GRAY16_LE                    | Only the 10 least significant bits are filled. Image will appear very dark! |
| Mono12              | GRAY16_LE                    | Only the 12 least significant bits are filled. Image will appear very dark! |
| Mono14              | GRAY16_LE                    | Only the 14 least significant bits are filled. Image will appear very dark! |
| Mono16              | GRAY16_LE                    |                                                                             |
| RGB8                | RGB                          |                                                                             |
| RGB8Packed          | RGB                          | Legacy GigE Vision Format. Does not follow PFNC                             |
| BGR8                | BGR                          |                                                                             |
| BGR8Packed          | BGR                          | Legacy GigE Vision Format. Does not follow PFNC                             |
| Argb8               | ARGB                         |                                                                             |
| Rgba8               | RGBA                         |                                                                             |
| Bgra8               | BGRA                         |                                                                             |
| Yuv422              | UYVY                         |                                                                             |
| Yuv422Packed        | UYVY                         | Legacy GigE Vision Format. Does not follow PFNC                             |
| YCbCr422_8_CbYCrY   | UYVY                         |                                                                             |
| Yuv444              | IYU2                         |                                                                             |
| Yuv444Packed        | IYU2                         | Legacy GigE Vision Format. Does not follow PFNC                             |
| YCbCr8_CbYCr        | IYU2                         |                                                                             |


#### GStreamer video/x-bayer Formats
The GStreamer `x-bayer` formats in the following table are compatible with the GStreamer
[`bayer2rgb`](https://gstreamer.freedesktop.org/documentation/bayer/bayer2rgb.html) element, which
is able to debayer the data into a widely accepted RGBA format.

| Vimba X Format      | GStreamer video/x-bayer Format |
|---------------------|--------------------------------|
| BayerGR8            | grbg                           |
| BayerRG8            | rggb                           |
| BayerGB8            | gbrg                           |
| BayerBG8            | bggr                           |

## Troubleshooting
- The `vmbsrc` element is not loadable
  - Ensure that the installation of the plugin was successful and that all required dependencies are
    available. Installation instructions can be found in `INSTALLING.md`. To verify that the element
    can be loaded try to inspect it by calling `gst-inspect-1.0 vmbsrc`.
  - It is possible that the `vmbsrc` element is blacklisted by GStreamer. This can be determined by
    checking the list of blacklisted elements with `gst-inspect-1.0 --print-blacklist`. Resetting
    the registry may be done by removing the file in which it is stored. It is typically saved in
    `~/.cache/gstreamer-1.0/registry.x86_64.bin`. The exact file name depends on the architecture of
    the system. For more details see [the official documentation on the
    registry](https://gstreamer.freedesktop.org/documentation/gstreamer/gstregistry.html)
- How can I enable logging for the plugin
  - To enable logging set the `GST_DEBUG` environment variable to `GST_DEBUG=vmbsrc:DEBUG` or
    another appropriate level. For further details see [the official
    documentation](https://gstreamer.freedesktop.org/documentation/tutorials/basic/debugging-tools.html)
- The camera features are not applied correctly
  - Not all cameras support access to their features via the names agreed upon in the Standard
    Feature Naming Convention. If your camera uses different names for its features, consider [using
    an XML file to pass camera settings](####Using-an-XML-file) instead.
- When displaying my images I only see black
  - This may be due to the selected pixel format. If for example a Mono10 pixel format is chosen for
    the camera, the resulting pixel intensities are written to 16bit fields in the used image buffer
    (see table in [video/x-raw formats overview](####GStreamer-video/x-raw-Formats)). Because the
    pixel data is only written to the least significant bits, the used pixel intensity range does
    not cover the expected 16bit range. If the display assumes the 16bit data range to be fully
    utilized, your recorded pixel intensities may be too small to show up on your display, because
    they are simply displayed as very dark pixels.
- When displaying my images I only see green
  - This is possibly caused by an error in the `videoconvert` element. Try enabling error messages
    for all elements (`GST_DEBUG=ERROR`) to see if there is a problem with the size of the data
    buffer. If that is the case check the troubleshooting entry to "The `videoconvert` element
    complains about too small buffer size"
- The `videoconvert` element complains about too small buffer size
  - This is most likely caused by the width of the image data not being evenly divisible by 4 as the
    `videoconvert` element expects. Try setting the width to a value that is evenly divisible by 4.

## Known issues and limitations
- In situations where cameras submit many frames per second, visualization may slow down the
  pipeline and lead to a large number of incomplete frames. For incomplete frames warnings are
  logged. The user may select whether they want to drop incomplete frames (default behavior) or to
  submit them into the pipeline for processing. Incomplete frames may contain pixel intensities from
  old acquisitions or random data. The behavior is selectable with the `incompleteframehandling`
  property.
- Complex camera feature setups may not be possible using the provided properties (e.g. complex
  trigger setups for multiple trigger selectors). For those cases it is recommended to [use an XML
  file to pass the camera settings](####Using-an-XML-file).

## Compatibility
`vmbsrc` is currently officially supported on the following operating systems and architectures:
- AMD64 (Validated on Ubuntu 22.04, Debian 11.6)
- ARM64 (Validated on NVIDIA L4T 35.2.1)

The following library versions have been validated to work with vmbsrc:
- Vimba X 2023-1
- GStreamer 1.20 (Ubuntu 22.04)
- GStreamer 1.16 (NVIDIA L4T 35.2.1)

# Command
`cmake --preset linux64 -D Vmb_DIR=/opt/VimbaX_2023-4/api/lib/cmake/vmb && cmake --build build-linux64`
`sudo cp build-linux64/libgstvmbsrc.so /usr/lib/x86_64-linux-gnu/gstreamer-1.0`