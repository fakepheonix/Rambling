# 适用于Intel GPU在Windows系统上使用ipex-llm运行SakuraLLM的入门教程
内容参考：
[使用 IPEX-LLM 在 Intel GPU 运行 llama.cpp Portable Zip](https://github.com/intel/ipex-llm/blob/main/docs/mddocs/Quickstart/llamacpp_portable_zip_gpu_quickstart.zh-CN.md)
[针对 intel 核显和独显更快的方案：使用IPEX-LLM](https://github.com/SakuraLLM/SakuraLLM/discussions/129)
[Sakura模型本地部署教程](https://books.fishhawk.top/forum/656d60530286f15e3384fcf8)

## 目录
- [系统要求](#系统要求)
- [资源准备](#资源准备)
- [运行步骤](#运行步骤)

## 系统要求
根据Intel官方的建议，GPU驱动版本不应低于`31.0.101.5522`，否则可能会输出乱码。另外官方文档有提到的最低驱动版本为`31.0.101.5122`。

如果需要更新驱动，可以在[Intel官方驱动下载页面](https://www.intel.com/content/www/us/en/download/785597/intel-arc-iris-xe-graphics-windows.html)下载更新。

## 资源准备
从[ipex-llm仓库的发布页](https://github.com/ipex-llm/ipex-llm/releases/tag/v2.2.0)下载适用于Windows的免安装ipex-llm llama.cpp的压缩包`llama-cpp-ipex-llm-2.2.0-win.zip`并解压。本教程中，假设将上述压缩包解压为`D:\ipex-llm`这一文件夹。

根据性能需要和实际硬件资源，从[SakuraLLM的仓库](https://huggingface.co/SakuraLLM)下载合适的SakuraLLM模型（即不同的`.gguf`文件）。

> 尽管SakuraLLM官方建议`sakura-7b-qwen2.5-v1.0-iq4xs.gguf`模型需要8G显存，但在本人的Intel Core Ultra 5 125H和Nvidia RTX 3050 8GB上实测，该模型大约只需要5G显存，而`sakura-14b-qwen2.5-v1.0-iq4xs.gguf`则刚好占用8G显存

在模型文件相同文件夹下，新建一个文本文档，将下列内容复制进去：

```bat
@echo off
@chcp 65001 > nul

set label=显卡
:: 下一行的ngl参数指的是在GPU上运行的神经网络层数。如果ngl值低于模型层数，意味着将部分层卸载到CPU上计算，可以降低显存占用，但速度会明显比全部用GPU计算的情况慢
set ngl=200
setlocal enableextensions enabledelayedexpansion
set SYCL_CACHE_PERSISTENT=1

:: 下一行的设置通常会提高性能，但也可能例外，请自行测试；如果需要关闭该设置，请在下一行行首输入两个连续英文冒号，即“::”
set SYCL_PI_LEVEL_ZERO_USE_IMMEDIATE_COMMANDLISTS=1

if /i "%label%"=="" (
	echo 脚本异常
	goto quit
)

@title 启动Sakura服务器-%label%

set n=0
for /f "delims=" %%i in ('where .:*.gguf') do (
	set models[!n!].path=%%i
	set models[!n!].name=%%~ni
	set /a n+=1
)

if %n% equ 0 goto no-model
if %n% equ 1 goto one-model
goto many-model

:no-model
echo 没有检测到gguf模型文件,请确定将模型文件放到了当前文件夹
goto quit

:one-model
set model.name=%models[0].name%
set model.path=%models[0].path%
goto launch

:many-model
set /a end=%n%-1

echo 请输入数字来选择要使用的模型,默认选择0
for /l %%i in (0,1,%end%) do (
	echo %%i. !models[%%i].name!
)
echo.
:choice-model
set choice=
set /p choice= 请选择:
if /i "%choice%"=="" set choice=0
for /l %%i in (0,1,%end%) do (
	if /i "%choice%"=="%%i" (
		set model.name=!models[%%i].name!
		set model.path=!models[%%i].path!
		goto launch
	)
)
echo 选择⽆效,请重新输⼊
goto choice-model

:launch
@title 启动Sakura服务器-%label%-%model.name%
echo.
echo 模型名称：%model.name%
echo 模型路径：%model.path%
echo.
echo 准备启动Sakura服务器...
@echo on

:: 将下一行的“D:\ipex-llm\llama-server.exe”替换为之前被解压的llama-cpp-ipex-llm-2.2.0-win.zip内的llama-server.exe路径
D:\ipex-llm\llama-server.exe -m .\%model.name%.gguf -c 2048 -ngl %ngl% -a %model.name% --host 127.0.0.1 

@echo off

goto quit

:quit
echo.
echo 按任意键退出
pause > nul
exit
```

请根据需要，自行修改上文开头的**ngl参数**、性能设置以及接近结尾处的**文件路径**。完成后将上述文本文件保存到**和模型相同的文件夹**内，修改文件后缀名为`.bat`

## 运行步骤
1. 打开[轻小说机翻机器人-Sakura工作区](https://books.fishhawk.top/workspace/sakura)；
2. 点击`任务队列-本地书架`，在侧栏右上角点击`添加`按钮，将需要翻译的小说添加给浏览器；
3. 双击或右键运行上面保存的`.bat`文件，等待模型加载，直到弹出的窗口最后一行出现“idle”字样，不要关闭窗口；
4. （可选）回到Sakura工作区的主界面，点击“本机”一行右侧的闪电形状的“测试”按钮进行测试；
5. 点击“本机”一行右侧的“启动”按钮，等待翻译完成；
6. 在“本地书架”栏目可以阅读或下载已完成翻译的小说。如果需要关闭Sakura模型，在“本机”一行右侧点击停止、关闭后台的黑色命令行窗口即可。
