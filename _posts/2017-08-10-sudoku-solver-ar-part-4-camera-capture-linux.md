---
layout: post
title: "Sudoku Solver AR Part 4: Camera Capture (Linux)"
date: 2017-08-10 09:54:00 -0500
---

_This post is part of a series where I explain how to build an **augmented reality Sudoku solver**. All of the image processing and computer vision is done from scratch. See the [first post]({% post_url 2017-07-20-sudoku-solver-ar-part-1-about %}) for a complete list of topics._

Capturing video on Linux is done using the Video for Linux (V4L2) API. It's actually a really nice API and [very well documented](https://www.kernel.org/doc/html/latest/media/uapi/v4l/v4l2.html). I encourage you to check out the documentation if you need to do something I don't cover here.

I'm going to go over how to find all of the connected cameras, query their supported formats, the simplest method to get a video frame, and convert it to an RGB image.

### List Connected Cameras ###

Cameras on Linux show up as files in the `/dev` directory. They are always listed in the format `video{number}` where {number} can be any number from 0 to 255. Finding all of the connected cameras is as simple as searching the `/dev` directory for filenames that start with `video`.

{% highlight C++ %}
#include <experimental/filesystem>
#include <vector>

...

//Find all of the files in the "/dev" directory that start with "video".
std::vector<std::experimental::filesystem::path> videoDevicePaths;
for(const auto& directoryEntry : std::experimental::filesystem::directory_iterator("/dev"))
{
    const std::string fileName = directoryEntry.path().filename().string();
    if(fileName.find("video") == 0)
        videoDevicePaths.push_back(directoryEntry);
}
{% endhighlight %}

