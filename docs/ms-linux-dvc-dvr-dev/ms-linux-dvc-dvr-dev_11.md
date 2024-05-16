# 第九章：从用户空间利用 V4L2 API

设备驱动程序的主要目的是控制和利用底层硬件，同时向用户公开功能。这些用户可能是在用户空间运行的应用程序或其他内核驱动程序。前两章涉及 V4L2 设备驱动程序，而在本章中，我们将学习如何利用内核公开的 V4L2 设备功能。我们将首先描述和枚举用户空间 V4L2 API，然后学习如何利用这些 API 从传感器中获取视频数据，包括篡改传感器属性。

本章将涵盖以下主题：

+   V4L2 用户空间 API

+   视频设备属性管理从用户空间

+   用户空间的缓冲区管理

+   V4L2 用户空间工具

# 技术要求

为了充分利用本章，您将需要以下内容：

+   高级计算机体系结构知识和 C 编程技能

+   Linux 内核 v4.19.X 源代码，可在[`git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/refs/tags`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/refs/tags)获取

# 从用户空间介绍 V4L2

编写设备驱动程序的主要目的是简化应用程序对底层设备的控制和使用。用户空间处理 V4L2 设备有两种方式：一种是使用诸如`GStreamer`及其`gst-*`工具之类的一体化工具，另一种是使用用户空间 V4L2 API 编写专用应用程序。在本章中，我们只涉及代码，因此我们将介绍如何编写使用 V4L2 API 的应用程序。

## V4L2 用户空间 API

V4L2 用户空间 API 具有较少的功能和大量的数据结构，所有这些都在`include/uapi/linux/videodev2.h`中定义。在本节中，我们将尝试描述其中最重要的，或者更确切地说，最常用的。您的代码应包括以下标头：

```
#include <linux/videodev2.h>
```

此 API 依赖以下功能：

+   `open()`: 打开视频设备

+   `close()`: 关闭视频设备

+   `ioctl()`: 向显示驱动程序发送 ioctl 命令

+   `mmap()`: 将驱动程序分配的缓冲区内存映射到用户空间

+   `read()`或`write()`，取决于流方法

这个减少的 API 集合由大量的 ioctl 命令扩展，其中最重要的是：

+   `VIDIOC_QUERYCAP`: 用于查询驱动程序的功能。人们过去常说它用于查询设备的功能，但这并不正确，因为设备可能具有驱动程序中未实现的功能。用户空间传递一个`struct v4l2_capability`结构，该结构将由视频驱动程序填充相关信息。

+   `VIDIOC_ENUM_FMT`: 用于枚举驱动程序支持的图像格式。驱动程序用户空间传递一个`struct v4l2_fmtdesc`结构，该结构将由驱动程序填充相关信息。

+   `VIDIOC_G_FMT`: 对于捕获设备，用于获取当前图像格式。但是，对于显示设备，您可以使用此功能获取当前显示窗口。在任何情况下，用户空间传递一个`struct v4l2_format`结构，该结构将由驱动程序填充相关信息。

+   `VIDIOC_TRY_FMT`应在不确定要提交给设备的格式时使用。这用于验证捕获设备的新图像格式或根据输出（显示）设备使用新的显示窗口。用户空间传递一个带有它想要应用的属性的`struct v4l2_format`结构，如果它们不受支持，驱动程序可能会更改给定的值。然后应用程序应检查授予了什么。

+   `VIDIOC_S_FMT`用于为捕获设备设置新的图像格式或为显示（输出设备）设置新的显示窗口。如果不首先使用`VIDIOC_TRY_FMT`，驱动程序可能会更改用户空间传递的值，如果它们不受支持。应用程序应检查是否授予了什么。

+   `VIDIOC_CROPCAP` 用于根据当前图像大小和当前显示面板大小获取默认裁剪矩形。驱动程序填充一个 `struct v4l2_cropcap` 结构。

+   `VIDIOC_G_CROP` 用于获取当前裁剪矩形。驱动程序填充一个 `struct v4l2_crop` 结构。

+   `VIDIOC_S_CROP` 用于设置新的裁剪矩形。驱动程序填充一个 `struct v4l2_crop` 结构。应用程序应该检查授予了什么。

+   `VIDIOC_REQBUFS`：这个 ioctl 用于请求一定数量的缓冲区，以便稍后进行内存映射。驱动程序填充一个 `struct v4l2_requestbuffers` 结构。由于驱动程序可能分配的缓冲区数量多于或少于实际请求的数量，应用程序应该检查实际授予了多少个缓冲区。在此之后还没有排队任何缓冲区。

+   `VIDIOC_QUERYBUF` ioctl 用于获取缓冲区的信息，这些信息可以被 `mmap()` 系统调用用来将缓冲区映射到用户空间。驱动程序填充一个 `struct v4l2_buffer` 结构。

+   `VIDIOC_QBUF` 用于通过传递与该缓冲区相关联的 `struct v4l2_buffer` 结构来排队一个缓冲区。在这个 ioctl 的执行路径上，驱动程序将把这个缓冲区添加到其缓冲区列表中，以便在没有更多待处理的排队缓冲区之前填充它。一旦缓冲区被填充，它就会传递给 V4L2 核心，它维护自己的列表（即准备好的缓冲区列表），并且它会从驱动程序的 DMA 缓冲区列表中移除。

+   `VIDIOC_DQBUF` 用于从 V4L2 的准备好的缓冲区列表（对于输入设备）或显示的（输出设备）缓冲区中出列一个已填充的缓冲区，通过传递与该缓冲区相关联的 `struct v4l2_buffer` 结构。如果没有准备好的缓冲区，它会阻塞，除非在 `open()` 中使用了 `O_NONBLOCK`，在这种情况下，`VIDIOC_DQBUF` 会立即返回一个 `EAGAIN` 错误代码。只有在调用了 `STREAMON` 之后才应该调用 `VIDIOC_DQBUF`。与此同时，在 `STREAMOFF` 之后调用这个 ioctl 会返回 `-EINVAL`。

+   `VIDIOC_STREAMON` 用于开启流。之后，任何 `VIDIOC_QBUF` 的结果都会呈现图像。

+   `VIDIOC_STREAMOFF` 用于关闭流。这个 ioctl 移除所有缓冲区。它实际上刷新了缓冲队列。

