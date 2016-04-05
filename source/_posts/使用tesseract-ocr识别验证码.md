---
title: 使用tesseract-ocr识别验证码
date: 2016-02-04 20:40:36
tags: 
---
## tesseract-ocr 简介
[tesseract](https://github.com/tesseract-ocr/tesseract) 是一个开源的OCR引擎。内置提供了多种语言的识别，也支持样本训练。

## tesseract-ocr 安装
tesseract 包括引擎和训练数据两部分。

### tesseract-ocr
工具可以直接从[tesseract](https://github.com/tesseract-ocr/tesseract)官网的源码编译，或者直接用安装包安装。

* Windows. 官网提供了一下第三方的下载[地址](https://github.com/tesseract-ocr/tesseract/wiki/Downloads)，我就是从http://domasofan.spdns.eu/tesseract/tesseract-core-20150916.exe 下载的。
* Linux. 很多Linux的安装组件里面就包含了tesseract-ocr。比如debian或ubuntu可以通过 apt-get install tesseract-ocr 安装。

### tesseract-data
官方提供了现成的[tessdata](https://github.com/tesseract-ocr/tessdata)，包括各种语言。如果在linux系统下面，可以直接通过 apt-get install tesseract-ocr-{lang}安装。lang可以是各种语言，比如eng等。 

## tesseract-ocr 使用
tesseract提供了命令行以及各种语言的API。
### 命令行
```
tesseract imagename outputbase [-l lang] [-psm pagesegmode] [configfiles...]

```
tessdata目录位置可以在系统变量TESSDATA_PREFIX中指定。
### Java API
maven依赖
```xml
		<dependency>
			<groupId>net.sourceforge.tess4j</groupId>
			<artifactId>tess4j</artifactId>
			<version>2.0.0</version>
		</dependency>

```
[tess4j](http://tess4j.sourceforge.net)本身不需要另外安装命令行工具，但是需要准备训练数据。
调用
```java
		net.sourceforge.tess4j.ITesseract instance = new net.sourceforge.tess4j.Tesseract.Tesseract();
		String result = instance.doOCR(image);
```
### Python API

pip安装
```
pip install pytesseract
```
pytesseract其实是通过调用命令行工具来识别验证码的，所以系统中一定要安装命令行工具和训练数据，如果命令行不包含在系统path上(linux上一般会自动添加)，那就需要设置tesseract到pytesseract.pytesseract.tesseract_cmd。
调用
```python
 import pytesseract
 result = pytesseract.image_to_string(image)
```

## 验证码识别实践
简单的验证码如果没有什么干扰，可以直接用tesseract识别。但是实际情况下，很多网站或者系统为了防止自动识别，大部分验证码一般都加入了一些干扰像素。所以需要先把源图片中的干扰像素去掉，然后才能让tesseract识别。

下面我就以上海车牌竞拍系统的验证码为例(去年11月以前的验证码，11月以后系统更新，这个方法已经不适用了):

* 1 截取验证码。 ![source.png](source.png)
* 2 根据提示过滤验证码。因为提示的类型就三种![begin.png](begin.png),![middle.png](middle.png),![end.png](end.png),直接用样本比较就可确定，不需要用OCR识别(中文的识别率很低，所以也没法用OCR)。于是截取前四位，图片就变成了 ![Filter2.png](Filter2.png)
* 3 过滤背景上的干扰色，主要是线条。基本思路就是判断每个一个像素周围与之相似的像素是否超过2个，如果超过2个就代表这个像素是验证码的一部分，否则可能是孤立的像素或者是线条上的像素。对于是否相似可以通过如下逻辑判断
```java
( Math.abs(rgb0.getBlue()-rgb.getBlue())
+ Math.abs(rgb0.getRed()-rgb.getRed())
+ Math.abs(rgb0.getGreen()-rgb.getGreen()) ) < 120;
 //120是个经验值，需要自己不断调试
```
识别结果 ![RemoveLine.png](RemoveLine.png)
* 4 图片二值化。简单说把验证码的前景色变成黑的，其他都变白色。至于如何判断什么是前景色，可以计算图片上颜色最多像素，然后保留与之相似像素(相似的逻辑和上面一样)，其他全部算作背景。最后结果 ![CleanBackground.png](CleanBackground.png)
* 5 调用tess识别就可以得到验证码的字符串了。

上面的方法针对当时的拍牌系统，基本可以做到100%的识别率。

大部分验证码的识别可能不需要这么高的准确率，如果识别失败，可以尝试重试。


## 样本训练
tess虽然提供了很多语言的训练数据，但是在特殊场景，比如特殊字体，也可以自己训练样本。

[官网](https://github.com/tesseract-ocr/tesseract/wiki/TrainingTesseract#introduction)有详细介绍。

因为默认的中文识别率不高，我也曾简单的尝试了一下，可是效果还是不好，所以也没有深入研究。

## 其他OCR工具
当初在研究上海车牌验证码的过程中，除了尝试tessact以外，还试过[Asprise](http://asprise.com/home/). 

java版本可以通过添加下面的maven依赖引人，API也很简单。因为我的验证码都是数字，其实用tessact和Asprise准确度都很高，只是Asprise是个商业版，有一定几率会跳出一个对话框提醒, 不过从配置上比tessact简单，另外tessact初始化有一到两秒的停顿。

```xml
		<!-- asprise OCR -->
		<dependency>
			<groupId>com.asprise.ocr</groupId>
			<artifactId>java-ocr-api</artifactId>
			<version>[15,)</version>
		</dependency>
```