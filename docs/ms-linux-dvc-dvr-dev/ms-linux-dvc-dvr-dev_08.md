*第六章*：ALSA SoC 框架-深入了解机器类驱动程序

在开始我们的 ALSA SoC 框架系列时，我们注意到平台和编解码器类驱动程序都不打算单独工作。ASoC 架构设计成平台和编解码器类驱动程序必须绑定在一起才能构建音频设备。这种绑定可以通过所谓的机器驱动程序或者设备树内部完成，每个都是特定于机器的。因此可以毫不夸张地说，机器驱动程序针对特定系统，可能会从一个板子变成另一个板子。在本章中，我们将重点介绍 AsoC 机器类驱动程序的不足之处，并讨论在需要编写机器类驱动程序时可能遇到的特定情况。

在本章中，我们将介绍 Linux ASoC 驱动程序架构和实现。本章将分为不同的部分，如下所示：

+   机器类驱动程序介绍

+   机器路由考虑

+   时钟和格式考虑

+   声卡注册

+   利用 simple-card 机器驱动程序

# 第六章：技术要求

本章需要以下内容：

+   对设备树概念的深入了解

+   熟悉平台和编解码器类驱动程序（在*第五章**,* *ALSA SoC 框架-利用编解码器和平台类驱动程序*中讨论）

+   Linux 内核 v4.19.X 源代码，可在[`git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/refs/tags`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/refs/tags)上找到

# 机器类驱动程序介绍

编解码器和平台驱动程序不能单独工作。机器驱动程序负责将它们绑定在一起，以完成音频信息处理。机器驱动程序类充当胶水，描述和绑定其他组件驱动程序以形成 ALSA 声卡设备。它管理任何特定于机器的控件和机器级音频事件（例如在播放开始时打开放大器）。机器驱动程序描述并绑定 CPU `struct snd_soc_dai_link`结构，并实例化声卡`struct snd_soc_card`。

平台和编解码器驱动程序通常是可重用的，但机器驱动程序通常不是，因为它们具有大多数情况下不可重用的特定硬件特性。所谓的硬件特性指的是 DAI 之间的链接；通过 GPIO 打开放大器；通过 GPIO 检测插入；使用时钟如 MCLK/外部 OSC 作为 I2 的参考时钟源；编解码器模块等。一般来说，机器驱动程序的责任包括以下内容：

+   使用适当的 CPU 和编解码器 DAI 填充`struct snd_soc_dai_link`结构

+   物理编解码器时钟设置（如果有）和编解码器初始化主/从配置（如果有）

+   定义 DAPM 小部件以通过物理编解码器内部进行路由，并根据需要完成 DAPM 路径

+   根据需要将运行时采样频率传播到各个编解码器驱动程序

总之，我们有以下流程：

1.  编解码器驱动程序注册了一个组件驱动程序、一个 DAI 驱动程序以及它们的操作函数。

1.  平台驱动程序注册了一个组件驱动程序、PCM 驱动程序、CPU DAI 驱动程序以及它们的操作函数，并设置适用的播放和捕获操作。

1.  机器层在编解码器和 CPU 之间创建 DAI 链路，并注册声卡和 PCM 设备。

既然我们已经看到了机器类驱动程序的开发流程，让我们从第一步开始，即填充 DAI 链路。

## DAI 链路

DAI 链路是 CPU 和编解码器 DAI 之间的逻辑表示。它在内核中使用`struct snd_soc_dai_link`表示，定义如下：

```
struct snd_soc_dai_link {
    const char *name;
    const char *stream_name;
    const char *cpu_name;
    struct device_node *cpu_of_node;
    const char *cpu_dai_name;
    const char *codec_name;
    struct device_node *codec_of_node;
    const char *codec_dai_name;
    struct snd_soc_dai_link_component *codecs;
    unsigned int num_codecs;
    const char *platform_name;
    struct device_node *platform_of_node;
    int id;
    const struct snd_soc_pcm_stream *params;
    unsigned int num_params;
    unsigned int dai_fmt;
    enum snd_soc_dpcm_trigger trigger[2];
  /* codec/machine specific init - e.g. add machine controls */
    int (*init)(struct snd_soc_pcm_runtime *rtd);
    /* machine stream operations */
    const struct snd_soc_ops *ops;
    /* For unidirectional dai links */
    unsigned int playback_only:1;
    unsigned int capture_only:1;
    /* Keep DAI active over suspend */
    unsigned int ignore_suspend:1;
[...]
    /* DPCM capture and Playback support */
    unsigned int dpcm_capture:1;
    unsigned int dpcm_playback:1;
    struct list_head list; /* DAI link list of the soc card */
};
```

