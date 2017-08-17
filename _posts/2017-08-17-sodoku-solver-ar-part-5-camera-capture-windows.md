---
layout: post
title: "Sudoku Solver AR Part 5: Camera Capture (Windows)"
date: 2017-08-17 10:15:00 -0500
---

_This post is part of a series where I explain how to build an **augmented reality Sudoku solver**. All of the image processing and computer vision is done from scratch. See the [first post]({% post_url 2017-07-20-sudoku-solver-ar-part-1-about %}) for a complete list of topics._ 

Video frames from a camera are captured on Windows (Vista and later) using the [Media Foundation](https://en.wikipedia.org/wiki/Media_Foundation) API. The API is designed to handle video/audio capture, processing, and rendering. And... it's really a pain to work with. The [documentation](https://msdn.microsoft.com/en-us/library/windows/desktop/ms694197(v=vs.85).aspx) contains so many words, yet requires jumping all over the place and never seems to explain what you want to know. Adding insult to injury, Media Foundation is a [COM](https://en.wikipedia.org/wiki/Component_Object_Model) based API, so you can expect lots of annoying reference counting.

Hopefully, I scared you into using a library for your own projects. Otherwise, just like with the Linux version of this post, I'm going to go over how to find all of the connected cameras, query their supported formats, capture video frames, and convert them to RGB images.

### List Connected Cameras ###

A list of connected cameras can be found using the [MFEnumDeviceSources](https://msdn.microsoft.com/en-us/library/windows/desktop/dd388503(v=vs.85).aspx) function. A pointer to an IMFAttributes must be provided to specify the type of devices that should be returned. To find a list of cameras, video devices should be specified. IMFAttributes is basically a key-value store used by Media Foundation. A new instance can be created by calling the [MFCreateAttributes](https://msdn.microsoft.com/en-us/library/windows/desktop/ms701878(v=vs.85).aspx) function.

After evaluating the available video devices, make sure to clean-up the unused devices by calling [Release](https://msdn.microsoft.com/en-us/library/windows/desktop/ms682317(v=vs.85).aspx) on each and [CoTaskMemFree](https://msdn.microsoft.com/en-us/library/windows/desktop/ms680722(v=vs.85).aspx) on the array of devices. This behavior of having to manually manage referencing counting everywhere within Media Foundation. It makes proper error handling incredibly tedious so I've only inserted `assert()` calls in the following code snippets for brevity.

{% highlight C++ %}
#include <mfapi.h>

...

//Create an instance of IMFAttributes. This class is basically a key-value store used by
//many Media Foundation functions to configure how they should behave.
IMFAttributes* attributes = nullptr;
HRESULT hr = MFCreateAttributes(&attributes,1);
assert(SUCCEEDED(hr));

//Configure the MFEnumDeviceSources call below to look for video capture devices only.
hr = attributes->SetGUID(MF_DEVSOURCE_ATTRIBUTE_SOURCE_TYPE,
                         MF_DEVSOURCE_ATTRIBUTE_SOURCE_TYPE_VIDCAP_GUID);
assert(SUCCEEDED(hr));

//Enumerate all of the connected video capture devices. Other devices besides cameras
//might be found like video capture cards.
UINT32 deviceCount = 0;
IMFActivate** devices = nullptr;
hr = MFEnumDeviceSources(attributes,&devices,&deviceCount);
assert(SUCCEEDED(hr));

//Evaluate which video devices should be used. The others should be freed by calling
//Release().
for(UINT32 x = 0;x < deviceCount;x++)
{
    IMFActivate* device = devices[x];

    //Use/store device here OR call Release() to free.

    device->Release();
}

//Clean up attributes and array of devices. Note that the devices themselves have not been
//cleaned up unless Release() is called on each one.
attributes->Release();
CoTaskMemFree(devices);
{% endhighlight %}

### Using a Camera ###

Before anything can be done with a camera, an [IMFMediaSource](https://msdn.microsoft.com/en-us/library/windows/desktop/ms700189(v=vs.85).aspx) must be fetched from the device. It's used for starting, pausing, or stopping capture on the device. None of which is necessary for this project since the device is expected to be always running. But, the IMFMediaSource is also used for creating an [IMFSourceReader](https://msdn.microsoft.com/en-us/library/windows/desktop/dd374655(v=vs.85).aspx) which is required for querying supported formats or making use of captured video frames.

{% highlight C++ %}
#include <mfidl.h>
#include <mfreadwrite.h>

...

IMFActivate* device; //Device selected above.

//Get a media source for the device to access the video stream.
IMFMediaSource* mediaSource = nullptr;
hr = device->ActivateObject(__uuidof(IMFMediaSource),
                            reinterpret_cast<void**>(&mediaSource));
assert(SUCCEEDED(hr));

//Create a source reader to actually read from the video stream. It's also used for
//finding out what formats the device supports delivering the video as.
IMFSourceReader* sourceReader = nullptr;
hr = MFCreateSourceReaderFromMediaSource(mediaSource,nullptr,&sourceReader);
assert(SUCCEEDED(hr));

...

//Clean-up much later when done querying the device or capturing video.
mediaSource->Release();
sourceReader->Release();
{% endhighlight %}

### Querying Supported Formats ###

Cameras capture video frames in different sizes, pixel formats, and rates. You can query a camera's supported formats to find the best fit for your use or just let the driver pick a sane default.

There's certainly an advantage to picking the formats yourself. For example, since cameras rarely provide an RGB image, if a camera uses a format that you already know how to convert to RGB, you can save time and use that instead. You can also choose to lower the frame rate for better quality at a cost of more motion blur or raise the frame rate for a more noisy image and less motion blur.

The supported formats are found by repeatedly calling the [GetNativeMediaType](https://msdn.microsoft.com/en-us/library/windows/desktop/dd374661(v=vs.85).aspx) method on the `IMFSourceReader` instance created above. After each call, the second parameter is incremented until the function returns an error indicating that there are no more supported formats. The actual format info is returned in the third parameter as an [IMFMediaType](https://msdn.microsoft.com/en-us/library/windows/desktop/ms704850(v=vs.85).aspx) instance. IMFMediaType inherits from the IMFAttributes class used earlier so it also behaves as a key-value store. The keys used to look up info about the current format are found [on this page](https://msdn.microsoft.com/en-us/library/windows/desktop/aa376629(v=vs.85).aspx).

{% highlight C++ %}
IMFSourceReader* sourceReader; //Source reader created above.

//Iterate over formats directly supported by the video device.
IMFMediaType* mediaType = nullptr;
for(DWORD x = 0;
    sourceReader->GetNativeMediaType(MF_SOURCE_READER_FIRST_VIDEO_STREAM,
                                     x,
                                     &mediaType) == S_OK;
    x++)
{
    //Query video pixel format. The list of possible video formats can be found at:
    //https://msdn.microsoft.com/en-us/library/windows/desktop/aa370819(v=vs.85).aspx
    GUID pixelFormat;
    hr = mediaType->GetGUID(MF_MT_SUBTYPE,&pixelFormat);
    assert(SUCCEEDED(hr));

    //Query video frame dimensions.
    UINT32 frameWidth = 0;
    UINT32 frameHeight = 0;
    hr = MFGetAttributeSize(mediaType,MF_MT_FRAME_SIZE,&frameWidth,&frameHeight);
    assert(SUCCEEDED(hr));

    //Query video frame rate.
    UINT32 frameRateNumerator = 0;
    UINT32 frameRateDenominator = 0;
    hr = MFGetAttributeRatio(mediaType,
                             MF_MT_FRAME_RATE,
                             &frameRateNumerator,
                             &frameRateDenominator);
    assert(SUCCEEDED(hr));

    //Make use of format, size, and frame rate here or hold onto mediaType as is (don't
    //call Release() next) for configuring the camera (see below).

    //Clean-up mediaType when done with it.
    mediaType->Release();
}
{% endhighlight %}

### Selecting a Format ###

There's not a lot to setting the device to use a particular format. Just call the [SetCurrentMediaType](https://msdn.microsoft.com/en-us/library/windows/desktop/dd374667(v=vs.85).aspx) method on the `IMFSourceReader` instance and pass along one of the IMFMediaType queried above.

{% highlight C++ %}
IMFSourceReader* sourceReader; //Source reader created above.
IMFMediaType* mediaType; //Media type selected above.

//Set source reader to use the format selected from the list of supported formats above.
hr = sourceReader->SetCurrentMediaType(MF_SOURCE_READER_FIRST_VIDEO_STREAM,
                                       nullptr,
                                       mediaType);
assert(SUCCEEDED(hr));

//Clean-up mediaType since it won't be used anymore.
mediaType->Release();
{% endhighlight %}

### Capturing a Video Frame ###

There's a couple of ways to capture a video frame using Media Foundation. The method described here is the synchronous approach which is the simpler of the two. Basically, the process involves asking for a frame of video whenever we want a one and the thread then blocks until a new frame is available. If frames are not requested fast enough, they get dropped and a gap is indicated.

This is done by calling the [ReadSample](https://msdn.microsoft.com/en-us/library/windows/desktop/dd374665(v=vs.85).aspx) method on the `IMFSourceReader` instance. This function returns an [IMFSample](https://msdn.microsoft.com/en-us/library/windows/desktop/ms702192(v=vs.85).aspx) instance or a `nullptr` if a gap occurred. The IMFSample is a container that stores various information about a frame including the actual data in the pixel format selected above.

Accessing the pixel data involves calling the [GetBufferByIndex](https://msdn.microsoft.com/en-us/library/windows/desktop/ms697014(v=vs.85).aspx) method of the `IMFSample` and calling [Lock](https://msdn.microsoft.com/en-us/library/windows/desktop/bb970366(v=vs.85).aspx) on the resulting [IMFMediaBuffer](https://msdn.microsoft.com/en-us/library/windows/desktop/ms696261(v=vs.85).aspx) instance. Locking the buffer prevents the frame data from being modified while you're processing it. For example, the operating system might want to re-use the buffer for frames in the future but writing to it at the same tile as it's being read will garble the image.

Once done working with the frame data, don't forget to call [Unlock](https://msdn.microsoft.com/en-us/library/windows/desktop/ms696259(v=vs.85).aspx) on it and clean-up IMFSample and IMFMediaBuffer in preparation for future frames.

{% highlight C++ %}
IMFSourceReader* sourceReader; //Source reader created above.

//Grab the next frame. A loop is used because ReadSample returns a null sample when there
//is a gap in the stream.
IMFSample* sample = nullptr;
while(sample == nullptr)
{
    DWORD streamFlags = 0; //Most be passed to ReadSample or it'll fail.
    hr = sourceReader->ReadSample(MF_SOURCE_READER_FIRST_VIDEO_STREAM,
                                  0,
                                  nullptr,
                                  &streamFlags,
                                  nullptr,
                                  &sample);
    assert(SUCCEEDED(hr));
}

//Get access to the underlying memory for the frame.
IMFMediaBuffer* buffer = nullptr;
hr = sample->GetBufferByIndex(0,&buffer);
assert(SUCCEEDED(hr));

//Begin using the frame's data.
BYTE* bufferData = nullptr;
DWORD bufferDataLength = 0;
hr = buffer->Lock(&bufferData,nullptr,&bufferDataLength);
assert(SUCCEEDED(hr));

//Copy buffer data or convert directly to RGB here. The frame format, width, and height
//match the media type set above.

//Stop using the frame's data.
buffer->Unlock();

//Clean-up sample and buffer since frame is no longer being used.
sample->Release();
buffer->Release();
{% endhighlight %}

### Converting a Video Frame To RGB ###

The selected pixel format probably cannot be used directly. For our use, it needs to be converted to RGB[^1]. The conversion process varies by format. In the last post I covered the [YUYV 4:2:2](https://www.kernel.org/doc/html/latest/media/uapi/v4l/pixfmt-yuyv.html) format. In this one, I'm going to go over the similar [NV12](https://msdn.microsoft.com/en-us/library/windows/desktop/dd206750(v=vs.85).aspx#nv12) format.

The NV12 format is a [Y'CbCr](https://en.wikipedia.org/wiki/YCbCr) format that's split into two chunks. The first chunk is the luminance (Y') channel which contains an entry for each pixel. The second chunk interleaves the Cb and Cr channels together. There are only one Cb and Cr pair for every 2x2 region of pixels.

Just like with the YUYV 4:2:2 format, the [Y'CbCr to RGB conversion](https://en.wikipedia.org/wiki/YCbCr#JPEG_conversion) can be done by following the [JPEG](https://en.wikipedia.org/wiki/JPEG_File_Interchange_Format#Color_space) standard. This is a sufficient approach because we are assuming the camera's color space information is unavailable.

\`R = Y' + 1.402 * (Cr - 128)\`

\`G = Y' - 0.344 * (Cb - 128) - 0.714 * (Cr - 128)\`

\`B = Y' + 1.772 * (Cb - 128)\`

{% highlight C++ %}
const unsigned int imageWidth; //Width of image.
const unsigned int imageHeight; //Height of image.
const std::vector<unsigned char> nv12Data; //Input NV12 buffer read from camera.
std::vector<unsigned char> rgbData(imageWidth * imageHeight * 3); //Output RGB buffer.

//Convenience function for clamping a signed 32-bit integer to an unsigned 8-bit integer.
auto ClampToU8 = [](const int value)
{
    return std::min(std::max(value,0),255);
};

const unsigned int widthHalf = imageWidth / 2;

//Convert from NV12 to RGB.
for(unsigned int y = 0;y < imageHeight;y++)
{
    for(unsigned int x = 0;x < imageWidth;x++)
    {
        //Clear the lowest bit for both the x and y coordinates so they are always even.
        const unsigned int xEven = x & 0xFFFFFFFE;
        const unsigned int yEven = y & 0xFFFFFFFE;

        //The Y chunk is at the beginning of the frame and easily indexed.
        const unsigned int yIndex = y * imageWidth + x;

        //The Cb and Cr channels are interleaved in their own chunk after the Y chunk.
        //This chunk is 1/4th the size of the first chunk. There is one Cb and one Cr
        //component for every 2x2 region of pixels.
        const unsigned int cIndex = imageWidth * imageHeight + yEven * widthHalf + xEven;

        //Extract YCbCr components.
        const unsigned char y = nv12Data[yIndex];
        const unsigned char cb = nv12Data[cIndex + 0];
        const unsigned char cr = nv12Data[cIndex + 1];

        //Convert from YCbCr to RGB.
        const unsigned int outputIndex = (y * imageWidth + x) * 3;
        rgbData[outputIndex + 0] = ClampToU8(y + 1.402 * (cr - 128));
        rgbData[outputIndex + 1] = ClampToU8(y - 0.344 * (cb - 128) - 0.714 * (cr - 128));
        rgbData[outputIndex + 2] = ClampToU8(y + 1.772 * (cb - 128));
    }
}
{% endhighlight %}

That's it for video capture, now the image is ready to be processed. I'm going to repeat what I said in the opening, go find a library to handle all of this for you. Save yourself the hassle. Especially if you decide to support more pixel formats or platforms in the future. Next up will be the Canny edge detector which is used to find edges in an image.

### Footnotes ###

[^1]: I have a camera that claims to produce RGB images but actually gives BGR (red and blue are switched) images with the rows in bottom-to-top order. Due to the age of the camera, the hardware is probably using the [.bmp](https://en.wikipedia.org/wiki/BMP_file_format) format which is then directly unpacked by the driver. Anyway, expect some type of conversion to always be necessary.