I'm using the [C++17 Filesystem](http://en.cppreference.com/w/cpp/filesystem) library here for convenience. If you're using an older compiler, you can use the [Boost Filesystem](http://www.boost.org/doc/libs/release/libs/filesystem/doc/index.htm) library or [readdir](http://manpages.ubuntu.com/manpages/zesty/en/man3/readdir.3.html).

### Querying Supported Formats ###

Cameras capture video frames in different sizes, pixel formats, and rates. You can query a camera's supported formats to find the best fit for your use or just let the driver pick a sane default.

There's certainly an advantage to picking the formats yourself. For example, since cameras rarely provide an RGB image, if a camera uses a format that you already know how to convert to RGB, you can save time and use that instead. You can also choose to lower the frame rate for better quality at a cost of more motion blur or raise the frame rate for a more noisy image and less motion blur.

The first thing you need to do to query a camera is to open the camera and get a file descriptor. This file descriptor is then used to refer to that specific camera.

{% highlight C++ %}
#include <fcntl.h>
#include <sys/stat.h>
#include <sys/types.h>

...

//Select one of the enumerated cameras from above.
const std::string videoDevicePath = videoDevicePaths[0].string();

//Get a file descriptor for the device.
const int fd = open(videoDevicePath.c_str(),O_RDONLY);
if(fd == -1)
{
    std::cerr << "Could not open video device: " << videoDevicePath << std::endl;
    return false;
}
{% endhighlight %}

When you're done using the camera, don't forget to close the file descriptor.

{% highlight C++ %}
//Close the file descriptor later when done using the camera.
close(fd);
{% endhighlight %}

All camera querying and configuration is done using the [ioctl](http://manpages.ubuntu.com/manpages/zesty/man2/ioctl.2.html) function. It's used by passing an open file descriptor for the device, a constant that indicates the operation to be performed, and a pointer to some type depending on the indicated operation[^1].

Three different versions of `ioctl` are used to query the formats supported by a camera. They give the supported pixel formats, sizes (width and height), and frame intervals[^2]. Querying is done in this same order. That is, each pixel format has its own supported frame sizes which then further restricts the supported frame intervals.

The pixel format is queried by calling `ioctl` with `VIDIOC_ENUM_FMT` as the second argument and a pointer to a `v4l2_fmtdesc` struct as the third argument. The `v4l2_fmtdesc` struct has an index field that indicates which pixel format is populated into the struct by the `ioctl` call. The index is then incremented and `ioctl` is called again until it returns an error code indicating that there are no more supported formats.

{% highlight C++ %}
#include <sys/ioctl.h>
#include <sys/mman.h>

...

//Enumerate all supported pixel formats by incrementing the index field until ioctl
//returns an error.
v4l2_fmtdesc fmtDesc;
fmtDesc.index = 0; //Start index.
fmtDesc.type = V4L2_BUF_TYPE_VIDEO_CAPTURE; //Must be set to V4L2_BUF_TYPE_VIDEO_CAPTURE.
for(;ioctl(fd,VIDIOC_ENUM_FMT,&fmtDesc) != -1;fmtDesc.index++)
{
    //Format identifier used for querying or selecting the format to use. It can be
    //anything from RGB, YUV, JPEG, or more.
    const unsigned int pixelFormat = fmtDesc.pixelformat;

    //Human readable version of pixel format that can be displayed to the user.
    const unsigned char* humanReadablePixelFormat = fmtDesc.description;
}
{% endhighlight %}

The list of possible pixel formats are found in the documentation.

- [RGB Formats](https://www.kernel.org/doc/html/latest/media/uapi/v4l/pixfmt-packed-rgb.html)
- [YUV Formats](https://www.kernel.org/doc/html/latest/media/uapi/v4l/yuv-formats.html#yuv-formats)
- [Other Formats](https://www.kernel.org/doc/html/latest/media/uapi/v4l/pixfmt-reserved.html)

Getting a list of supported sizes is a little trickier. You start by calling `ioctl` with `VIDIOC_ENUM_FRAMESIZES` and a pointer to a `v4l2_frmsizeenum` struct. But after the first call, you need to check if frame sizes are discrete or given as a minimum and maximum that can be adjusted in some step size.

{% highlight C++ %}
//Call once to figure out how frame sizes are specified.
v4l2_frmsizeenum frameSizeEnum;
frameSizeEnum.index = 0; //Start index.
frameSizeEnum.pixel_format = pixelFormat; //A supported pixel format must be specified!
if(ioctl(fd,VIDIOC_ENUM_FRAMESIZES,&frameSizeEnum) == -1)
    return false;

//Frame sizes are specified in discrete units (eg. 320x240 and 640x480).
if(frameSizeEnum.type == V4L2_FRMSIZE_TYPE_DISCRETE)
{
    //Iterate through remaining sizes.
    do
    {
        const unsigned int width = frameSizeEnum.discrete.width;
        const unsigned int height = frameSizeEnum.discrete.height;

        //Make use of width and height here.

        frameSizeEnum.index += 1;
    } while(ioctl(fd,VIDIOC_ENUM_FRAMESIZES,&frameSizeEnum) != -1);
}
//Frame sizes are specified as individual ranges where each size is a fixed step
//difference from one another. Say the width is specified with a range from 32 to 96 in
//16 pixel steps, then the supported widths would be 32, 48, 64, 80, and 96.
else if(frameSizeEnum.type == V4L2_FRMSIZE_TYPE_STEPWISE ||
        frameSizeEnum.type == V4L2_FRMSIZE_TYPE_CONTINUOUS)
{
    //The only difference between V4L2_FRMSIZE_TYPE_STEPWISE and
    //V4L2_FRMSIZE_TYPE_CONTINUOUS is the step size of continuous is always 1.

    //Iterate through every possible size.
    for(unsigned int height = frameSizeEnum.stepwise.min_height;
        height <= frameSizeEnum.stepwise.max_height;
        height += frameSizeEnum.stepwise.step_height)
    {
        for(unsigned int width = frameSizeEnum.stepwise.min_width;
            width <= frameSizeEnum.stepwise.max_width;
            width += frameSizeEnum.stepwise.step_width)
        {
            //Make use of width and height here.
        }
    }
}
{% endhighlight %}

Finally, the list of supported frame intervals is very similar to getting frame sizes. Call `ioctl` with `VIDIOC_ENUM_FRAMEINTERVALS` and a pointer to a `v4l2_frmivalenum` struct. The frame intervals can be given in discrete chunks, incremental steps, or any value within a specified range.

{% highlight C++ %}
//Similar to above, call once to figure out how frame intervals are specified.
v4l2_frmivalenum frameIntervalEnum;
frameIntervalEnum.index = 0;
frameIntervalEnum.pixel_format = pixelFormat; //A supported pixel format must be specified!
frameIntervalEnum.width = width; //A supported width and height must be specified!
frameIntervalEnum.height = height;
if(ioctl(fd,VIDIOC_ENUM_FRAMEINTERVALS,&frameIntervalEnum) == -1)
    return false;

//Frame intervals are specified in discrete units.
if(frameIntervalEnum.type == V4L2_FRMIVAL_TYPE_DISCRETE)
{
    //Iterate through remaining frame intervals.
    do
    {
        assert(frameIntervalEnum.discrete.denominator != 0);
        const float interval = static_cast<float>(frameIntervalEnum.discrete.numerator) /
                               static_cast<float>(frameIntervalEnum.discrete.denominator);

        //Make use of interval here.

        frameIntervalEnum.index += 1;
    } while(ioctl(fd,VIDIOC_ENUM_FRAMEINTERVALS,&frameIntervalEnum));
}
//Frame intervals are available in fixed steps between the min and max, including the ends.
else if(frameIntervalEnum.type == V4L2_FRMIVAL_TYPE_STEPWISE)
{
    assert(frameIntervalEnum.stepwise.min.denominator != 0);
    assert(frameIntervalEnum.stepwise.max.denominator != 0);
    assert(frameIntervalEnum.stepwise.step.denominator != 0);

    const float stepMin = static_cast<float>(frameIntervalEnum.stepwise.min.numerator) /
                          static_cast<float>(frameIntervalEnum.stepwise.min.denominator);
    const float stepMax = static_cast<float>(frameIntervalEnum.stepwise.max.numerator) /
                          static_cast<float>(frameIntervalEnum.stepwise.max.denominator);
    const float stepIncr = static_cast<float>(frameIntervalEnum.stepwise.step.numerator) /
                           static_cast<float>(frameIntervalEnum.stepwise.step.denominator);

    for(float interval = stepMin;interval <= stepMax;interval += stepIncr)
    {
        //Make use of interval here.
    }
}
//Any frame interval between the min and max are okay, including the ends.
else if(frameIntervalEnum.type == V4L2_FRMIVAL_TYPE_CONTINUOUS)
{
    assert(frameIntervalEnum.stepwise.min.denominator != 0);
    const float intervalMin =
        static_cast<float>(frameIntervalEnum.stepwise.min.numerator) /
        static_cast<float>(frameIntervalEnum.stepwise.min.denominator);

    assert(frameIntervalEnum.stepwise.max.denominator != 0);
    const float intervalMax =
        static_cast<float>(frameIntervalEnum.stepwise.max.numerator) /
        static_cast<float>(frameIntervalEnum.stepwise.max.denominator);

    //Make use of intervals here.
}
{% endhighlight %}

### Selecting a Format ###

Setting the camera to use a particular format and size is done by filling out a `v4l2_format` struct and passing it to a `ioctl` call with `VIDIOC_S_FMT`. If you don't specify a supported format and size, the driver will select a different one and modify the `v4l2_format` struct to indicate what's being used instead.

{% highlight C++ %}
//Set the width, height, and pixel format that should be used. If a format is not
//supported, the driver will select something else and modify the v4l2_format
//structure to indicate the change.
v4l2_format format;
format.type = V4L2_BUF_TYPE_VIDEO_CAPTURE; //Must be set to V4L2_BUF_TYPE_VIDEO_CAPTURE.
format.fmt.pix.width = width;
format.fmt.pix.height = height;
format.fmt.pix.pixelformat = pixelFormat;
format.fmt.pix.field = V4L2_FIELD_NONE; //Non-interlaced.
format.fmt.pix.bytesperline = 0; //Depends on pixel format but can be larger to pad extra
                                 //bytes at the end of each line. Set to zero to get a
                                 //reasonable default.
if(ioctl(fd,VIDIOC_S_FMT,&format) == -1)
    return false;

//Driver sets the size of each image in bytes.
const unsigned int imageSizeInBytes = format.fmt.pix.sizeimage;
{% endhighlight %}

After you set the format and size, you can change the frame interval as well. If you change this first, it might be reset to a default value. You can also reset it to default by setting the numerator and denominator to zero.

{% highlight C++ %}
//Set the frame interval that should be used.
v4l2_streamparm streamParm;
memset(&streamParm,0,sizeof(streamParm));
streamParm.type = V4L2_BUF_TYPE_VIDEO_CAPTURE; //Must be set to V4L2_BUF_TYPE_VIDEO_CAPTURE.
streamParm.parm.capture.timeperframe.numerator = intervalNumerator;
streamParm.parm.capture.timeperframe.denominator = intervalDenominator;
if(ioctl(fd,VIDIOC_S_PARM,&streamParm) == -1)
    return false;
{% endhighlight %}

### Capturing a Video Frame ###

There are multiple ways to capture a frame of video using V4L2. The simplest method is to just read from the file descriptor like it's a file. Every `sizeimage` (returned by the `ioctl(fd,VIDIOC_S_FMT,...);` call) bytes read represents a single frame. Remember that the pixel format is probably not RGB so it usually needs to be converted before it can be used.

{% highlight C++ %}
#include <unistd.h>

...

//Simply read from the file descriptor into a buffer. Every `imageSizeInBytes` (defined
//above) read is one frame.
std::vector<unsigned char> buffer(imageSizeInBytes);
const ssize_t bytesRead = read(fd,&buffer[0],buffer.size());
if(bytesRead == -1)
    return false;
{% endhighlight %}

### Converting a Video Frame To RGB ###

Since the pixel format is not RGB and we need RGB, it obviously needs to be converted. The process varies by format, so I'm going to only go over one format here: [YUYV 4:2:2](https://www.kernel.org/doc/html/latest/media/uapi/v4l/pixfmt-yuyv.html).

The YUYV 4:2:2 format is a [Y'CbCr](https://en.wikipedia.org/wiki/YCbCr) interleaved format where every pixel has its own luminance (Y') channel and every two pixels share the Cb and Cr channels. Its layout is a repeating pattern that looks like this: \`[Y'_0, C_b, Y'_1, C_r, Y'_0, C_b, Y'_1, C_r,...]\`. Where \`Y'_0\` is luminance for the first pixel, \`C_b\` is the Cb channel, \`Y'_1\` is lumiance for the second pixel, and \`C_r\` is the Cr channel.

The actual [Y'CbCr to RGB conversion](https://en.wikipedia.org/wiki/YCbCr#JPEG_conversion) can be done by following the [JPEG](https://en.wikipedia.org/wiki/JPEG_File_Interchange_Format#Color_space) standard which is good enough because we are assuming all cameras are uncalibrated.

\`R = Y' + 1.402 * (Cr - 128)\`

\`G = Y' - 0.344 * (Cb - 128) - 0.714 * (Cr - 128)\`

\`B = Y' + 1.772 * (Cb - 128)\`

Since this format specifies two Y' for every pair of Cb and Cr, it's easiest to do the actual conversion to two pixels at a time.

{% highlight C++ %}
const unsigned int imageWidth; //Width of image.
const unsigned int imageHeight; //Height of image.
const std::vector<unsigned char> yuyvData; //Input YUYV buffer read from camera.
std::vector<unsigned char> rgbData(imageWidth * imageHeight * 3); //Output RGB buffer.

//Convenience function for clamping a signed 32-bit integer to an unsigned 8-bit integer.
auto ClampToU8 = [](const int value)
{
    return std::min(std::max(value,0),255);
};

//Convert from YUYV 4:2:2 to RGB using two pixels at a time.
for(unsigned int x = 0;x < imageWidth * imageHeight;x += 2)
{
    const unsigned int inputIndex = x * 2;
    const unsigned int outputIndex = x * 3;

    //Extract pixel's YCbCr components.
    const unsigned char y0 = yuyvData[inputIndex + 0];
    const unsigned char cb = yuyvData[inputIndex + 1];
    const unsigned char y1 = yuyvData[inputIndex + 2];
    const unsigned char cr = yuyvData[inputIndex + 3];

    //Convert first pixel.
    rgbData[outputIndex + 0] = ClampToU8(y0 + 1.402 * (cr - 128));
    rgbData[outputIndex + 1] = ClampToU8(y0 - 0.344 * (cb - 128) - 0.714 * (cr - 128));
    rgbData[outputIndex + 2] = ClampToU8(y0 + 1.772 * (cb - 128));

    //Convert second pixel.
    rgbData[outputIndex + 3] = ClampToU8(y1 + 1.402 * (cr - 128));
    rgbData[outputIndex + 4] = ClampToU8(y1 - 0.344 * (cb - 128) - 0.714 * (cr - 128));
    rgbData[outputIndex + 5] = ClampToU8(y1 + 1.772 * (cb - 128));
}
{% endhighlight %}

Now the image is ready to be displayed or processed further. Working with a single camera on a single platform isn't too much work. Expanding to support more, however, can be very time consuming. I highly recommend using a library that handles most of this for you. Next time, I'm going to ignore my own advice and show how to capture video frames from a camera on Windows.

### Resources ###

[V4L2 Documentation](https://www.kernel.org/doc/html/latest/media/uapi/v4l/v4l2.html)

### Footnotes ###

[^1]: The function definition is actually `int ioctl(int fd, unsigned long int request, ...);`. This indicates that the function takes two or more arguments but we only need the format that takes exactly three arguments.
[^2]: Frame interval is the time between successive frames. Frames per second is equal to 1 / frame interval.