重要说明

完整的 `snd_soc_dai_link` 数据结构定义可以在 [`elixir.bootlin.com/linux/v4.19/source/include/sound/soc.h#L880`](https://elixir.bootlin.com/linux/v4.19/source/include/sound/soc.h#L880) 找到。

此链接是在机器驱动程序中设置的。它应该指定 `cpu_dai`、`codec_dai` 和使用的平台。设置完成后，DAI 链接将被馈送到 `struct snd_soc_card`，它表示一个声卡。以下列表描述了结构中的元素：

+   `name`：这是任意选择的。可以是任何东西。

+   `codec_dai_name`：这必须与编解码器芯片驱动程序中的 `snd_soc_dai_driver.name` 字段匹配。编解码器可能有一个或多个 DAIs。请参考编解码器驱动程序以识别 DAI 名称。

+   `cpu_dai_name`：这必须与 CPU DAI 驱动程序中的 `snd_soc_dai_driver.name` 字段匹配。

+   `stream_name`：这是此链接的流名称。

+   `init`：这是 DAI 链接初始化回调。通常用于添加 DAI 链接特定的小部件或其他类型的一次性设置。

+   `dai_fmt`：这应该设置为支持的格式和时钟配置，对于 CPU 和 CODEC DAI 驱动程序应该是一致的。此字段的可能位标志稍后介绍。

+   `ops`：此字段是 `struct snd_soc_ops` 类型。它应该设置为 DAI 链接的机器级 PCM 操作：`startup`、`hw_params`、`prepare`、`trigger`、`hw_free`、`shutdown`。此字段稍后将详细描述。

+   `codec_name`：如果设置，这应该是编解码器驱动程序的名称，例如 `platform_driver.driver.name` 或 `i2c_driver.driver.name`。

+   `codec_of_node`：与编解码器关联的设备树节点。

+   `cpu_name`：如果设置，这应该是 CPU DAI 驱动程序 CPU 的名称。

+   `cpu_of_node`：这是与 CPU DAI 关联的设备树节点。

+   `platform_name` 或 `platform_of_node`：这是提供 DMA 能力的平台节点的名称或 DT 节点引用。

+   `playback_only` 和 `capture_only` 用于单向链接，例如 SPDIF。如果这是一个仅输出的链接（仅播放），那么必须将 `playback_only` 和 `capture_only` 分别设置为 `true` 和 `false`。对于仅输入的链接，应使用相反的值。

在大多数情况下，`.cpu_of_node` 和 `.platform_of_node` 是相同的，因为 CPU DAI 驱动程序和 DMA PCM 驱动程序是由同一设备实现的。也就是说，您必须通过名称或 `of_node` 指定链接的编解码器，但不能同时使用两者。对于 CPU 和平台，您必须做同样的事情。但是，至少必须指定 CPU DAI 名称或 CPU 设备名称/节点中的一个。这可以总结如下：

```
if (link->platform_name && link->platform_of_node)
    ==> Error
if (link->cpu_name && link->cpu_of_node)
    ==> Eror
if (!link->cpu_dai_name && !(link->cpu_name ||                              link->cpu_of_node))
    ==> Error
```

这里有一个值得注意的关键点。我们如何在 DAI 链接中引用平台或 CPU 节点？我们将在后面回答这个问题。首先考虑以下两个设备节点。第一个（`ssi1`）是 i.mx6 SoC 的 SSI `cpu-dai` 节点。第二个节点（`sgtl5000`）代表 sgtl5000 编解码器芯片：

```
ssi1: ssi@2028000 {
    #sound-dai-cells = <0>;
    compatible = "fsl,imx6q-ssi", "fsl,imx51-ssi";
    reg = <0x02028000 0x4000>;
    interrupts = <0 46 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&clks IMX6QDL_CLK_SSI1_IPG>,
             <&clks IMX6QDL_CLK_SSI1>;
    clock-names = "ipg", "baud";
    dmas = <&sdma 37 1 0>, <&sdma 38 1 0>;
    dma-names = "rx", "tx";
    fsl,fifo-depth = <15>;
    status = "disabled";
};
&i2c0{
    sgtl5000: codec@0a {
        compatible = "fsl,sgtl5000";
        #sound-dai-cells = <0>;
        reg = <0x0a>;
        clocks = <&audio_clock>;
        VDDA-supply = <&reg_3p3v>;
        VDDIO-supply = <&reg_3p3v>;
        VDDD-supply = <&reg_1p5v>;
    };
};
```

重要提示

在 SSI 节点中，您可以看到 `dma-names = "rx", "tx";` 属性，这是 pcmdmaengine 框架请求的预期 DMA 通道名称。这也可能表明 CPU DAI 和平台 PCM 由同一节点表示。

我们将考虑一个系统，其中 i.MX6 SoC 连接到 sgtl5000 音频编解码器。通常，机器驱动程序会通过引用这些节点（实际上是它们的 `phandle`）作为其属性来获取 CPU 或 CODEC 设备树节点。这样，您可以使用 `OF` 助手之一（例如 `of_parse_phandle()`）来获取对这些节点的引用。以下是一个通过 `OF` 节点引用编解码器和平台的机器节点的示例：

```
sound {
    compatible = "fsl,imx51-babbage-sgtl5000",
                 "fsl,imx-audio-sgtl5000";
    model = "imx51-babbage-sgtl5000";
    ssi-controller = <&ssi1>;
    audio-codec = <&sgtl5000>;
    [...]
};
```

在前面的机器节点中，编解码器和 CPUE 通过`audio-codec`和`ssi-controller`属性（它们的`phandle`）传递引用。只要机器驱动程序是由您编写的（如果您使用`simple-card`机器驱动程序，这就不成立，因为它期望一些预定义的名称），这些属性名称就不是标准化的。在机器驱动程序中，你会看到类似这样的东西：

```
static int imx_sgtl5000_probe(struct platform_device *pdev)
{
    struct device_node *np = pdev->dev.of_node;
    struct device_node *ssi_np, *codec_np;
    struct imx_sgtl5000_data *data = NULL;
    int int_port, ext_port; int ret;
[...]
    ssi_np = of_parse_phandle(pdev->dev.of_node,                               "ssi-controller", 0);
    codec_np = of_parse_phandle(pdev->dev.of_node,                                 "audio-codec", 0);
    if (!ssi_np || !codec_np) {
        dev_err(&pdev->dev, "phandle missing or invalid\n");
        ret = -EINVAL;
        goto fail;
    }
    data = devm_kzalloc(&pdev->dev, sizeof(*data), GFP_KERNEL);
    if (!data) {
        ret = -ENOMEM;
       goto fail;
    }
    data->dai.name = "HiFi";
    data->dai.stream_name = "HiFi";
    data->dai.codec_dai_name = "sgtl5000";
    data->dai.codec_of_node = codec_np;
    data->dai.cpu_of_node = ssi_np;
    data->dai.platform_of_node = ssi_np;
    data->dai.init = &imx_sgtl5000_dai_init;
    data->card.dev = &pdev->dev;
    [...]
};
```

前面的摘录使用`of_parse_phandle()`来获取节点引用。这是来自内核源码中的`imx_sgtl5000`机器的摘录，它在`sound/soc/fsl/imx-sgtl5000.c`中。现在我们已经熟悉了应该如何处理 DAI 链路，我们可以继续从机器驱动程序中进行音频路由，以定义音频数据应该遵循的路径。

# 机器路由考虑

机器驱动程序可以更改（或者说追加）从编解码器内部定义的路由。它对应该使用哪些编解码器引脚有最后的决定权，例如。

## 编解码器引脚

编解码器引脚应该连接到板连接器。可用的编解码器引脚在编解码器驱动程序中使用`SND_SOC_DAPM_INPUT`和`SND_SOC_DAPM_OUTPUT`宏来定义。这些宏可以在编解码器驱动程序中使用`grep`命令进行搜索，以找到可用的引脚。

例如，`sgtl5000`编解码器驱动程序定义了以下输出和输入：

```
static const struct snd_soc_dapm_widget sgtl5000_dapm_widgets[] = {
    SND_SOC_DAPM_INPUT("LINE_IN"),
    SND_SOC_DAPM_INPUT("MIC_IN"),
    SND_SOC_DAPM_OUTPUT("HP_OUT"),
    SND_SOC_DAPM_OUTPUT("LINE_OUT"),
    SND_SOC_DAPM_SUPPLY("Mic Bias", SGTL5000_CHIP_MIC_CTRL, 8,                         0,
                        mic_bias_event,
                        SND_SOC_DAPM_POST_PMU |                         SND_SOC_DAPM_PRE_PMD),
[...]
};
```

在接下来的章节中，我们将看到这些引脚是如何连接到板上的。

## 板连接器

板连接器在已注册的`struct snd_soc_card`的机器驱动程序中定义在`struct snd_soc_dapm_widget`部分。大多数情况下，这些板连接器是虚拟的。它们只是逻辑标签，与编解码器引脚连接（这次是真实的）。以下列出了`imx-sgtl5000`机器驱动程序中定义的连接器，`sound/soc/fsl/imx-sgtl5000.c`（其文档是`Documentation/devicetree/bindings/sound/imx-audio-sgtl5000.txt`），迄今为止已经给出了一个例子。

```
static const struct snd_soc_dapm_widget imx_sgtl5000_dapm_widgets[] = { 
    SND_SOC_DAPM_MIC("Mic Jack", NULL),
    SND_SOC_DAPM_LINE("Line In Jack", NULL),
    SND_SOC_DAPM_HP("Headphone Jack", NULL),
    SND_SOC_DAPM_SPK("Line Out Jack", NULL),
    SND_SOC_DAPM_SPK("Ext Spk", NULL),
};
```

接下来的章节将把这个连接器连接到编解码器引脚。

## 机器路由

最终的机器路由可以是静态的（即从机器驱动程序内部填充）或者从设备树内部填充。此外，机器驱动程序可以选择扩展编解码器电源映射，并通过连接到已在编解码器驱动程序中定义的供应小部件，使用`SND_SOC_DAPM_SUPPLY`或`SND_SOC_DAPM_REGULATOR_SUPPLY`成为音频子系统的音频电源映射。

### 设备树路由

让我们以我们的机器节点为例，它连接了一个 i.MX6 SoC 和一个 sgtl5000 编解码器（这个摘录可以在机器文档中找到）。

```
sound {
    compatible = "fsl,imx51-babbage-sgtl5000",
                 "fsl,imx-audio-sgtl5000";
    model = "imx51-babbage-sgtl5000";
    ssi-controller = <&ssi1>;
    audio-codec = <&sgtl5000>;
    audio-routing = "MIC_IN", "Mic Jack",
                    "Mic Jack", "Mic Bias",
                    "Headphone Jack", "HP_OUT";
[...]
};
```

从设备树中的路由期望音频映射以特定格式给出。也就是说，条目被解析为字符串对，第一个是连接的接收端，第二个是连接的源端。大多数情况下，这些连接被实现为编解码器引脚和板连接器映射。源和接收端的有效名称取决于硬件绑定，如下所示：

+   **编解码器**：这里应该已经定义了这里使用的引脚的名称。

+   **机器**：这里应该已经定义了这里使用的连接器或插孔的名称。

在前面的摘录中，你注意到了什么？我们可以看到`MIC_IN`、`HP_OUT`和`"Mic Bias"`，这些是编解码器引脚（来自编解码器驱动程序），以及`"Mic Jack"`和`"Headphone Jack"`，这些在机器驱动程序中被定义为板连接器。

为了使用在设备树中定义的路由，机器驱动程序必须调用`snd_soc_of_parse_audio_routing()`，它具有以下原型：

```
int snd_soc_of_parse_card_name(struct snd_soc_card *card,
                               const char *prop);
```

在前面的原型中，`card`代表解析路由的声卡，`prop`是包含设备树节点中路由的属性的名称。此函数在成功时返回`0`，在错误时返回负错误代码。

### 静态路由

静态路由包括从机器驱动程序定义 DAPM 路由映射，并直接分配给声卡，如下所示：

```
static const struct snd_soc_dapm_widget rk_dapm_widgets[] = {
    SND_SOC_DAPM_HP("Headphone", NULL),
    SND_SOC_DAPM_MIC("Headset Mic", NULL),
    SND_SOC_DAPM_MIC("Int Mic", NULL),
    SND_SOC_DAPM_SPK("Speaker", NULL),
};
/* Connection to the codec pin */
static const struct snd_soc_dapm_route rk_audio_map[] = {
    {"IN34", NULL, "Headset Mic"},
    {"Headset Mic", NULL, "MICBIAS"},
    {"DMICL", NULL, "Int Mic"},
    {"Headphone", NULL, "HPL"},
    {"Headphone", NULL, "HPR"},
    {"Speaker", NULL, "SPKL"},
    {"Speaker", NULL, "SPKR"},
};
static struct snd_soc_card snd_soc_card_rk = {
    .name = "ROCKCHIP-I2S",
    .owner = THIS_MODULE,
[...]
    .dapm_widgets = rk_dapm_widgets,
    .num_dapm_widgets = ARRAY_SIZE(rk_dapm_widgets),
    .dapm_routes = rk_audio_map,
    .num_dapm_routes = ARRAY_SIZE(rk_audio_map),
    .controls = rk_mc_controls,
    .num_controls = ARRAY_SIZE(rk_mc_controls),
};
```

上述片段摘自`sound/soc/rockchip/rockchip_rt5645.c`。通过这种方式使用它，就不需要使用`snd_soc_of_parse_audio_routing()`。然而，使用这种方法的一个缺点是，无法在不重新编译内核的情况下更改路由。接下来，我们将看一下时钟和格式考虑。

# 时钟和格式考虑

在深入研究本节之前，让我们花一些时间在`snd_soc_dai_link->ops`字段上。该字段是`struct snd_soc_ops`类型，定义如下：

```
struct snd_soc_ops {
    int (*startup)(struct snd_pcm_substream *);
    void (*shutdown)(struct snd_pcm_substream *);
    int (*hw_params)(struct snd_pcm_substream *,
                     struct snd_pcm_hw_params *);
    int (*hw_free)(struct snd_pcm_substream *);
    int (*prepare)(struct snd_pcm_substream *);
    int (*trigger)(struct snd_pcm_substream *, int);
};
```

这个结构中的回调字段应该让你想起了`snd_soc_dai_driver->ops`字段中定义的那些，它是`struct snd_soc_dai_ops`类型。在 DAI 链中，这些回调表示 DAI 链的机器级 PCM 操作，而在`struct snd_soc_dai_driver`中，它们要么是特定于编解码器 DAI 的，要么是特定于 CPU-DAI 的。

`startup()`在 PCM 子流打开时（当有人打开捕获/播放设备时）由 ALSA 调用，而`hw_params()`在设置音频流时调用。机器驱动程序可以在这两个回调中从 DAI 链配置 DAI 链数据格式。`hw_params()`具有接收流参数（*通道数*、*格式*、*采样率*等）的优势。

数据格式配置应该在 CPU DAI 和编解码器之间保持一致。ASoC 核心提供了帮助函数来更改这些配置。它们如下：

```
int snd_soc_dai_set_fmt(struct snd_soc_dai *dai,                         unsigned int fmt)
int snd_soc_dai_set_pll(struct snd_soc_dai *dai, int pll_id,
                        int source, unsigned int freq_in,
                        unsigned int freq_out)
int snd_soc_dai_set_sysclk(struct snd_soc_dai *dai, int clk_id,
                           unsigned int freq, int dir)
int snd_soc_dai_set_clkdiv(struct snd_soc_dai *dai,
                           int div_id, int div)
```

在上述帮助列表中，`snd_soc_dai_set_fmt`设置了 DAI 格式，例如时钟主/从关系、音频格式和信号反转；`snd_soc_dai_set_pll`配置了时钟 PLL；`snd_soc_dai_set_sysclk`配置了时钟源；`snd_soc_dai_set_clkdiv`配置了时钟分频器。这些帮助程序将调用底层 DAI 驱动程序 ops 中的适当回调。例如，使用 CPU DAI 调用`snd_soc_dai_set_fmt()`将调用此 CPU DAI 的`dai->driver->ops->set_fmt`回调。

以下是可以分配给 DAI 或`dai_link.format`字段的实际格式/标志列表：

+   `snd_soc_dai_set_fmt()`：

A) `SND_SOC_DAIFMT_CBM_CFM`: CPU 是位时钟和帧同步的从设备。这也意味着编解码器是两者的主机。

