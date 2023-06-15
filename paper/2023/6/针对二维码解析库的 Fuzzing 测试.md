# 针对二维码解析库的 Fuzzing 测试
**作者：evilpan  
原文链接：[https://mp.weixin.qq.com/s/w6und9w0CAlcISrrJX4vnA](https://mp.weixin.qq.com/s/w6und9w0CAlcISrrJX4vnA)**

### 背景

在四月份的时候出了那么一个新闻，说微信有一个点击图片就崩溃的 bug，当时各大微信群里都在传播导致手机各种闪退。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/6/eeea7052-0c2c-4eea-978e-f0cb870b3c7f.png?raw=true)

由于当时笔者正在忙着卷 Java Web，没有第一时间去蹭这个热点，不过当时也稍微了解了一下 crash 的原理。最近对于 Java 的漏洞挖掘正好告一段落，于是就突然想起了这个事。

这个 bug 的根源在于一个畸形的二维码解码时导致的空指针错误，详细分析和修复的记录可以参考 [fix(wechat_qrcode): Init nBytes after the count value is determined #3480](https://github.com/opencv/opencv_contrib/pull/3480)。

在国内二维码可以说是随处可见的东西，如果说解码的核心 SDK 有漏洞，那么很可能影响非常广泛。加上许多应用，基于各种奇怪的业务和目的还实现了静默解码的功能，要是能够发现一个可以利用的漏洞，那也是一个典型的 0-click 场景。

因此笔者打算简单 Fuzz 一下相关的二维码解码代码，放在服务器后台慢慢跑着，也不会过多占用搬砖时间，希望哪天清晨 boki 醒来能够有些意外收获。

### 编译

Fuzz 的第一步自然是先能成功编译目标代码，然后再尝试使用带插桩的编译器进行编译。

之前出现问题的代码仓库是 [https://github.com/opencv/opencv_contrib](https://github.com/opencv/opencv_contrib)，这是 opencv 下的一个子仓库，提供了许多额外的 OpenCV 模块，根据 README 的介绍，编译这些模块的过程如下:

```
$ cd <opencv_build_directory>
$ cmake -DOPENCV_EXTRA_MODULES_PATH=<opencv_contrib>/modules <opencv_source_directory>
$ make -j5
```

也就是说编译这些模块还需要依赖 OpenCV 的源代码仓库，因此我们把这两个仓库都下载下来进行编译。这里图省事我就直接使用 AFL++ 进行编译了，使用 cmake 的编译命令如下:

```
cmake -DCMAKE_C_COMPILER=afl-clang-fast -DCMAKE_CXX_COMPILER=afl-clang-fast++ \
  -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -fno-sanitize-recover=all -g" \
  -DCMAKE_C_FLAGS="-fsanitize=address,undefined -fno-sanitize-recover=all -g" \
  -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=address,undefined -fno-sanitize-recover=all" \
  -DCMAKE_MODULE_LINKER_FLAGS="-fsanitize=address,undefined -fno-sanitize-recover=all" \
  -DCMAKE_BUILD_TYPE=Debug,ASAN,UBSAN -DWITH_SSE2=ON -DMONOLITHIC_BUILD=ON -DBUILD_SHARED_LIBS=OFF \
  -DOPENCV_EXTRA_MODULES_PATH=/home/evilpan/fuzzing/opencv_contrib/modules \
  /home/evilpan/fuzzing/opencv

make -j128 -C modules/wechat_qrcode
```

关键点在于不编译动态库，而是使用静态库，因为 AFL 对于最终的测试程序只会测试静态编译代码中的覆盖率而不会考虑动态编译的库。

后来发现 cmake 可以不用指定那么多选项，只需要指定 afl-clang，config 之后在 make 时通过环境变量指定 `AFL_USE_ASAN` 即可，AFL 会选择好合适的编译参数。参考:[https://aflplus.plus/docs/fuzzing\_in\_depth/](https://aflplus.plus/docs/fuzzing_in_depth/)

```
$ cmake -DCMAKE_C_COMPILER=afl-clang-fast -DCMAKE_CXX_COMPILER=afl-clang-fast++ -DBUILD_SHARED_LIBS=OFF
$ env AFL_USE_ASAN=1 make -j32
```

### 小试牛刀

编译完后就可以编写测试代码了。不过根据前面的 pull-request，在修复漏洞时也提供了一个 testcase:

```
TEST(Objdetect_QRCode_bug, issue_3478) {
    auto detector = wechat_qrcode::WeChatQRCode();
    std::string image_path = findDataFile("qrcode/issue_3478.png");
    Mat src = imread(image_path, IMREAD_GRAYSCALE);
    ASSERT_FALSE(src.empty()) << "Can't read image: " << image_path;
    std::vector<std::string> outs = detector.detectAndDecode(src);
    ASSERT_EQ(1, (int) outs.size());
    ASSERT_EQ(16, (int) outs[0].size());
    ASSERT_EQ("KFCVW50         ", outs[0]);
}
```

可以参考这个代码来编写我们的 test-harness。我这里是直接改掉了 `test_main.cpp`，不使用单元测试框架而是直接自己编写，一个初始的 test-harness 代码如下:

```
int main(int argc, char **argv) {
  if (argc < 2) return 1;
  std::string image_path(argv[1]);
  Mat src = imread(image_path, IMREAD_GRAYSCALE);
  if (!src.empty()) {
    auto detector = wechat_qrcode::WeChatQRCode();
    std::vector<std::string> outs = detector.detectAndDecode(src);
  }
  return 0
}
```

修改后只需要重新编译这个模块即可:

```
make -C modules/wechat_qrcode
```

生成的测试二进制文件为 `build/bin/opencv_test_wechat_qrcode`:

```
afl-fuzz -i corpus/ -o output -t +2000 -- ./opencv_test_wechat_qrcode @@
```

`corpus` 中随便丢了几张畸形二维码文件，直接开跑！

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/6/5f4ff198-37fd-49c1-bfda-b7267f07a41e.png?raw=true)

虽然能跑，但是慢的一批！

### 初步优化

上面的的 fuzz 代码实在太慢，必须要进行优化！组织架构调整！把寒气也传递给每一个 fuzzer！

由于前面的的运行是直接读取文件，因此一个直观的优化是使用 AFL++ 的 [persistent mode](https://github.com/AFLplusplus/AFLplusplus/blob/ed73c632a5791ca740fe64770b6d238206033ec4/instrumentation/README.persistent_mode.md)，稍微修改一下代码，直接把 buffer 当做源图片进行读取，如下所示:

```
int fuzz_buf(unsigned char *buf, size_t size) {
  Mat src = imdecode(Mat(1, size, CV_8UC1, buf), IMREAD_GRAYSCALE);
  if (!src.empty()) {
    auto detector = wechat_qrcode::WeChatQRCode();
    std::vector<std::string> outs = detector.detectAndDecode(src);
    return outs.size();
  }
  return -1;
}

__AFL_FUZZ_INIT();

int main(int argc, char **argv) {
  #ifdef __AFL_HAVE_MANUAL_CONTROL
  __AFL_INIT();
#endif
  unsigned char *buf = __AFL_FUZZ_TESTCASE_BUF;
  while (__AFL_LOOP(10000)) {
    int len = __AFL_FUZZ_TESTCASE_LEN;
    fuzz_buf(buf, len);
  }
  return 0;
}
```

这次执行就不需要 `@@` 了，可以直接运行:

```
afl-fuzz -i corpus/ -o output -t +2000 -- ./opencv_test_wechat_qrcode_persist
```

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/6/77c27b78-1562-43e4-918f-7fde9ac0549b.png?raw=true)

可以看到速度已经有了显著的提升！并且跑了十几分钟后就已经出现了 Crash！

### 再次优化

先暂停前面的 fuzzer 来分析一下现有的 crash，执行出现的崩溃都大致如下所示:

```
$ ./opencv_test_wechat_qrcode output/default/crashes/id\:000001\,sig\:06\,src\:000478\,time\:871148\,execs\:98951\,op\:havoc\,rep\:4
terminate called after throwing an instance of 'cv::Exception'
  what():  OpenCV(4.7.0-dev) /home/jielv/fuzzing/opencv/modules/imgcodecs/src/loadsave.cpp:74: error: (-215:Assertion failed) size.width > 0 in function 'validateInputImageSize'

Aborted
```

其实，我对你是有一些失望的。当初把你写出来，是希望你能对标古河的。我是希望你跑起来后，你能够拼一把，快速给我几个 segment fault。对于你这个层级，不是丢出几个 Abort 就可以的。

虽然也有我 `-g` 的原因，但这里的 Abort 实际上都出现在上面的 `imdecode` 中，这跟我的预期有点跑偏，明明想跑的是二维码解码，结果却在无关的地方运行了半天。虽然 image codec 也是一个攻击面，但并不是现在想要搞的，而且这个攻击面肯定也被很多其他人搞过了，出现问题的几率并不是很大。

因此，我们需要进一步优化代码。现在 AFL 变异的是 jpg/png 的图片，但我们想要变异的其实是二维码的图片内容！因此这次优化有两个思路:

1.  提供自定义的变异策略，每次返回一个变异的二维码；
2.  还是让 AFL 自己变异，但是变异后的内容直接转成 `cv::Mat` 然后进行二维码解码；

不管使用哪个思路，都需要能够将图片和 `cv::Mat` 进行互相转换，否则即便有 crash 也可能不能生成合法的图片导致无法利用。

#### Mat2Vector

一开始想着偷个懒，直接让 ChatGPT 来帮我生成代码将 Mat 转成字符数组，如下所示:

```
std::vector<char> mat_to_vector(const cv::Mat& mat) {
       std::vector<char> data(mat.data, mat.data + mat.total() * mat.elemSize());
    return data;
}
```

然后，将原来的预料图片用 imread 读取成 `cv::Mat`，并用上面的函数转成 vector 并保存到磁盘中，形成新的预料，这样在变异的时候可以直接变异并生成 `cv::Mat`:

```
int fuzz_buf(unsigned char *buf, size_t size) {
   Mat src = Mat(std::vector<unsigned char>(buf, buf + size));
  return fuzz_mat(src);
}
```

理想很美好，但是这样跑并没有结果！猜测是 Mat 中除了 data 数据，应该还有一些元数据(metadata)，用以表示矩阵的特性，比如行/列，直接填入数据无法表示原始的矩阵，从而在二维码解码时很早就出错退出了。

#### cv::Mat

既然偷懒走不通，就只能认真看一下 Mat 了。

[cv::Mat](https://docs.opencv.org/4.x/d3/d63/classcv_1_1Mat.html) 是 OpenCV 中用于表示 n 维数组的数据结构，用于表示 n 维的单通道或者多通道数组，通常是结构比较紧凑的矩阵。对于稀疏数据的高维矩阵则一般用 `SparseMat` 来进行表示。

对于矩阵/数组 `M` 而言，其数据布局根据 `data` 和 `step` 决定，例如对于二维数组:

```
M.at(i,j) = M.data + M.step[0] * i + M.step[1] * j
```

注意 `M.step[i] >= M.step[i+1]`，事实上 `M.step[i] >= M.step[i+1]*M.size[i+1]`。这意味着对于二维矩阵而言，数据是按照行进行存储的(row-by-row)，而对于三维矩阵数据则是按面进行存储(plane-by-plane)。

我们的目标是创建一个代表二维码图片的 Mat，最好是能够保存到磁盘中并从磁盘读取，方便我们使用 `afl-fuzz` 指定语料并进行 fuzz。于是想着有没有什么序列化/反序列化是针对 `cv::Mat` 的，查了一下还真有，序列化的代码如下:

```
cv::Mat img = cv::imread("example.jpg");cv::FileStorage fs("example.yml", cv::FileStorage::WRITE);fs << "image" << img;fs.release();
```

反序列化过程和序列化过程类似:

```
cv::Mat img;

cv::FileStorage fs("example.yml", cv::FileStorage::READ);fs["image"] >> img;fs.release();
```

这里是将 `cv::Mat` 序列化成了 YAML 数据，输出的格式如下:

```
%YAML:1.0
---
img: !!opencv-matrix
   rows: 480
   cols: 640
   dt: u
   data: [ 103, 103, 104, 107, 110, 112, 112, 111, 114, 113, 113, 115,
           ...
           134, 135, 134, 132 ]
```

如果将 YAML 文件作为语料的话，又涉及了 YAML 的解析以及 FileStorage 中的无关代码，和需求有所差距。不过从序列化的数据中我们能够看出一点，即 `cv::Mat` 中除了 `data` 数据外包含的额外元数据只有 rows、cols 和 dt。其中 `dt` 应该表示 data type，`u` 是 `CV_8U` 的缩写。

由于这是一个黑白的，480x640 的二维码图片，如果我们想要变异 `cv::Mat`，只需要保存行/列/类型不变，针对 `data` 区域进行 bitflip 变异即可！

不过，只进行 bitflip 变异的空间将会高达 `2^(640*480)`，无疑是个几何数据，即便是只针对小尺寸图片如 `20x20` 那也是不能接受的。因此我们还是寄希望于 AFL 的覆盖率反馈能产生合适的变异！

#### First Blood

使用上述策略优化之后裁剪语料的长度，仅使用一个 `29x29` 的二维码图片生成的矩阵作为初始语料进行变异:

```
int fuzz_buf(void *buf, size_t size) {
   if (size < 841) return -1;
  Mat src = Mat(29, 29, CV_8U, buf);
  return fuzz_mat(src);
}
```

跑起来之后发现速度又变得很慢，估计语料还是太大，先放在后台跑一会，然后去看了两集异世界厕纸番。回来之后发现居然有 crash:

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/6/83c17bc8-06a0-473e-adfa-789d5cbebc0e.png?raw=true)

简单分析一下发现都是同一处代码的 crash，精简的 poc 执行输出如下:

```
$ opencv_test_wechat_qrcode poc.bin
/home/evilpan/fuzzing/opencv_contrib/modules/wechat_qrcode/src/zxing/qrcode/detector/detector.cpp:1022:50: runtime error: signed integer overflow: -2147483648 + -2147483648 cannot be represented in type 'int'
```

这是一个整数溢出，出现在 qrcdoe 检测过程中计算维度时:

```
int Detector::computeDimension(Ref<ResultPoint> topLeft, Ref<ResultPoint> topRight,
                               Ref<ResultPoint> bottomLeft, float moduleSizeX, float moduleSizeY) {
    int tltrCentersDimension = ResultPoint::distance(topLeft, topRight) / moduleSizeX;
    int tlblCentersDimension = ResultPoint::distance(topLeft, bottomLeft) / moduleSizeY;

    float tmp_dimension = ((tltrCentersDimension + tlblCentersDimension) / 2.0) + 7.0;    int dimension = cvRound(tmp_dimension);
    int mod = dimension & 0x03; 
    switch (mod) {         case 0:
            dimension++;
            break;
                   case 2:
            dimension--;
            break;
    }
    return dimension;
}
```

计算锚点时 `tltrCentersDimension + tlblCentersDimension` 产生了整数溢出，PoC 如下:

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/6/e71d5896-71a5-4e0a-a286-079435397b9d.png?raw=true)

当然，这个 PoC 直接扫码是扫不出来的，因为我们传入的是原始数据，而且是死在了 detector 之中。

#### 画蛇添足

虽然现在已经可以产生一些崩溃，但注意到图像并不是二值图像而是灰度图，因为我们原始的语料是 `Mat.data()` 保存而成的二进制数据，这个数据中每个像素还是 8 bit 的，只不过只有两种值，分别是 0 和 255，而我们变异过程中就成了灰度图。

为了解决这个问题，我们只能重新构建序列化的数据结构，确保每个字节的变异都能产生有效的二值图像。图像中每个像素只占 1 bit，由于内存是字节寻址的，每个字节可以保存 8 个像素。

写了个 `MatWrapper` 来封装 cols、rows、type 和 data 数据，并添加 serialize/deserialize 方法去实现序列化:

```
class MatWrapper {
public:
  int mRows;
  int mCols;
  int mType;
  int mSize;
  std::vector<uchar> mBuf;

  MatWrapper() {}

  void save(const std::string& filename, bool compact=false) {
    std::ofstream fs(filename);
    fs << serialize(compact);
    fs.close();
  }

  void load(const std::string& filename, bool compact=false) {
    std::stringstream ss;
    std::ifstream file(filename, std::ios::binary);
    ss << file.rdbuf();
    deserialize(ss.str(), compact);
  }

  void fromImage(const std::string& filename) {
    auto mat = imread(filename, IMREAD_GRAYSCALE);
    mRows = mat.rows;
    mCols = mat.cols;
    mType = mat.type();
       mSize = mat.total() * mat.elemSize();
    mBuf = std::vector<uchar>(mat.data, mat.data + mSize);
  }

  void toImage(const std::string& filename) {
    imwrite(filename, get());
  }

  Mat get() { return Mat(mRows, mCols, mType, mBuf.data()); }

  std::string serialize(bool compact = false) const { }
  bool deserialize(const std::string& buf, bool compact = false) { }
 };
```

`compact` 表示是否将 8 像素压缩为 1 字节，完整代码比较长这里就不贴了。写完之后先把之前的二维码图片语料批量转换为了自定义序列化的数据，然后使用 AFL 指定新的语料进行变异和 fuzz。一开始跑出来的都是我自己代码的 bug (囧)，挨个修完之后终于可以很快跑出上面的整数溢出了，而这次的 PoC 也是真正的黑白二维码图片。

但是！新的测试用例跑起来也是非常之慢，可能是反序列化的时候做了太多变换，导致速度其实和使用标准的压缩图片格式去跑差不多，总而言之感觉有点白给，但还是让它先跑一会。

### 二维码

在把上面的 Fuzzer 放在后台运行的时，我也想清楚了这个测试用例的问题所在。当前变异的策略虽然能够生成二值的 Bitmap 图像，但并不总是合法的二维码，所以代码覆盖率始终在 `detect` 阶段过不去而没有执行到 `decode`。

一个理想的策略是每次都能生成随机的、合法的二维码，并且在点阵的基础上进行变异。为此，我们需要先了解二维码的实现方式。

#### 基本结构

一个二维码通常由于多个部分组成，如下所示:

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/6/6025ae1f-4581-4875-9e8f-02bcf7ca372b.png?raw=true)

