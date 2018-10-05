####本篇笔记主要记录Eos根路径下每个目录的功能。
####1、从github上克隆代码，并编译。
```
git clone https://github.com/EOSIO/eos --recursive
git submodule update --init --recursive

cd eos
./eosio_build.sh
./eosio_install.sh
```
编译过程中出现的任何错误，可以留言。
####2、编译完成之后的代码目录如下图：
![eos目录.png](https://upload-images.jianshu.io/upload_images/14187740-43be13bb81f01b5e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
####3、eos每个目录的功能：
build目录：代码编译之后生成的文件。包含各种插件生成的静态库文件、可执行文件、测试脚本等。
cmake-build-debug目录：我自己使用clion编译器生成的目录，非eos目录。
CMakeModules目录：eos项目依赖的第三方模块，编译用到的cmake脚本（搜索lib文件和头文件等功能）。
contracts目录：智能合约示例文件，可以最后再看。
debian目录：包含了eos再debian下安装的一些配置文件。
Docker目录：Docker安装用到的配置脚本和Dockerfile。
docs目录：包含一个css格式文件，可能是渲染教程页面。
externals目录：外部第三方依赖。binaryen（wasm项目的二进制工具包）、magic_get（C++14 的库，通过索引访问不同类型的元素）。
images目录：图片、logo等。
libraries目录：和业务无关的一些类、工具函数等（比如插件管理类、wasm虚拟机、ECC加密算法等），注意其中有些库本来是boost库里面的，但是eos重新写了。
plugins目录：插件目录（钱包插件、链插件、p2p插件、http插件等），需要重点分析。
programs目录：可执行文件，包括了cleos、nodeos、keosd等可执行文件，也需要重点分析，分析的入口。
scripts目录：脚本目录，不同平台下的安装命令脚本等。
tests目录：测试py脚本，每个功能都有其测试脚本。
tools目录：辅助工具，可执行文件。gcov（计算代码覆盖率，执行路径），eosiocpp生成智能合约的abi和wast文件。
tutorials目录：部分教程（bios boot启动，普通功能使用教程）。
unittests目录：单元测试。

####4、重点分析的目录：
plugins目录和programs目录。
####5、三个可执行文件
nodeos文件：出块、验证交易、广播交易、提供http接口等主要功能，重点分析。
cleos文件：与nodeos交互的命令行工具，后续分析。
keos文件：钱包文件，后续分析。