b) `SND_SOC_DAIFMT_CBS_CFS`。CPU 是位时钟和帧同步的主机。这也意味着编解码器对两者都是从设备。

c) `SND_SOC_DAIFMT_CBM_CFS`。CPU 是位时钟的从设备，帧同步的主机。这也意味着编解码器是前者的主机，后者的从设备。

B) `SND_SOC_DAIFMT_DSP_A`: 帧同步为 1 位时钟宽，1 位延迟。

b) `SND_SOC_DAIFMT_DSP_B`: 帧同步为 1 位时钟宽，0 位延迟。此格式可用于 TDM 协议。

c) `SND_SOC_DAIFMT_I2S`: 帧同步为 1 个音频字宽，1 位延迟，I2S 模式。

d) `SND_SOC_DAIFMT_RIGHT_J`: 右对齐模式。

e) `SND_SOC_DAIFMT_LEFT_J`: 左对齐模式。

f) `SND_SOC_DAIFMT_DSP_A`: 帧同步为 1 位时钟宽，1 位延迟。

g) `SND_SOC_DAIFMT_AC97`: AC97 模式。

h) `SND_SOC_DAIFMT_PDM`: 脉冲密度调制。

i) `SND_SOC_DAIFMT_DSP_B`: 帧同步为 1 位时钟宽，1 位延迟。