> 此外还有 M3 Micro QR Code，只包含一个定位点，不过比较少用，微信也扫不出来，这里先不介绍。

各个部分的简单介绍如下:

*   **Quiet Zone**: 二维码周围的白边，底色需要与二维码的颜色不同防止影响解析；
*   **Postion Detection Patterns**: 位置检测模式，也称为定位点，用于定位二维码；使用点格(module)表示外围黑边大小是 `7x7`，中间的黑块大小是 `3x3`；
*   **Separators for Postion Detection Patterns**: 定位点周围的分隔线，线宽为一个点格；
*   **Timing Patterns**: 时序模式，用于定位 x/y 轴，二维码弯曲时可以通过这两条线的弯曲去进行一定程度的修复；
*   **Alignment Patterns**: 对齐模式，在 Version 2 及以后才有，用于进一步辅助对齐。每个对齐模式的大小是黑格 `5x5`，白格 `3x3`，中间黑点占一格。对齐模式的个数和 Version 有关；
*   **Format Infomation**: 格式信息，占 30 个点格，两边各占 15 个，内容相同，互为备份；
*   **Version Information**: 版本信息，也是有两个互为备份的区块，各占 `3x6` 大小；
*   **Data and Error Correction Codewords**: 数据区，包含编码的数据正文以及纠错信息；

