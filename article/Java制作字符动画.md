### 前言

今晚闲来无事，整理了一下电脑中尘封已久的旧代码，看着那些年自己写过的代码，踩过的坑，顿时老泪纵横。正当在感叹之际，突然发现在“马克思”文件夹下出现了一个好玩的项目，那就是N年前刚学Java时写的GIF转字符动画的小玩具，虽然是个小玩意，但是在当时能搞点东西出来还是非常有成就感的。

### 正文

#### 效果展示

原图，某两年半练习生

![](https://user-gold-cdn.xitu.io/2019/5/7/16a92c8bda6e06f7?w=506&h=471&f=gif&s=2831164)

转成字符动画后的练习生


![](https://user-gold-cdn.xitu.io/2019/5/7/16a92c939fd7c9f1?w=504&h=469&f=gif&s=1757195)

#### 实现原理

其实字符动画的实现原理比较简单，这里我们抛开GIF，直接拿一张静态图片来说明。  
首先我们要把原图转成灰度图，这样图片中每个像素就只存在亮度信息0-255。

![](https://user-gold-cdn.xitu.io/2019/5/7/16a92d6e9d54b13b?w=402&h=300&f=png&s=140792)

取颜色的RGB均值灰度后


![](https://user-gold-cdn.xitu.io/2019/5/7/16a92d76f064c7dd?w=402&h=300&f=png&s=65723)

接着我们可以定义需要使用的字符，每个字符对应一段亮度范围，比如 图中的`M`,`@`,`;`等字符，接着我们就可以去遍历替换图片中的所有像素，慢慢的调试每个字符对应像素的亮度范围，调试到输出的图像轮毂清晰即可，这样单张图片的字符画就已经成型了。下面关键代码注释。

```java
BufferedWriter bufferedWriter = new BufferedWriter(new OutputStreamWriter(new FileOutputStream(file)));
		int width = bi.getWidth();//原图宽度
		int height = bi.getHeight();//原图高度
		int minx = bi.getMinX();//BufferedImage 原图 最小X坐标
		int miny = bi.getMinY(); //BufferedImage 原图 最小Y坐标
		for (int i = miny; i < height; i += 8) {//遍历图片中的像素点，用字符判断像素范围来替换
			for (int j = minx; j < width; j += 8) {
				int pixel = bi.getRGB(j, i); // 下面三行代码将一个数字转换为RGB数字
				int red = (pixel & 0xff0000) >> 16;
				int green = (pixel & 0xff00) >> 8;
				int blue = (pixel & 0xff);
				double gray = 0.299 * red + 0.578 * green + 0.114 * blue; //图片变灰计算公式
				char c = toChar((int) gray); //根据计算出来的gray值返回不同字符
				bufferedWriter.write(c);
			}
			bufferedWriter.newLine();
		}
		//输出图片
```

若要读取GIF，输出GIF，我们可以使用一些开源的包，例如animated-gif，GifDecoder等，通过这些类我们可以读取到gif的每一帧，然后我们对每一帧的操作都跟上方的静态图操作是一致的。处理完每一帧之后再合成GIF输出即可。（视频同理）

由于完全自己处理的话，可能会有很多细节需要调整的地方，为了方便，这里推荐一个项目。Github地址：https://github.com/korhner/asciimg 。使用方法：

```java
// initialize caches
AsciiImgCache smallFontCache = AsciiImgCache.create(new Font("Courier",Font.BOLD, 6));
// initialize ssimStrategy
BestCharacterFitStrategy ssimStrategy = new StructuralSimilarityFitStrategy();

String srcFilePath = "examples/xxx.gif";
String disFilePath = "examples/xxx.gif";
int delay = 100;//ms

GifToAsciiConvert asciiConvert = new GifToAsciiConvert(smallFontCache, ssimStrategy);

asciiConvert.convertGitToAscii(srcFilePath, disFilePath, delay,0);
```

只需要简单的几行，就可以完成字符动画的转换，其原理跟我们上面介绍的基本一致，有兴趣的同学可以自行研究。

### 结语

代码除了用来工作，其实还能用在很多能让我们开心的地方，例如写点小工具，小游戏，帮自己或他人解决一些繁琐的事情，这样才能在工作多年后任然保持对代码的那份初心，不至于被重复的工作磨灭了激情。

#### 推荐阅读

《[如何优化代码中大量的if/else,switch/case?](https://mp.weixin.qq.com/s/ImswCn_eL4jBleET7fJVdw)》  
《[如何提高使用Java反射的效率？](https://mp.weixin.qq.com/s/-HXqicBROZU8XDF5YSCHMw)》  
《[Java日志正确使用姿势](https://mp.weixin.qq.com/s/aQx2ROajH2SqgHL77yxW3Q)》   
《[Java异常处理最佳实践及陷阱防范](https://mp.weixin.qq.com/s/zeGqY0ZcrU_oOHpVW9V3zQ)》    
《[论JVM爆炸的几种姿势及自救方法](https://mp.weixin.qq.com/s/2oLX-i5zbTNayjJzAOSN8A)》    


<p align="center">
有收获的话，就分享给更多的朋友吧<br/>
<b>关注「深夜里的程序猿」，分享最干的干货</b>
</p>
<p align="center">
<img src="/resource/qrcode.png" alt="Sample"  width="200" height="200">