C) `SND_SOC_DAIFMT_NB_NF`: 正常位时钟，正常帧同步。CPU 发射器在位时钟的下降沿上移出数据，接收器在上升沿上采样数据。CPU 帧同步发生器在帧同步的上升沿上开始帧。建议在 CPU 端使用此参数进行 I2S。

b) `SND_SOC_DAIFMT_NB_IF`: 正常位时钟，反转帧同步。CPU 发射器在位时钟的下降沿上移出数据，接收器在上升沿上采样数据。CPU 帧同步发生器在帧同步的下降沿上开始帧。

c) `SND_SOC_DAIFMT_IB_NF`：反转位时钟，正常帧同步。CPU 发射器在位时钟的上升沿上移出数据，接收器在下降沿上采样数据。CPU 帧同步生成器在帧同步的上升沿上启动帧。

d) `SND_SOC_DAIFMT_IB_IF`：反转位时钟，反转帧同步。CPU 发射器在位时钟的上升沿上移出数据，接收器在下降沿上采样数据。CPU 帧同步生成器在帧同步的下降沿上启动帧。此配置可用于 PCM 模式（例如蓝牙或基于调制解调器的音频芯片）。

+   `snd_soc_dai_set_sysclk()`。以下是方向参数，让 ALSA 知道使用的时钟：