二维码的最小单位是点格(module)，一个二维码边长所占点格的大小称为二维码的维度(Dimension)，维度和版本有关:

```
Dimension = Version * 4 + 17
```

例如当 Version 等于 2 时，边长就是 25 个点格:

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/6/469e8b84-303f-4285-80c9-22d2ff508b7c.png?raw=true)

`Format Information` 区包含了纠错级别等信息，前面说过一共 15 点格，包括:

*   纠错等级: 2 点格，即 4 个纠错等级；
*   数据掩码: 3 点格，即 8 种掩码类型；
*   纠错码: 10 点格；

纠错码计算使用 BCH Code，用到的 Code Words 就保存在前面说过的数据区。

#### 变异策略

了解二维码的基础知识之后，我们至少知道了在一个二维码中哪些能变哪些不能变。二维码的编码是相当复杂的，虽然我们可以使用现有的编码策略去生成二维码，但是这会导致我们错过可能由于编码问题导致的漏洞。

秉承着先跑起来再说的原则，三下五除二用 Python 写了一个非常丑陋的原型，首先是生成一个随机的二维码:

```
def random_qrcode() -> qrcode.QRCode:
    version = randint(1, 40)
    box_size = randint(1, 8)
    border = 1
    error_correction = randint(0, 3)
    mask_pattern = randint(0, 7)
    data = randstr(randint(1, 25))

    qr = qrcode.QRCode(
        version=version,
        error_correction=error_correction,
        box_size=box_size,
        border=border,
        image_factory=None,
        mask_pattern=mask_pattern)
    qr.add_data(data)
    qr.make()
    return qr
```