有很多 ioctl 命令，不仅仅是我们刚刚列举的那些。实际上，内核的 `v4l2_ioctl_ops` 数据结构中至少有和操作一样多的 ioctl。然而，上述的 ioctl 已经足够深入了解 V4L2 用户空间 API。在本节中，我们不会详细介绍每个数据结构。因此，你应该保持 `include/uapi/linux/videodev2.h` 文件的打开状态，也可以在 [`elixir.bootlin.com/linux/v4.19/source/include/uapi/linux/videodev2.h`](https://elixir.bootlin.com/linux/v4.19/source/include/uapi/linux/videodev2.h) 找到，因为它包含了所有的 V4L2 API 和数据结构。话虽如此，以下伪代码展示了使用 V4L2 API 从用户空间抓取视频的典型 ioctl 序列：

```
open()
int ioctl(int fd, VIDIOC_QUERYCAP,           struct v4l2_capability *argp)
int ioctl(int fd, VIDIOC_S_FMT, struct v4l2_format *argp)
int ioctl(int fd, VIDIOC_S_FMT, struct v4l2_format *argp)
/* requesting N buffers */
int ioctl(int fd, VIDIOC_REQBUFS,           struct v4l2_requestbuffers *argp)
/* queueing N buffers */
int ioctl(int fd, VIDIOC_QBUF, struct v4l2_buffer *argp)
/* start streaming */
int ioctl(int fd, VIDIOC_STREAMON, const int *argp) 
read_loop: (for i=0; I < N; i++)
    /* Dequeue buffer i */
    int ioctl(int fd, VIDIOC_DQBUF, struct v4l2_buffer *argp)
    process_buffer(i)
    /* Requeue buffer i */
    int ioctl(int fd, VIDIOC_QBUF, struct v4l2_buffer *argp)
end_loop
    releases_memories()
    close()
```

上述序列将作为指南来处理用户空间中的 V4L2 API。

请注意，`ioctl` 系统调用可能返回 `-1` 值，而 `errno = EINTR`。在这种情况下，这并不意味着错误，而只是系统调用被中断，此时应该再次尝试。为了解决这个（虽然可能性很小但是可能发生的）问题，我们可以考虑编写自己的 `ioctl` 包装器，例如以下内容：

```
static int xioctl(int fh, int request, void *arg)
{
        int r;
        do {
                r = ioctl(fh, request, arg);
        } while (-1 == r && EINTR == errno);
        return r;
}
```

现在我们已经完成了视频抓取序列的概述，我们可以弄清楚从设备打开到关闭的视频流程需要哪些步骤，包括格式协商。现在我们可以跳转到代码，从设备打开开始，一切都从这里开始。

# 视频设备的打开和属性管理

驱动程序在`/dev/`目录中公开节点条目，对应于它们负责的视频接口。这些文件节点对应于捕获设备的`/dev/videoX`特殊文件（在我们的情况下）。应用程序必须在与视频设备的任何交互之前打开适当的文件节点。它使用`open()`系统调用来打开，这将返回一个文件描述符，将成为发送到设备的任何命令的入口点，如下例所示：

```
static const char *dev_name = "/dev/video0";
fd = open (dev_name, O_RDWR);
if (fd == -1) {
    perror("Failed to open capture device\n");
    return -1;
}
```

前面的片段是以阻塞模式打开的。将`O_NONBLOCK`传递给`open()`将防止应用程序在尝试出队时没有准备好的缓冲区时被阻塞。完成视频设备的使用后，应使用`close()`系统调用关闭它：

```
close (fd);
```

在我们能够打开视频设备之后，我们可以开始与其进行交互。通常，视频设备打开后发生的第一个动作是查询其功能，通过这个功能，我们可以使其以最佳方式运行。

## 查询设备功能

通常查询设备的功能以确保它支持我们需要处理的模式是很常见的。您可以使用`VIDIOC_QUERYCAP` ioctl 命令来执行此操作。为此，应用程序传递一个`struct v4l2_capability`结构（在`include/uapi/linux/videodev2.h`中定义），该结构将由驱动程序填充。该结构具有一个`.capabilities`字段需要进行检查。该字段包含整个设备的功能。内核源代码的以下摘录显示了可能的值：

```
/* Values for 'capabilities' field */
#define V4L2_CAP_VIDEO_CAPTURE 0x00000001 /*video capture device*/ #define V4L2_CAP_VIDEO_OUTPUT 0x00000002  /*video output device*/ #define V4L2_CAP_VIDEO_OVERLAY 0x00000004 /*Can do video overlay*/ [...] /* VBI device skipped */
/* video capture device that supports multiplanar formats */#define V4L2_CAP_VIDEO_CAPTURE_MPLANE	0x00001000
/* video output device that supports multiplanar formats */ #define V4L2_CAP_VIDEO_OUTPUT_MPLANE	0x00002000
/* mem-to-mem device that supports multiplanar formats */#define V4L2_CAP_VIDEO_M2M_MPLANE	0x00004000
/* Is a video mem-to-mem device */#define V4L2_CAP_VIDEO_M2M	0x00008000
[...] /* radio, tunner and sdr devices skipped */
#define V4L2_CAP_READWRITE	0x01000000 /*read/write systemcalls */ #define V4L2_CAP_ASYNCIO	0x02000000	/* async I/O */
#define V4L2_CAP_STREAMING	0x04000000	/* streaming I/O ioctls */ #define V4L2_CAP_TOUCH	0x10000000	/* Is a touch device */
```

以下代码块显示了一个常见用例，展示了如何使用`VIDIOC_QUERYCAP` ioctl 从代码中查询设备功能：

```
#include <linux/videodev2.h>
[...]
struct v4l2_capability cap;
memset(&cap, 0, sizeof(cap));
if (-1 == xioctl(fd, VIDIOC_QUERYCAP, &cap)) {
    if (EINVAL == errno) {
        fprintf(stderr, "%s is no V4L2 device\n", dev_name);
        exit(EXIT_FAILURE);
    } else {
        errno_exit("VIDIOC_QUERYCAP" 
    }
}
```

在前面的代码中，`struct v4l2_capability`在传递给`ioctl`命令之前首先通过`memset()`清零。在这一步，如果没有错误发生，那么我们的`cap`变量现在包含了设备的功能。您可以使用以下内容来检查设备类型和 I/O 方法：

```
if (!(cap.capabilities & V4L2_CAP_VIDEO_CAPTURE)) {
    fprintf(stderr, "%s is not a video capture device\n",             dev_name);
    exit(EXIT_FAILURE);
}
if (!(cap.capabilities & V4L2_CAP_READWRITE))
    fprintf(stderr, "%s does not support read i/o\n",             dev_name);
/* Check whether USERPTR and/or MMAP method are supported */
if (!(cap.capabilities & V4L2_CAP_STREAMING))
    fprintf(stderr, "%s does not support streaming i/o\n",             dev_name);
/* Check whether driver support read/write i/o */
if (!(cap.capabilities & V4L2_CAP_READWRITE))
    fprintf (stderr, "%s does not support read i/o\n",              dev_name);
```

您可能已经注意到，在使用之前，我们首先将`cap`变量清零。在给 V4L2 API 提供参数时，清除参数是一个好的做法，以避免陈旧的内容。然后，让我们定义一个宏——比如`CLEAR`——它将清零作为参数给定的任何变量，并在本章的其余部分中使用它：

```
#define CLEAR(x) memset(&(x), 0, sizeof(x))
```

现在，我们已经完成了查询视频设备功能。这使我们能够配置设备并根据我们需要实现的内容调整图像格式。通过协商适当的图像格式，我们可以利用视频设备，正如我们将在下一节中看到的那样。

# 缓冲区管理

在 V4L2 中，维护两个缓冲队列：一个用于驱动程序（称为`VIDIOC_QBUF` ioctl）。缓冲区按照它们被入队的顺序由驱动程序填充。一旦填充，每个缓冲区就会从输入队列移出，并放入输出队列，即用户队列。

每当用户应用程序调用`VIDIOC_DQBUF`以出队一个缓冲区时，该缓冲区将在输出队列中查找。如果在那里，缓冲区将被出队并*推送*到用户应用程序；否则，应用程序将等待直到有填充的缓冲区。用户完成使用缓冲区后，必须调用`VIDIOC_QBUF`将该缓冲区重新入队到输入队列中，以便可以再次填充。

驱动程序初始化后，应用程序调用`VIDIOC_REQBUFS` ioctl 来设置它需要处理的缓冲区数量。一旦获准，应用程序使用`VIDIOC_QBUF`队列中的所有缓冲区，然后调用`VIDIOC_STREAMON` ioctl。然后，驱动程序自行填充所有排队的缓冲区。如果没有更多排队的缓冲区，那么驱动程序将等待应用程序入队缓冲区。如果出现这种情况，那么这意味着在捕获本身中丢失了一些帧。

## 图像（缓冲区）格式

在确保设备是正确类型并支持其可以使用的模式之后，应用程序必须协商其需要的视频格式。应用程序必须确保视频设备配置为以应用程序可以处理的格式发送视频帧。在开始抓取和收集数据（或视频帧）之前，必须这样做。V4L2 API 使用`struct v4l2_format`来表示缓冲区格式，无论设备类型是什么。该结构定义如下：

```
struct v4l2_format {
 u32 type;
 union {
  struct v4l2_pix_format pix; /* V4L2_BUF_TYPE_VIDEO_CAPTURE */    
  struct v4l2_pix_format_mplane pix_mp; /* _CAPTURE_MPLANE */
  struct v4l2_window win;	 /* V4L2_BUF_TYPE_VIDEO_OVERLAY */
  struct v4l2_vbi_format vbi; /* V4L2_BUF_TYPE_VBI_CAPTURE */
  struct v4l2_sliced_vbi_format sliced;/*_SLICED_VBI_CAPTURE */ 
  struct v4l2_sdr_format sdr;   /* V4L2_BUF_TYPE_SDR_CAPTURE */
  struct v4l2_meta_format meta;/* V4L2_BUF_TYPE_META_CAPTURE */
        [...]
    } fmt;
};
```

在前面的结构中，`type`字段表示数据流的类型，并应由应用程序设置。根据其值，`fmt`字段将是适当的类型。在我们的情况下，`type`必须是`V4L2_BUF_TYPE_VIDEO_CAPTURE`，因为我们正在处理视频捕获设备。然后，`fmt`将是`struct v4l2_pix_format`类型。

重要说明

几乎所有（如果不是全部）直接或间接与缓冲区播放的 ioctl（如裁剪、缓冲区请求/排队/出队/查询）都需要指定缓冲区类型，这是有道理的。我们将使用`V4L2_BUF_TYPE_VIDEO_CAPTURE`，因为这是我们设备类型的唯一选择。缓冲区类型的整个列表是在`include/uapi/linux/videodev2.h`中定义的`enum v4l2_buf_type`类型。你应该看一看。

应用程序通常会查询视频设备的当前格式，然后仅更改其中感兴趣的属性，并将新的混合缓冲区格式发送回视频设备。但这并不是强制性的。我们只是在这里做了这个演示，以演示您如何获取或设置当前格式。应用程序使用`VIDIOC_G_FMT` ioctl 命令查询当前缓冲区格式。它必须传递一个新的（我指的是清零的）`struct v4l2_format`结构，并设置`type`字段。驱动程序将在 ioctl 的返回路径中填充其余部分。以下是一个例子：

```
struct v4l2_format fmt;
CLEAR(fmt);
/* Get the current format */
fmt.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
if (ioctl(fd, VIDIOC_G_FMT, &fmt)) {
    printf("Getting format failed\n");
    exit(2);
}
```

一旦我们有了当前的格式，我们就可以更改相关属性，并将新格式发送回设备。这些属性可能是像素格式，每个颜色分量的内存组织，以及每个字段的交错捕获内存组织。我们还可以描述缓冲区的大小和间距。设备支持的常见（但不是唯一的）像素格式如下：

+   `V4L2_PIX_FMT_YUYV`：YUV422（交错）

+   `V4L2_PIX_FMT_NV12`：YUV420（半平面）

+   `V4L2_PIX_FMT_NV16`：YUV422（半平面）

+   `V4L2_PIX_FMT_RGB24`：RGB888（打包）

现在，让我们编写改变我们需要的属性的代码片段。但是，将新格式发送到视频设备需要使用新的 ioctl 命令，即`VIDIOC_S_FMT`：

```
#define WIDTH	1920
#define HEIGHT	1080
#define PIXFMT	V4L2_PIX_FMT_YUV420
/* Changing required properties and set the format */ fmt.fmt.pix.width = WIDTH;
fmt.fmt.pix.height = HEIGHT;
fmt.fmt.pix.bytesperline = fmt.fmt.pix.width * 2u;
fmt.fmt.pix.sizeimage = fmt.fmt.pix.bytesperline * fmt.fmt.pix.height; 
fmt.fmt.pix.colorspace = V4L2_COLORSPACE_REC709;
fmt.fmt.pix.field = V4L2_FIELD_ANY;
fmt.fmt.pix.pixelformat = PIXFMT;
fmt.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
if (xioctl(fd, VIDIOC_S_FMT, &fmt)) {
    printf("Setting format failed\n");
    exit(2);
}
```

重要说明

我们可以使用前面的代码，而不需要当前格式。

ioctl 可能会成功。但是，这并不意味着您的参数已经被应用。默认情况下，设备可能不支持每种图像宽度和高度的组合，甚至不支持所需的像素格式。在这种情况下，驱动程序将应用其支持的最接近的值，以符合您请求的值。然后，您需要检查您的参数是否被接受，或者被授予的参数是否足够好，以便您继续进行：

```
if (fmt.fmt.pix.pixelformat != PIXFMT)
   printf("Driver didn't accept our format. Can't proceed.\n");
/* because VIDIOC_S_FMT may change width and height */
if ((fmt.fmt.pix.width != WIDTH) ||     (fmt.fmt.pix.height != HEIGHT))     
 fprintf(stderr, "Warning: driver is sending image at %dx%d\n",
            fmt.fmt.pix.width, fmt.fmt.pix.height);
```

我们甚至可以进一步改变流参数，例如每秒帧数。我们可以通过以下方式实现这一点：

+   使用`VIDIOC_G_PARM` ioctl 查询视频设备的流参数。此 ioctl 接受一个新的`struct v4l2_streamparm`结构作为参数，并设置其`type`成员。此类型应该是`enum v4l2_buf_type`值之一。

+   检查`v4l2_streamparm.parm.capture.capability`，并确保设置了`V4L2_CAP_TIMEPERFRAME`标志。这意味着驱动程序允许更改捕获帧速率。

如果是这样，我们可以（可选地）使用`VIDIOC_ENUM_FRAMEINTERVALS` ioctl 来获取可能的帧间隔列表（API 使用帧间隔，这是帧速率的倒数）。

+   使用`VIDIOC_S_PARM` ioctl 并填写`v4l2_streamparm.parm.capture.timeperframe`成员的适当值。这应该允许设置捕获端的帧速率。您的任务是确保您读取得足够快，以免出现帧丢失。

以下是一个例子：

```
#define FRAMERATE 30
struct v4l2_streamparm parm;
int error;
CLEAR(parm);
parm.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
/* first query streaming parameters */
error = xioctl(fd, VIDIOC_G_PARM, &parm);
if (!error) {
    /* Now determine if the FPS selection is supported */
    if (parm.parm.capture.capability & V4L2_CAP_TIMEPERFRAME) {
        /* yes we can */
        CLEAR(parm);
        parm.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
        parm.parm.capture.capturemode = 0;
        parm.parm.capture.timeperframe.numerator = 1;
        parm.parm.capture.timeperframe.denominator = FRAMERATE;
        error = xioctl(fd, VIDIOC_S_PARM, &parm);
        if (error)
            printf("Unable to set the FPS\n");
        else
           /* once again, driver may have changed our requested 
            * framerate */
            if (FRAMERATE != 
                  parm.parm.capture.timeperframe.denominator)
                printf ("fps coerced ......: from %d to %d\n",
                        FRAMERATE,
                   parm.parm.capture.timeperframe.denominator);
```

现在，我们可以协商图像格式并设置流参数。下一个逻辑延续将是请求缓冲区并继续进行进一步处理。

## 请求缓冲区

完成格式准备后，现在是指示驱动程序分配用于存储视频帧的内存的时候了。`VIDIOC_REQBUFS` ioctl 就是为了实现这一点。此 ioctl 将新的`struct v4l2_requestbuffers`结构作为参数。在传递给 ioctl 之前，`v4l2_requestbuffers`必须设置其一些字段：

+   `v4l2_requestbuffers.count`：此成员应设置为要分配的内存缓冲区的数量。此成员应设置为确保帧不会因输入队列中排队的缓冲区不足而丢失的值。大多数情况下，`3`或`4`是正确的值。因此，驱动程序可能不满意请求的缓冲区数量。在这种情况下，驱动程序将在 ioctl 的返回路径上使用授予的缓冲区数量设置`v4l2_requestbuffers.count`。然后，应用程序应检查此值，以确保此授予的值符合其需求。

+   `v4l2_requestbuffers.type`：这必须使用`enum 4l2_buf_type`类型的视频缓冲区类型进行设置。在这里，我们再次使用`V4L2_BUF_TYPE_VIDEO_CAPTURE`。例如，对于输出设备，这将是`V4L2_BUF_TYPE_VIDEO_OUTPUT`。

+   `v4l2_requestbuffers.memory`：这必须是可能的`enum v4l2_memory`值之一。感兴趣的可能值是`V4L2_MEMORY_MMAP`，`V4L2_MEMORY_USERPTR`和`V4L2_MEMORY_DMABUF`。这些都是流式传输方法。但是，根据此成员的值，应用程序可能需要执行其他任务。不幸的是，`VIDIOC_REQBUFS`命令是应用程序发现给定驱动程序支持哪些类型的流式 I/O 缓冲区的唯一方法。然后，应用程序可以尝试使用这些值中的每一个`VIDIOC_REQBUFS`，并根据失败或成功的情况调整其逻辑。

### 请求用户指针缓冲区 - VIDIOC_REQBUFS 和 malloc

这一步涉及驱动程序支持流式传输模式，特别是用户指针 I/O 模式。在这里，应用程序通知驱动程序即将分配一定数量的缓冲区：

```
#define BUF_COUNT 4
struct v4l2_requestbuffers req; CLEAR (req);
req.count	= BUF_COUNT;
req.type	= V4L2_BUF_TYPE_VIDEO_CAPTURE;
req.memory	= V4L2_MEMORY_USERPTR;
if (-1 == xioctl (fd, VIDIOC_REQBUFS, &req)) {
    if (EINVAL == errno)
        fprintf(stderr,                 "%s does not support user pointer i/o\n", 
                dev_name);
    else
        fprintf("VIDIOC_REQBUFS failed \n");
}
```

然后，应用程序从用户空间分配缓冲区内存：

```
struct buffer_addr {
    void  *start;
    size_t length;
};
struct buffer_addr *buf_addr;
int i;
buf_addr = calloc(BUF_COUNT, sizeof (*buffer_addr));
if (!buf_addr) {
    fprintf(stderr, "Out of memory\n");
    exit (EXIT_FAILURE);
}
for (i = 0; i < BUF_COUNT; ++i) {
    buf_addr[i].length = buffer_size;
    buf_addr[i].start = malloc(buffer_size);
    if (!buf_addr[i].start) {
        fprintf(stderr, "Out of memory\n");
        exit(EXIT_FAILURE);
    }
}
```

这是第一种流式传输，其中缓冲区在用户空间中分配并交给内核以便填充视频数据：所谓的用户指针 I/O 模式。还有另一种花哨的流式传输模式，几乎所有操作都是从内核完成的。让我们立刻介绍它。

### 请求内存可映射缓冲区 - VIDIOC_REQBUFS，VIDIOC_QUERYBUF 和 mmap

在驱动程序缓冲区模式中，此 ioctl 还返回`v4l2_requestbuffer`结构的`count`成员中分配的实际缓冲区数量。此流式传输方法还需要一个新的数据结构`struct v4l2_buffer`。在内核中由驱动程序分配缓冲区后，此结构与`VIDIOC_QUERYBUFS` ioctl 一起使用，以查询每个分配的缓冲区的物理地址，该地址可与`mmap()`系统调用一起使用。驱动程序返回的物理地址将存储在`buffer.m.offset`中。

以下代码摘录指示驱动程序分配内存缓冲区并检查授予的缓冲区数量：

```
#define BUF_COUNT_MIN 3
struct v4l2_requestbuffers req; CLEAR (req);
req.count	= BUF_COUNT;
req.type	= V4L2_BUF_TYPE_VIDEO_CAPTURE;
req.memory	= V4L2_MEMORY_MMAP;
if (-1 == xioctl (fd, VIDIOC_REQBUFS, &req)) {
    if (EINVAL == errno)
        fprintf(stderr, "%s does not support memory mapping\n", 
                dev_name);
    else
        fprintf("VIDIOC_REQBUFS failed \n");
}
/* driver may have granted less than the number of buffers we
 * requested let's then make sure it is not less than the
 * minimum we can deal with
 */
if (req.count < BUF_COUNT_MIN) {
    fprintf(stderr, "Insufficient buffer memory on %s\n",             dev_name);
    exit (EXIT_FAILURE);
}
```

之后，应用程序应该对每个分配的缓冲区调用`VIDIOC_QUERYBUF` ioctl，以获取它们对应的物理地址，如下例所示：

```
struct buffer_addr {
    void *start;
    size_t length;
};
struct buffer_addr *buf_addr;
buf_addr = calloc(BUF_COUNT, sizeof (*buffer_addr));
if (!buf_addr) {
    fprintf (stderr, "Out of memory\n");
    exit (EXIT_FAILURE);
}
for (i = 0; i < req.count; ++i) {
    struct v4l2_buffer buf;
    CLEAR (buf);
    buf.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
    buf.memory = V4L2_MEMORY_MMAP; buf.index	= i;
    if (-1 == xioctl (fd, VIDIOC_QUERYBUF, &buf))
        errno_exit("VIDIOC_QUERYBUF");
    buf_addr[i].length = buf.length;
    buf_addr[i].start =
        mmap (NULL /* start anywhere */, buf.length,
              PROT_READ | PROT_WRITE /* required */,
              MAP_SHARED /* recommended */, fd, buf.m.offset);
    if (MAP_FAILED == buf_addr[i].start)
        errno_exit("mmap");
}
```

为了使应用程序能够内部跟踪每个缓冲区的内存映射（使用`mmap()`获得），我们定义了一个自定义数据结构`struct buffer_addr`，为每个授予的缓冲区分配，该结构将保存与该缓冲区对应的映射。

### 请求 DMABUF 缓冲区 - VIDIOC_REQBUFS、VIDIOC_EXPBUF 和 mmap

DMABUF 主要用于`mem2mem`设备，并引入了**导出者**和**导入者**的概念。假设驱动程序**A**想要使用由驱动程序**B**创建的缓冲区；那么我们称**B**为导出者，**A**为缓冲区用户/导入者。

`export`方法指示驱动程序通过文件描述符将其 DMA 缓冲区导出到用户空间。应用程序使用`VIDIOC_EXPBUF` ioctl 来实现这一点，并需要一个新的数据结构`struct v4l2_exportbuffer`。在此 ioctl 的返回路径上，驱动程序将使用文件描述符设置`v4l2_requestbuffers.md`成员，该文件描述符对应于给定缓冲区。这是一个 DMABUF 文件描述符：

```
/* V4L2 DMABuf export */
struct v4l2_requestbuffers req;
CLEAR (req);
req.count = BUF_COUNT;
req.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
req.memory = V4L2_MEMORY_DMABUF;
if (-1 == xioctl(fd, VIDIOC_REQBUFS, &req))
    errno_exit ("VIDIOC_QUERYBUFS");
```

应用程序可以将这些缓冲区导出为 DMABUF 文件描述符，以便可以将其内存映射到访问捕获的视频内容。应用程序应该使用`VIDIOC_EXPBUF`ioctl 来实现这一点。此 ioctl 扩展了内存映射 I/O 方法，因此仅适用于`V4L2_MEMORY_MMAP`缓冲区。但是，使用`VIDIOC_EXPBUF`导出捕获缓冲区然后映射它们实际上是没有意义的。应该使用`V4L2_MEMORY_MMAP`。

`VIDIOC_EXPBUF`在涉及 V4L2 输出设备时变得非常有趣。这样，应用程序可以使用`VIDIOC_REQBUFS`ioctl 在捕获和输出设备上分配缓冲区，然后应用程序将输出设备的缓冲区导出为 DMABUF 文件描述符，并在捕获设备上的入队 ioctl 之前使用这些文件描述符来设置`v4l2_buffer.m.fd`字段。然后，排队的缓冲区将填充其对应的缓冲区（与`v4l2_buffer.m.fd`对应的输出设备缓冲区）。

在下面的示例中，我们将输出设备缓冲区导出为 DMABUF 文件描述符。这假设已经使用`VIDIOC_REQBUFS`ioctl 分配了此输出设备的缓冲区，其中`req.type`设置为`V4L2_BUF_TYPE_VIDEO_OUTPUT`，`req.memory`设置为`V4L2_MEMORY_DMABUF`：

```
int outdev_dmabuf_fd[BUF_COUNT] = {-1};
int i;
for (i = 0; i < req.count; i++) {
    struct v4l2_exportbuffer expbuf;
    CLEAR (expbuf);
    expbuf.type = V4L2_BUF_TYPE_VIDEO_OUTPUT;
    expbuf.index = i;
    if (-1 == xioctl(fd, VIDIOC_EXPBUF, &expbuf)
        errno_exit ("VIDIOC_EXPBUF");
    outdev_dmabuf_fd[i] = expbuf.fd;
}
```

现在，我们已经了解了基于 DMABUF 的流式传输，并介绍了它所带来的概念。接下来和最后的流式传输方法要简单得多，需要的代码也更少。让我们来看看。

请求读/写 I/O 内存

从编码的角度来看，这是更简单的流式传输模式。在**读/写 I/O**的情况下，除了分配应用程序将存储读取数据的内存位置之外，没有其他事情要做，就像下面的示例中一样：

```
struct buffer_addr {
    void *start;
    size_t length;
};
struct buffer_addr *buf_addr;
buf_addr = calloc(1, sizeof(*buf_addr));
if (!buf_addr) {
    fprintf(stderr, "Out of memory\n");
    exit(EXIT_FAILURE);
}
buf_addr[0].length = buffer_size;
buf_addr[0].start = malloc(buffer_size);
if (!buf_addr[0].start) {
    fprintf(stderr, "Out of memory\n");
    exit(EXIT_FAILURE);
}
```

在前面的代码片段中，我们定义了相同的自定义数据结构`struct buffer_addr`。但是，这里没有真正的缓冲区请求（没有使用`VIDIOC_REQBUFS`），因为还没有任何东西传递给内核。缓冲区内存只是被分配了，就是这样。

现在，我们已经完成了缓冲区请求。下一步是将请求的缓冲区加入队列，以便内核可以用视频数据填充它们。现在让我们看看如何做到这一点。

## 将缓冲区加入队列并启用流式传输

在访问缓冲区并读取其数据之前，必须将该缓冲区加入队列。这包括在使用流式 I/O 方法（除了读/写 I/O 之外的所有方法）时，在缓冲区上使用`VIDIOC_QBUF` ioctl。将缓冲区加入队列将锁定该缓冲区在物理内存中的内存页面。这样，这些页面就无法被交换到磁盘上。请注意，这些缓冲区保持锁定状态，直到它们被出队列，直到调用`VIDIOC_STREAMOFF`或`VIDIOC_REQBUFS` ioctls，或者直到设备被关闭。

在 V4L2 上下文中，锁定缓冲区意味着将该缓冲区传递给驱动程序进行硬件访问（通常是 DMA）。如果应用程序访问（读/写）已锁定的缓冲区，则结果是未定义的。

要将缓冲区入队，应用程序必须准备`struct v4l2_buffer`，并根据缓冲区类型、流模式和分配缓冲区时的索引设置`v4l2_buffer.type`、`v4l2_buffer.memory`和`v4l2_buffer.index`。其他字段取决于流模式。

重要提示

*读/写 I/O*方法不需要入队。

### 主缓冲区的概念

对于捕获应用程序，通常在开始捕获并进入读取循环之前，入队一定数量（大多数情况下是分配的缓冲区数量）的空缓冲区是惯例。这有助于提高应用程序的流畅性，并防止因为缺少填充的缓冲区而被阻塞。这应该在分配缓冲区后立即完成。

### 入队用户指针缓冲区

要将用户指针缓冲区入队，应用程序必须将`v4l2_buffer.memory`成员设置为`V4L2_MEMORY_USERPTR`。这里的特殊之处在于`v4l2_buffer.m.userptr`字段，必须设置为先前分配的缓冲区的地址，并且`v4l2_buffer.length`设置为其大小。当使用多平面 API 时，必须使用传递的`struct v4l2_plane`数组的`m.userptr`和`length`成员：

```
/* Prime buffers */
for (i = 0; i < BUF_COUNT; ++i) {
    struct v4l2_buffer buf;
    CLEAR(buf);
    buf.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
    buf.memory = V4L2_MEMORY_USERPTR; buf.index = i;
    buf.m.userptr = (unsigned long)buf_addr[i].start;
    buf.length = buf_addr[i].length;
    if (-1 == xioctl(fd, VIDIOC_QBUF, &buf))
        errno_exit("VIDIOC_QBUF");
}
```

### 入队内存映射缓冲区

要将内存映射缓冲区入队，应用程序必须通过设置`type`、`memory`（必须为`V4L2_MEMORY_MMAP`）和`index`成员来填充`struct v4l2_buffer`，就像以下摘录中所示：

```
/* Prime buffers */
for (i = 0; i < BUF_COUNT; ++i) {
    struct v4l2_buffer buf; CLEAR (buf);
    buf.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
    buf.memory = V4L2_MEMORY_MMAP;
    buf.index = i;
    if (-1 == xioctl (fd, VIDIOC_QBUF, &buf))
        errno_exit ("VIDIOC_QBUF");
}
```

### 入队 DMABUF 缓冲区

要将输出设备的 DMABUF 缓冲区填充到捕获设备的缓冲区中，应用程序应填充`struct v4l2_buffer`，将`memory`字段设置为`V4L2_MEMORY_DMABUF`，将`type`字段设置为`V4L2_BUF_TYPE_VIDEO_CAPTURE`，将`m.fd`字段设置为与输出设备的 DMABUF 缓冲区关联的文件描述符，如下所示：

```
/* Prime buffers */
for (i = 0; i < BUF_COUNT; ++i) {
    struct v4l2_buffer buf; CLEAR (buf);
    buf.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
    buf.memory = V4L2_MEMORY_DMABUF; buf.index	= i;
    buf.m.fd = outdev_dmabuf_fd[i];
    /* enqueue the dmabuf to capture device */
    if (-1 == xioctl (fd, VIDIOC_QBUF, &buf))
        errno_exit ("VIDIOC_QBUF");
}
```

上述代码摘录显示了 V4L2 DMABUF 导入的工作原理。ioctl 中的`fd`参数是与捕获设备关联的文件描述符，在`open()`系统调用中获得。`outdev_dmabuf_fd`是包含输出设备的 DMABUF 文件描述符的数组。您可能会想知道，这如何能在不是 V4L2 但是兼容 DRM 的输出设备上工作，例如。以下是一个简要的解释。

首先，DRM 子系统以驱动程序相关的方式提供 API，您可以使用这些 API 在 GPU 上分配（愚笨的）缓冲区，它将返回一个 GEM 句柄。DRM 还提供了`DRM_IOCTL_PRIME_HANDLE_TO_FD` ioctl，允许通过`PRIME`将缓冲区导出到 DMABUF 文件描述符，然后使用`drmModeAddFB2()` API 创建一个`framebuffer`对象（这是将要读取和显示在屏幕上的东西，或者我应该说，确切地说是 CRT 控制器），对应于这个缓冲区，最终可以使用`drmModeSetPlane()`或`drmModeSetPlane()`API 进行渲染。然后，应用程序可以使用`DRM_IOCTL_PRIME_HANDLE_TO_FD` ioctl 返回的文件描述符设置`v4l2_requestbuffers.m.fd`字段。然后，在读取循环中，在每个`VIDIOC_DQBUF` ioctl 之后，应用程序可以使用`drmModeSetPlane()`API 更改平面的帧缓冲区和位置。

重要提示

`drm dma-buf`接口层集成了`GEM`，这是 DRM 子系统支持的内存管理器之一

### 启用流式传输

启用流式传输有点像通知 V4L2 从现在开始将*输出*队列作为访问对象。应用程序应使用`VIDIOC_STREAMON`来实现这一点。以下是一个示例：

```
/* Start streaming */
int ret;
int a = V4L2_BUF_TYPE_VIDEO_CAPTURE;
ret = xioctl(capt.fd, VIDIOC_STREAMON, &a);
if (ret < 0) {
    perror("VIDIOC_STREAMON\n");
    return -1;
}
```

上述摘录很短，但是必须启用流式传输，否则稍后无法出队缓冲区。

## 出队缓冲区

这实际上是应用程序的读取循环的一部分。应用程序使用`VIDIOC_DQBUF` ioctl 出队缓冲区。只有在流启用之后才可能。当应用程序调用`VIDIOC_DQBUF` ioctl 时，它指示驱动程序检查是否有任何已填充的缓冲区（在`open()`系统调用期间设置了`O_NONBLOCK`标志），直到缓冲区排队并填充。

重要提示

尝试在排队之前出队缓冲区是一个错误，`VIDIOC_DQBUF` ioctl 应该返回`-EINVAL`。当`O_NONBLOCK`标志给定给`open()`函数时，当没有可用的缓冲区时，`VIDIOC_DQBUF`立即返回`EAGAIN`错误代码。

出队缓冲区并处理其数据后，应用程序必须立即将此缓冲区重新排队，以便为下一次读取重新填充，依此类推。

### 出队内存映射缓冲区

以下是一个出队已经内存映射的缓冲区的示例：

```
struct v4l2_buffer buf;
CLEAR (buf);
buf.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
buf.memory = V4L2_MEMORY_MMAP;
if (-1 == xioctl (fd, VIDIOC_DQBUF, &buf)) {
    switch (errno) {
    case EAGAIN:
        return 0;
    case EIO:
    default:
        errno_exit ("VIDIOC_DQBUF");
    }
}
/* make sure the returned index is coherent with the number
 * of buffers allocated  */
assert (buf.index < BUF_COUNT);
/* We use buf.index to point to the correct entry in our  * buf_addr  */ 
process_image(buf_addr[buf.index].start);
/* Queue back this buffer again, after processing is done */
if (-1 == xioctl (fd, VIDIOC_QBUF, &buf))
    errno_exit ("VIDIOC_QBUF");
```

这可以在循环中完成。例如，假设您需要 200 张图像。读取循环可能如下所示：

```
#define MAXLOOPCOUNT 200
/* Start the loop of capture */
for (i = 0; i < MAXLOOPCOUNT; i++) {
    struct v4l2_buffer buf;
    CLEAR (buf);
    buf.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
    buf.memory = V4L2_MEMORY_MMAP;
    if (-1 == xioctl (fd, VIDIOC_DQBUF, &buf)) {
        [...]
    }
   /* Queue back this buffer again, after processing is done */
    [...]
}
```

上面的片段只是使用循环重新实现了缓冲区出队，其中计数器表示需要抓取的图像数量。

### 出队用户指针缓冲区

以下是使用**用户指针**出队缓冲区的示例：

```
struct v4l2_buffer buf; int i;
CLEAR (buf);
buf.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
buf.memory = V4L2_MEMORY_USERPTR;
/* Dequeue a captured buffer */
if (-1 == xioctl (fd, VIDIOC_DQBUF, &buf)) {
    switch (errno) {
    case EAGAIN:
        return 0;
    case EIO:
        [...]
    default:
        errno_exit ("VIDIOC_DQBUF");
    }
}
/*
 * We may need the index to which corresponds this buffer
 * in our buf_addr array. This is done by matching address
 * returned by the dequeue ioctl with the one stored in our
 * array  */
for (i = 0; i < BUF_COUNT; ++i)
    if (buf.m.userptr == (unsigned long)buf_addr[i].start &&
                        buf.length == buf_addr[i].length)
        break;
/* the corresponding index is used for sanity checks only */ 
assert (i < BUF_COUNT);
process_image ((void *)buf.m.userptr);
/* requeue the buffer */
if (-1 == xioctl (fd, VIDIOC_QBUF, &buf))
    errno_exit ("VIDIOC_QBUF");
```

上面的代码展示了如何出队用户指针缓冲区，并且有足够的注释，不需要进一步解释。然而，如果需要许多缓冲区，这可以在循环中实现。

### 读/写 I/O

这是最后一个示例，展示了如何使用`read()`系统调用出队缓冲区：

```
if (-1 == read (fd, buffers[0].start, buffers[0].length)) {
    switch (errno) {
    case EAGAIN:
        return 0;
    case EIO:
        [...]
    default:
        errno_exit ("read");
    }
}
process_image (buffers[0].start);
```

之前的示例没有详细讨论，因为它们每个都使用了在*V4L2 用户空间 API*部分已经介绍的概念。现在我们已经熟悉了编写 V4L2 用户空间代码，让我们看看如何通过使用专用工具来快速原型设计摄像头系统而不编写任何代码。

# V4L2 用户空间工具

到目前为止，我们已经学会了如何编写用户空间代码与内核中的驱动程序进行交互。对于快速原型设计和测试，我们可以利用一些社区提供的 V4L2 用户空间工具。通过使用这些工具，我们可以专注于系统设计并验证摄像头系统。最知名的工具是`v4l2-ctl`，我们将重点关注它；它随`v4l-utils`软件包一起提供。

尽管本章没有讨论，但还有**yavta**工具（代表**Yet Another V4L2 Test Application**），它可以用于测试、调试和控制摄像头子系统。

## 使用 v4l2-ctl

`v4l2-utils`是一个用户空间应用程序，可用于查询或配置 V4L2 设备（包括子设备）。该工具可以帮助设置和设计精细的基于 V4L2 的系统，因为它有助于调整和利用设备的功能。

重要提示

`qv4l2`是`v4l2-ctl`的 Qt GUI 等效物。`v4l2-ctl`非常适合嵌入式系统，而`qv4l2`非常适合交互式测试。

### 列出视频设备及其功能

首先，我们需要使用`--list-devices`选项列出所有可用的视频设备：

```
# v4l2-ctl --list-devices
Integrated Camera: Integrated C (usb-0000:00:14.0-8):
	/dev/video0
	/dev/video1
```

如果有多个设备可用，我们可以在任何`v4l2-ctl`命令之后使用`-d`选项来针对特定设备。请注意，如果未指定`-d`选项，默认情况下会针对`/dev/video0`。

要获取有关特定设备的信息，必须使用`-D`选项，如下所示：

```
# v4l2-ctl -d /dev/video0 -D
Driver Info (not using libv4l2):
	Driver name   : uvcvideo
	Card type     : Integrated Camera: Integrated C
	Bus info      : usb-0000:00:14.0-8
	Driver version: 5.4.60
	Capabilities  : 0x84A00001
		Video Capture
		Metadata Capture
		Streaming
		Extended Pix Format
		Device Capabilities
	Device Caps   : 0x04200001
		Video Capture
		Streaming
		Extended Pix Format
```

上面的命令显示了设备信息（如驱动程序及其版本）以及其功能。也就是说，`--all`命令提供更好的详细信息。你应该试一试。

### 更改设备属性（控制设备）

在查看更改设备属性之前，我们首先需要知道设备支持的控制、它们的值类型（整数、布尔、字符串等）、它们的默认值以及接受的值是什么。

为了获取设备支持的控制列表，我们可以使用`v4l2-ctl`和`-L`选项，如下所示：

```
# v4l2-ctl -L
                brightness 0x00980900 (int)  : min=0 max=255 step=1 default=128 value=128
                contrast 0x00980901 (int)    : min=0 max=255 step=1 default=32 value=32
                saturation 0x00980902 (int)  : min=0 max=100 step=1 default=64 value=64
                     hue 0x00980903 (int)    : min=-180 max=180 step=1 default=0 value=0
 white_balance_temperature_auto 0x0098090c (bool)   : default=1 value=1
                     gamma 0x00980910 (int)  : min=90 max=150 step=1 default=120 value=120
         power_line_frequency 0x00980918 (menu)   : min=0 max=2 default=1 value=1
				0: Disabled
				1: 50 Hz
				2: 60 Hz
      white_balance_temperature 0x0098091a (int)  : min=2800 max=6500 step=1 default=4600 value=4600 flags=inactive
                    sharpness 0x0098091b (int)    : min=0 max=7 step=1 default=3 value=3
       backlight_compensation 0x0098091c (int)    : min=0 max=2 step=1 default=1 value=1
                exposure_auto 0x009a0901 (menu)   : min=0 max=3 default=3 value=3
				1: Manual Mode
				3: Aperture Priority Mode
         exposure_absolute 0x009a0902 (int)    : min=5 max=1250 step=1 default=157 value=157 flags=inactive
         exposure_auto_priority 0x009a0903 (bool)   : default=0 value=1
jma@labcsmart:~$
```

在上述输出中，`"value="`字段返回控制的当前值，其他字段都是不言自明的。

既然我们已经知道设备支持的控制列表，控制值可以通过`--set-ctrl`选项进行更改，如下例所示：

```
# v4l2-ctl --set-ctrl brightness=192
```

之后，我们可以使用以下命令检查当前值：

```
# v4l2-ctl -L
                 brightness 0x00980900 (int)    : min=0 max=255 step=1 default=128 value=192
                     [...]
```

或者，我们可以使用`--get-ctrl`命令，如下所示：

```
# v4l2-ctl --get-ctrl brightness 
brightness: 192
```

现在可能是时候调整设备了。在此之前，让我们先检查一下设备的视频特性。

设置像素格式、分辨率和帧率

在选择特定格式或分辨率之前，我们需要列举设备可用的内容。为了获取支持的像素格式、分辨率和帧率，需要向`v4l2-ctl`提供`--list-formats-ext`选项，如下所示：

```
# v4l2-ctl --list-formats-ext
ioctl: VIDIOC_ENUM_FMT
	Index       : 0
	Type        : Video Capture
	Pixel Format: 'MJPG' (compressed)
	Name        : Motion-JPEG
		Size: Discrete 1280x720
			Interval: Discrete 0.033s (30.000 fps)
		Size: Discrete 960x540
			Interval: Discrete 0.033s (30.000 fps)
		Size: Discrete 848x480
			Interval: Discrete 0.033s (30.000 fps)
		Size: Discrete 640x480
			Interval: Discrete 0.033s (30.000 fps)
		Size: Discrete 640x360
			Interval: Discrete 0.033s (30.000 fps)
		Size: Discrete 424x240
			Interval: Discrete 0.033s (30.000 fps)
		Size: Discrete 352x288
			Interval: Discrete 0.033s (30.000 fps)
		Size: Discrete 320x240
			Interval: Discrete 0.033s (30.000 fps)
		Size: Discrete 320x180
			Interval: Discrete 0.033s (30.000 fps)
	Index       : 1
	Type        : Video Capture
	Pixel Format: 'YUYV'
	Name        : YUYV 4:2:2
		Size: Discrete 1280x720
			Interval: Discrete 0.100s (10.000 fps)
		Size: Discrete 960x540
			Interval: Discrete 0.067s (15.000 fps)
		Size: Discrete 848x480
			Interval: Discrete 0.050s (20.000 fps)
		Size: Discrete 640x480
			Interval: Discrete 0.033s (30.000 fps)
		Size: Discrete 640x360
			Interval: Discrete 0.033s (30.000 fps)
		Size: Discrete 424x240
			Interval: Discrete 0.033s (30.000 fps)
		Size: Discrete 352x288
			Interval: Discrete 0.033s (30.000 fps)
		Size: Discrete 320x240
			Interval: Discrete 0.033s (30.000 fps)
		Size: Discrete 320x180
			Interval: Discrete 0.033s (30.000 fps)
```

从上述输出中，我们可以看到目标设备支持的内容，即`mjpeg`压缩格式和 YUYV 原始格式。

现在，为了更改摄像头配置，首先使用`--set-parm`选项选择帧率，如下所示：

```
# v4l2-ctl --set-parm=30
Frame rate set to 30.000 fps
#
```

然后，可以使用`--set-fmt-video`选项选择所需的分辨率和/或像素格式，如下所示：

```
# v4l2-ctl --set-fmt-video=width=640,height=480,  pixelformat=MJPG
```

在帧率方面，您可能希望使用`v4l2-ctl`和`--set-parm`选项，只提供帧率的分子—分母固定为`1`（只允许整数帧率值）—如下所示：

```
# v4l2-ctl --set-parm=<framerate numerator>
```

### 捕获帧和流处理

`v4l2-ctl`支持的选项比您想象的要多得多。为了查看可能的选项，可以打印适当部分的帮助消息。与流处理和视频捕获相关的常见帮助命令如下：

+   `--help-streaming`：打印所有与流处理相关的选项的帮助消息

+   `--help-subdev`：打印所有与`v4l-subdevX`设备相关的选项的帮助消息

+   `--help-vidcap`：打印所有获取/设置/列出视频捕获格式的选项的帮助消息

从这些帮助命令中，我已经构建了以下命令，以便在磁盘上捕获 QVGA MJPG 压缩帧：

```
# v4l2-ctl --set-fmt-video=width=320,height=240,  pixelformat=MJPG \
   --stream-mmap --stream-count=1 --stream-to=grab-320x240.mjpg
```

我还使用以下命令捕获了一个具有相同分辨率的原始 YUV 图像：

```
# v4l2-ctl --set-fmt-video=width=320,height=240,  pixelformat=YUYV \
  --stream-mmap --stream-count=1 --stream-to=grab-320x240-yuyv.raw
```

除非使用一个体面的原始图像查看器，否则无法显示原始 YUV 图像。为了做到这一点，必须使用`ffmpeg`工具转换原始图像，例如如下所示：

```
# ffmpeg -f rawvideo -s 320x240 -pix_fmt yuyv422 \
         -i grab-320x240-yuyv.raw grab-320x240.png
```

您可以注意到原始图像和压缩图像之间的大小差异很大，如下摘录所示：

```
# ls -hl grab-320x240.mjpg
-rw-r--r-- 1 root root 8,0K oct.  21 20:26 grab-320x240.mjpg
# ls -hl grab-320x240-yuyv.raw 
-rw-r--r-- 1 root root 150K oct.  21 20:26 grab-320x240-yuyv.raw
```

请注意，在原始捕获的文件名中包含图像格式是一个好习惯（例如在`grab-320x240-yuyv.raw`中包含`yuyv`），这样您就可以轻松地从正确的格式进行转换。对于压缩图像格式，这条规则是不必要的，因为这些格式是带有描述其后的像素数据的标头的图像容器格式，并且可以很容易地使用`gst-typefind-1.0`工具进行读取。JPEG 就是这样一种格式，以下是如何读取其标头的方法：

```
# gst-typefind-1.0 grab-320x240.mjpg 
grab-320x240.mjpg - image/jpeg, width=(int)320, height=(int)240, sof-marker=(int)0
# gst-typefind-1.0 grab-320x240-yuyv.raw 
grab-320x240-yuyv.raw - FAILED: Could not determine type of stream.
```

现在我们已经完成了工具的使用，让我们看看如何深入了解 V4L2 调试以及从用户空间开始学习。

## 在用户空间调试 V4L2

由于我们的视频系统设置可能不是没有错误的，V4L2 为了从用户空间进行跟踪和排除来自 VL4L2 框架核心或用户空间 API 的故障，提供了一个简单但大的后门调试。

可以按照以下步骤启用框架调试：

```
# echo 0x3 > /sys/module/videobuf2_v4l2/parameters/debug
# echo 0x3 > /sys/module/videobuf2_common/parameters/debug
```

上述命令将指示 V4L2 向内核日志消息添加核心跟踪。这样，它将很容易地跟踪故障的来源，假设故障来自核心。运行以下命令：

```
# dmesg
[831707.512821] videobuf2_common: __setup_offsets: buffer 0, plane 0 offset 0x00000000
[831707.512915] videobuf2_common: __setup_offsets: buffer 1, plane 0 offset 0x00097000
[831707.513003] videobuf2_common: __setup_offsets: buffer 2, plane 0 offset 0x0012e000
[831707.513118] videobuf2_common: __setup_offsets: buffer 3, plane 0 offset 0x001c5000
[831707.513119] videobuf2_common: __vb2_queue_alloc: allocated 4 buffers, 1 plane(s) each
[831707.513169] videobuf2_common: vb2_mmap: buffer 0, plane 0 successfully mapped
[831707.513176] videobuf2_common: vb2_core_qbuf: qbuf of buffer 0 succeeded
[831707.513205] videobuf2_common: vb2_mmap: buffer 1, plane 0 successfully mapped
[831707.513208] videobuf2_common: vb2_core_qbuf: qbuf of buffer 1 succeeded
[...]
```

在先前的内核日志消息中，我们可以看到与内核相关的 V4L2 核心函数调用，以及一些其他细节。如果由于任何原因 V4L2 核心跟踪不是必要的或者对您来说不够，您还可以使用以下命令启用 V4L2 用户空间 API 跟踪：

```
$ echo 0x3 > /sys/class/video4linux/video0/dev_debug
```

运行命令后，允许您捕获原始图像，我们可以在内核日志消息中看到以下内容：

```
$ dmesg
[833211.742260] video0: VIDIOC_QUERYCAP: driver=uvcvideo, card=Integrated Camera: Integrated C, bus=usb-0000:00:14.0-8, version=0x0005043c, capabilities=0x84a00001, device_caps=0x04200001
[833211.742275] video0: VIDIOC_QUERY_EXT_CTRL: id=0x980900, type=1, name=Brightness, min/max=0/255, step=1, default=128, flags=0x00000000, elem_size=4, elems=1, nr_of_dims=0, dims=0,0,0,0
[...]
[833211.742318] video0: VIDIOC_QUERY_EXT_CTRL: id=0x98090c, type=2, name=White Balance Temperature, Auto, min/max=0/1, step=1, default=1, flags=0x00000000, elem_size=4, elems=1, nr_of_dims=0, dims=0,0,0,0
[833211.742365] video0: VIDIOC_QUERY_EXT_CTRL: id=0x98091c, type=1, name=Backlight Compensation, min/max=0/2, step=1, default=1, flags=0x00000000, elem_size=4, elems=1, nr_of_dims=0, dims=0,0,0,0
[833211.742376] video0: VIDIOC_QUERY_EXT_CTRL: id=0x9a0901, type=3, name=Exposure, Auto, min/max=0/3, step=1, default=3, flags=0x00000000, elem_size=4, elems=1, nr_of_dims=0, dims=0,0,0,0
[...]
[833211.756641] videobuf2_common: vb2_mmap: buffer 1, plane 0 successfully mapped
[833211.756646] videobuf2_common: vb2_core_qbuf: qbuf of buffer 1 succeeded
[833211.756649] video0: VIDIOC_QUERYBUF: 00:00:00.00000000 index=2, type=vid-cap, request_fd=0, flags=0x00012000, field=any, sequence=0, memory=mmap, bytesused=0, offset/userptr=0x12e000, length=614989
[833211.756657] timecode=00:00:00 type=0, flags=0x00000000, frames=0, userbits=0x00000000
[833211.756698] videobuf2_common: vb2_mmap: buffer 2, plane 0 successfully mapped
[833211.756704] videobuf2_common: vb2_core_qbuf: qbuf of buffer 2 succeeded
[833211.756706] video0: VIDIOC_QUERYBUF: 00:00:00.00000000 index=3, type=vid-cap, request_fd=0, flags=0x00012000, field=any, sequence=0, memory=mmap, bytesused=0, offset/userptr=0x1c5000, length=614989
[833211.756714] timecode=00:00:00 type=0, flags=0x00000000, frames=0, userbits=0x00000000
[833211.756751] videobuf2_common: vb2_mmap: buffer 3, plane 0 successfully mapped
[833211.756755] videobuf2_common: vb2_core_qbuf: qbuf of buffer 3 succeeded
[833212.967229] videobuf2_common: vb2_core_streamon: successful
[833212.967234] video0: VIDIOC_STREAMON: type=vid-cap
```

在先前的输出中，我们可以跟踪不同的 V4L2 用户空间 API 调用，这些调用对应于不同的`ioctl`命令及其参数。

### V4L2 合规性驱动程序测试

为了使驱动程序符合 V4L2 标准，它必须满足一些标准，其中包括通过`v4l2-compliance`工具测试，该工具用于测试各种类型的 V4L 设备。`v4l2-compliance`试图测试 V4L2 设备的几乎所有方面，并涵盖几乎所有 V4L2 ioctls。

与其他 V4L2 工具一样，可以使用`-d`或`--device=`命令来定位视频设备。如果未指定设备，则将定位到`/dev/video0`。以下是一个输出摘录：

```
# v4l2-compliance
v4l2-compliance SHA   : not available
Driver Info:
	Driver name   : uvcvideo
	Card type     : Integrated Camera: Integrated C
	Bus info      : usb-0000:00:14.0-8
	Driver version: 5.4.60
	Capabilities  : 0x84A00001
		Video Capture
		Metadata Capture
		Streaming
		Extended Pix Format
		Device Capabilities
	Device Caps   : 0x04200001
		Video Capture
		Streaming
		Extended Pix Format
Compliance test for device /dev/video0 (not using libv4l2):
Required ioctls:
	test VIDIOC_QUERYCAP: OK
Allow for multiple opens:
	test second video open: OK
	test VIDIOC_QUERYCAP: OK
	test VIDIOC_G/S_PRIORITY: OK
	test for unlimited opens: OK
Debug ioctls:
	test VIDIOC_DBG_G/S_REGISTER: OK (Not Supported)
	test VIDIOC_LOG_STATUS: OK (Not Supported)
[]
Output ioctls:
	test VIDIOC_G/S_MODULATOR: OK (Not Supported)
	test VIDIOC_G/S_FREQUENCY: OK (Not Supported)
[...]
Test input 0:
	Control ioctls:
		fail: v4l2-test-controls.cpp(214): missing control class for class 00980000
		fail: v4l2-test-controls.cpp(251): missing control class for class 009a0000
		test VIDIOC_QUERY_EXT_CTRL/QUERYMENU: FAIL
		test VIDIOC_QUERYCTRL: OK
		fail: v4l2-test-controls.cpp(437): s_ctrl returned an error (84)
		test VIDIOC_G/S_CTRL: FAIL
		fail: v4l2-test-controls.cpp(675): s_ext_ctrls returned an error (
```

在先前的日志中，我们可以看到已定位到`/dev/video0`。此外，我们注意到我们的驱动程序不支持`Debug ioctls`和`Output ioctls`（这些不是失败）。尽管输出已经足够详细，但最好也使用`--verbose`命令，这样输出会更加用户友好和更加详细。因此，毫无疑问，如果要提交新的 V4L2 驱动程序，该驱动程序必须通过 V4L2 合规性测试。

# 摘要

在本章中，我们介绍了 V4L2 的用户空间实现。我们从视频流的 V4L2 缓冲区管理开始。我们还学习了如何处理视频设备属性管理，都是从用户空间进行的。然而，V4L2 是一个庞大的框架，不仅在代码方面，而且在功耗方面也是如此。因此，在下一章中，我们将讨论 Linux 内核的电源管理，以使系统在不降低系统性能的情况下保持尽可能低的功耗水平。