a) `SND_SOC_CLOCK_IN`：这意味着 sysclock 使用内部时钟。

b) `SND_SOC_CLOCK_OUT`：这意味着 sysclock 使用外部时钟。

+   `snd_soc_dai_set_clkdiv()`。

在`dai_link->dai_fmt`字段中设置或分配给机器驱动程序的编解码器或 CPU DAIs 的可能值如上所示。以下是典型的`hw_param()`实现：

```
static int foo_hw_params(struct snd_pcm_substream *substream,
                          struct snd_pcm_hw_params *params)
{
    struct snd_soc_pcm_runtime *rtd = substream->private_data;
    struct snd_soc_dai *codec_dai = rtd->codec_dai;
    struct snd_soc_dai *cpu_dai = rtd->cpu_dai;
    unsigned int pll_out = 24000000;
    int ret = 0;
    /* set the cpu DAI configuration */
    ret = snd_soc_dai_set_fmt(cpu_dai, SND_SOC_DAIFMT_I2S |
                              SND_SOC_DAIFMT_NB_NF |                               SND_SOC_DAIFMT_CBM_CFM);
    if (ret < 0)
        return ret;
    /* set codec DAI configuration */
    ret = snd_soc_dai_set_fmt(codec_dai, SND_SOC_DAIFMT_I2S |
                              SND_SOC_DAIFMT_NB_NF |                               SND_SOC_DAIFMT_CBM_CFM);
    if (ret < 0)
        return ret;
    /* set the codec PLL */
    ret = snd_soc_dai_set_pll(codec_dai, WM8994_FLL1, 0,
                          pll_out, params_rate(params) * 256);
    if (ret < 0)
        return ret;
    /* set the codec system clock */
    ret = snd_soc_dai_set_sysclk(codec_dai, WM8994_SYSCLK_FLL1,
                  params_rate(params) * 256, SND_SOC_CLOCK_IN);
    if (ret < 0)
        return ret;
    return 0;
}
```