`make` 之后 qrcode 的 modules 已经生成了，所以我们在其基础上进行变异:

```
def mutate(qr: qrcode.QRCode, add_buf):
    if not add_buf or len(add_buf) == 1:
        # do not mutate
        return
    # flip modules in qrcode randomly
    width = len(qr.modules)
    rng = random.Random()
    flipped = set()
    for i in range(0, len(add_buf) - 1, 2):
        val = add_buf[i] + (add_buf[i+1] << 8)
        if val in flipped:
            continue
        flipped.add(val)
        rng.seed(val)
        x = rng.randint(0, width - 1) # col
        y = rng.randint(0, width - 1) # row
        # print("flip", (x, y))
        # avoid position patterns, 7x7 and 1 space
        if x < 8 and y < 8:
            continue
        if x < 8 and y >= (width - 8):
            continue
        if x >= (width - 8) and y < 8:
            continue
        qr.modules[y][x] = not qr.modules[y][x]
```

`qr.modules` 是表示点格的二维数组，我们的变异策略非常简单粗暴，就是直接基于输入的数据对点格进行随机的翻转，黑的变白的，白的变黑的。几个小优化:

1.  使用 Random 来将 0xffff 大小的随机数 scale 到宽度 width 的范围；
2.  对于已经翻转过的点格就不用再次翻转了；
3.  随机翻转的范围排除掉三个角落的定位标志区；