在`foo_hw_params()`函数的上述实现中，我们可以看到编解码器和平台 DAI 是如何配置的，包括格式和时钟设置。现在我们来到机器驱动程序实现的最后一步，即注册音频声卡，这是系统上执行音频操作的设备。

# 声卡注册

声卡在内核中表示为`struct snd_soc_card`的实例，定义如下：

```
struct snd_soc_card {
    const char *name;
    struct module *owner;
    [...]
    /* callbacks */
    int (*set_bias_level)(struct snd_soc_card *,
                          struct snd_soc_dapm_context *dapm,
                          enum snd_soc_bias_level level);
    int (*set_bias_level_post)(struct snd_soc_card *,
                             struct snd_soc_dapm_context *dapm,
                             enum snd_soc_bias_level level);
    [...]
    /* CPU <--> Codec DAI links	*/
    struct snd_soc_dai_link *dai_link;
    int num_links;
    const struct snd_kcontrol_new *controls;
    int num_controls;
    const struct snd_soc_dapm_widget *dapm_widgets;
    int num_dapm_widgets;
    const struct snd_soc_dapm_route *dapm_routes;
    int num_dapm_routes;
    const struct snd_soc_dapm_widget *of_dapm_widgets;
    int num_of_dapm_widgets;
    const struct snd_soc_dapm_route *of_dapm_routes;
    int num_of_dapm_routes;
[...]
};
```

为了便于阅读，仅列出了相关字段，完整定义可以在[`elixir.bootlin.com/linux/v4.19/source/include/sound/soc.h#L1010`](https://elixir.bootlin.com/linux/v4.19/source/include/sound/soc.h#L1010)找到。也就是说，以下列表描述了我们列出的字段：

+   `name`是声卡的名称。

+   `owner`是此声卡的模块所有者。

+   `dai_link`是构成此声卡的 DAI 链接数组，`num_links`指定数组中的条目数。

+   `controls`是一个包含由机器驱动程序静态定义和设置的控件的数组，`num_controls`指定数组中的条目数。

+   `dapm_widgets`是一个包含由机器驱动程序静态定义和设置的 DAPM 小部件的数组，`num_dapm_widgets`指定数组中的条目数。

+   `damp_routes`是一个包含由机器驱动程序静态定义和设置的 DAPM 路由的数组，`num_dapm_routes`指定数组中的条目数。

+   `of_dapm_widgets`表示从 DT（通过`snd_soc_of_parse_audio_simple_widgets()`）提供的 DAPM 小部件，`num_of_dapm_widgets`是小部件条目的实际数量。

+   `of_dapm_routes`表示从 DT（通过`snd_soc_of_parse_audio_routing()`）提供的 DAPM 路由，`num_of_dapm_routes`是路由条目的实际数量。

在设置完声卡结构之后，可以通过机器使用`devm_snd_soc_register_card()`方法进行注册，其原型如下：

```
int devm_snd_soc_register_card(struct device *dev,
                               struct snd_soc_card *card);
```

在上述原型中，`dev`表示用于管理卡的基础设备，`card`是先前设置的实际声卡数据结构。此函数在成功时返回`0`。但是，当调用此函数时，将会探测每个组件驱动程序和 DAI 驱动程序。因此，将为每个成功探测到的 DAI 链接创建一个新的 PCM 设备。

以下摘录（来自 Rockchip 机器 ASoC 驱动程序，用于使用 MAX90809 CODEC 的板子，实现在内核源码中的`sound/soc/rockchip/rockchip_max98090.c`中）将展示整个声卡的创建过程，从小部件到路由，再到 DAI 链接配置。让我们首先定义这个机器的小部件和控件，以及用于配置 CPU 和编解码器 DAI 的回调函数：

```
static const struct snd_soc_dapm_widget rk_dapm_widgets[] = { 
    [...]
};
static const struct snd_soc_dapm_route rk_audio_map[] = {
    [...]
};
static const struct snd_kcontrol_new rk_mc_controls[] = {
    SOC_DAPM_PIN_SWITCH("Headphone"),
    SOC_DAPM_PIN_SWITCH("Headset Mic"),
    SOC_DAPM_PIN_SWITCH("Int Mic"),
    SOC_DAPM_PIN_SWITCH("Speaker"),
};
static const struct snd_soc_ops rk_aif1_ops = {
    .hw_params = rk_aif1_hw_params,
};
static struct snd_soc_dai_link rk_dailink = {
    .name = "max98090",
    .stream_name = "Audio",
    .codec_dai_name = "HiFi",
    .ops = &rk_aif1_ops,
    /* set max98090 as slave */
    .dai_fmt = SND_SOC_DAIFMT_I2S | SND_SOC_DAIFMT_NB_NF |
                 SND_SOC_DAIFMT_CBS_CFS,
};
```

在上面的摘录中，可以看到原始代码实现文件中的`rk_aif1_hw_params`。现在是用于构建声卡的数据结构，定义如下：

```
static struct snd_soc_card snd_soc_card_rk = {
    .name = "ROCKCHIP-I2S",
    .owner = THIS_MODULE,
    .dai_link = &rk_dailink,
    .num_links = 1,
    .dapm_widgets = rk_dapm_widgets,
    .num_dapm_widgets = ARRAY_SIZE(rk_dapm_widgets),
    .dapm_routes = rk_audio_map,
    .num_dapm_routes = ARRAY_SIZE(rk_audio_map),
    .controls = rk_mc_controls,
    .num_controls = ARRAY_SIZE(rk_mc_controls),
};
```

这个声卡最终是在驱动程序的`probe`方法中创建的：

```
static int snd_rk_mc_probe(struct platform_device *pdev)
{
    int ret = 0;
    struct snd_soc_card *card = &snd_soc_card_rk;
    struct device_node *np = pdev->dev.of_node;
[...]
    card->dev = &pdev->dev;
    /* Assign codec, cpu and platform node */
    rk_dailink.codec_of_node = of_parse_phandle(np,
                                  "rockchip,audio-codec", 0);
    rk_dailink.cpu_of_node = of_parse_phandle(np,
                                "rockchip,i2s-controller", 0);
    rk_dailink.platform_of_node = rk_dailink.cpu_of_node;
[...]
    ret = snd_soc_of_parse_card_name(card, "rockchip,model");
    ret = devm_snd_soc_register_card(&pdev->dev, card);
[...]
}
```

再次强调，前面三个代码块都是从`sound/soc/rockchip/rockchip_max98090.c`中摘录的。到目前为止，我们已经了解了机器驱动程序的主要目的，即将 Codec 和 CPU 驱动程序绑定在一起，并定义音频路径。也就是说，有时我们可能需要更少的代码。这些情况涉及到既不需要特殊黑客的 CPU 也不需要特殊黑客的板子。在这种情况下，ASoC 框架提供了**simple-card 机器驱动程序**，在下一节中介绍。

# 利用 simple-card 机器驱动程序

有时候，您的板子不需要来自 Codec 或 CPU DAI 的任何黑客。ASoC 核心提供了`simple-audio`机器驱动程序，可以用来从 DT 描述整个声卡。以下是这样一个节点的摘录：

```
sound {
    compatible ="simple-audio-card";
    simple-audio-card,name ="VF610-Tower-Sound-Card";
    simple-audio-card,format ="left_j";
    simple-audio-card,bitclock-master = <&dailink0_master>;
    simple-audio-card,frame-master = <&dailink0_master>;
    simple-audio-card,widgets ="Microphone","Microphone Jack",
                               "Headphone","Headphone Jack",                              
                               "Speaker","External Speaker";
    simple-audio-card,routing = "MIC_IN","Microphone Jack",
                                "Headphone Jack","HP_OUT",
                                "External Speaker","LINE_OUT";
    simple-audio-card,cpu {
        sound-dai = <&sh_fsi20>;
    };
    dailink0_master: simple-audio-card,codec {
        sound-dai = <&ak4648>;
        clocks = <&osc>;
    };
};
```

这完全记录在`Documentation/devicetree/bindings/sound/simple-card.txt`中。在上面的摘录中，我们可以看到指定了机器小部件和路由映射，以及引用的编解码器和 CPU 节点。现在我们熟悉了 simple-card 机器驱动程序，我们可以利用它，并尽可能地不编写自己的机器驱动程序。话虽如此，有些情况下编解码器设备无法分离，这改变了机器应该编写的方式。这样的音频设备称为无编解码器声卡，在下一节中我们将讨论它们。

## 无编解码器声卡

可能会出现从外部系统采样数字音频数据的情况，例如使用 SPDIF 接口时，数据因此被预格式化。在这种情况下，声卡注册是相同的，但 ASoC 核心需要意识到这种特殊情况。

通过输出，DAI 链接对象的`.capture_only`字段应该是`false`，而`.playback_only`应该是`true`。输入应该做相反的操作。此外，机器驱动程序必须将 DAI 链接的`codec_dai_name`和`codec_name`设置为`"snd-soc-dummy-dai"`和`"snd-soc-dummy"`。例如，这是`imx-spdif`机器驱动程序（`sound/soc/fsl/imx-spdif.c`）的情况，其中包含以下摘录：

```
data->dai.name = "S/PDIF PCM";
data->dai.stream_name = "S/PDIF PCM";
data->dai.codecs->dai_name = "snd-soc-dummy-dai";
data->dai.codecs->name = "snd-soc-dummy";
data->dai.cpus->of_node = spdif_np;
data->dai.platforms->of_node = spdif_np;
data->dai.playback_only = true;
data->dai.capture_only = true;
if (of_property_read_bool(np, "spdif-out"))
    data->dai.capture_only = false;
if (of_property_read_bool(np, "spdif-in"))
    data->dai.playback_only = false;
if (data->dai.playback_only && data->dai.capture_only) {
    dev_err(&pdev->dev, "no enabled S/PDIF DAI link\n");
    goto end;
}
```

您可以在`Documentation/devicetree/bindings/sound/imx-audio-spdif.txt`中找到此驱动程序的绑定文档。在机器类驱动程序研究结束时，我们已经完成了整个 ASoC 类驱动程序的开发。在这个机器类驱动程序中，除了在代码中绑定 CPU 和 Codec 以及提供设置回调之外，我们还看到了如何通过使用 simple-card 机器驱动程序并在设备树中实现其余部分来避免编写代码。

# 总结

在本章中，我们已经了解了 ASoC 机器类驱动程序的架构，这代表了 ASoC 系列中的最后一个元素。我们已经学会了如何绑定平台和子设备驱动程序，以及如何为音频数据定义路由。

在下一章中，我们将介绍另一个 Linux 媒体子系统，即 V4L2，用于处理视频设备。