#### Custom Mutators

现在我们已经可以生成随机的二维码图片了！接下来就需要将其与 Fuzzer 进行结合。在官方文档 [Custom Mutators in AFL++](https://aflplus.plus/docs/custom_mutators/) 中提到，我们可以编译一个 `.so` 并通过 `AFL_CUSTOM_MUTATOR_LIBRARY` 环境变量给 AFL++ 提供自定义的变异器，而且也可以使用 Python API 来提供变异器。

因此我们的 mutator 如下即可:

```
def fuzz(buf, add_buf, max_size):
    """
    Called per fuzzing iteration.
    """
    qr = random_qrcode()
    mutate(qr, add_buf)
    img = qr.make_image()
    stream = BytesIO()
    img.save(stream, "png")
    ret = stream.getvalue()
    if len(ret) > max_size:
        return bytearray(buf)
    return bytearray(ret)
```

> 注: `fuzz` 接口的参数 buf 一般包含原始 corpus 的内容，add\_buf 则是变异后返回的内容，这里是 PNG 数据。所以前面想把 add\_buf 作为变异源来修改 modules 的想法其实是有问题的，当时误以为 add_buf 是随机数据但熵根本不够，因此后面直接把输入数据忽略了，全部随机生成。

将其保存为 `random_qrcode.py` 并使用下面的命令去启动:

```
env AFL_CUSTOM_MUTATOR_ONLY=1 PYTHONPATH=$PWD/mutators AFL_PYTHON_MODULE=random_qrcode afl-fuzz -i corpus/ -o output -t +9000 -- ./opencv_test_wechat_qrcode_persist2
```

这里我指定了 `AFL_CUSTOM_MUTATOR_ONLY=1`，不需要 AFL 自身的变异，因为 AFL 的变异是基于原始语料的，而我并不希望在原始的图片二进制层级进行变异，否则就又回到了最初的场景。现在我们每次生成一张随机的二维码图片，且肯定是合法的 PNG 格式，避免遭遇图片解码时的异常。初始语料我也设置为表示坐标的点而不是图片。

当然，这个 fuzzer 跑起来巨慢！这是因为我们每次生成时都需要先生成一张二维码，变异，然后编码成 PNG 图片，再再将图片输入给目标进行解析。

解决方案可以通过 C++ 直接去生成二维码，然后将二维码直接转成 `cv::Mat` 去作为输入。这样一方面可以节省掉 PNG 编解码的过程，另一方面也可以摆脱 Python 的依赖。[QR-Code-generator](https://github.com/nayuki/QR-Code-generator) 也许是一个不错的方案。不过考虑到之前画蛇添足的失败尝试，能带来多少速度的提升也不甚确定。

另外当前覆盖率反馈的变异其实是有限的(如果有的话)，从文档中来看，[LibFuzzer](https://github.com/google/fuzzing/blob/master/docs/structure-aware-fuzzing.md) 对于自定义变异的支持可能会更完善一些。而且 LibFuzzer 可以天然支持多核，不用像 AFL 要自己手打一堆 slave。

Anyway，最终运行结果:

```
$ afl-whatsup -s output/
/usr/local/bin/afl-whatsup status check tool for afl-fuzz by Michal Zalewski

Summary stats
=============

       Fuzzers alive : 17
      Total run time : 32 days, 13 hours
         Total execs : 8 millions, 198 thousands
    Cumulative speed : 36 execs/sec
       Average speed : 2 execs/sec
       Pending items : 567 faves, 22476 total
  Pending per fuzzer : 33 faves, 1322 total (on average)
       Crashes saved : 0
         Hangs saved : 0
Cycles without finds : 0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0
  Time without finds : 35 seconds
```

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/6/d4d72bff-f0d8-466f-859e-56e67876a2eb.jpeg?raw=true)

好吧，虽然 OpenCV "非常安全"，但是二维码解析库又不止这一个！于是又找了另外一个常用的解析库 ZXing 去进行测试，事实证明还是可以找出问题的！

*   [heap-buffer-overflow in ZXing::DataMatrix::DMRegressionLine::modules #572](https://github.com/zxing-cpp/zxing-cpp/issues/572)

### 总结

本文可以看做是一个典型的 Fuzzing 漏洞挖掘心路历程。即先用 Generic Fuzzer 跑起来，然后通过不断深入理解代码，改良变异策略，最终变成 Structure Aware Fuzzing 的结果！虽然 AFL 一把梭能让机器为自己打工很爽，但是想得到更好的结果还是免不了动手优化。而不看代码的话可能即便找到问题也无法理解成因，轻则无法编写利用导致 award-0，重则提交错误的 patch 导致后续被其他开发者 revert 并批判一番钉在历史的耻辱柱上。因此，自动化测试和代码审计相结合往往才能达到更加有效的漏洞挖掘效果。

### 参考链接

*   [AFL++ Document](https://aflplus.plus/)
*   [How to create QRcode](https://www.swetake.com/qrcode/qr1_en.html)
*   [QR code - Wikipedia](https://en.wikipedia.org/wiki/QR_code)

* * *

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/6/d57ee3de-e717-4da2-a451-34bbcd376d6c.jpeg?raw=true)
 本文由 Seebug Paper 发布，如需转载请注明来源。本文地址：[https://paper.seebug.org/2079/](https://paper.seebug.org/2079/)