# EasyRayTracing Project

该项目为一个简单的光追项目，整个流程参考了教程[Ray tracing in one weekend](https://raytracing.github.io/books/RayTracingInOneWeekend.html)，同时记录一下完成项目的过程中一些自己的想法，如果哪里想错了欢迎指正，也希望能够真的如标题所说一个周末完成，搞完这个项目就接着回去看opengl，换换口味。整个教程大概12节的内容，那就每完成一节就总结一下好了。

因为这个教程用最基础的C++功能完成了整个渲染流程，也没有借助别的图形API，所以整个工作环境也就用了Visual studio 2019，建个C++项目就可以开搞了，那就走起。

## · 创建图片文件
为了简化渲染流程，我们最终渲染得到的结果为一张PPM格式的图片，PPM的内容如下所示：
![PPM例子](1.jpg)
如注释所说，第一行P3代表颜色都以ASCII码的形式编写，下面的3 2代表我这张图3行2列的像素，255代表颜色的最大值，然后就是六个RGB值。PPM可以用各类图片浏览软件打开，比如photoshop，如果没有又懒得装，教程中提供了一个在线加载PPM图片的网站[PPM Viewer](https://www.cs.rhodes.edu/welshc/COMP141_F16/ppmReader.html)。

那我们就先来创建一张图片试试，代码如下，我们创建了一张256*256大小的图片，循环里写入rgb值，red值从左到右越来越大，green值从上到下越来越小，中间红色和绿色混在一起就是黄色。因为我们用了cout输出流来输出PPM文件的内容，所以如果我们想让程序输出一些进度信息就需要用其他的输出流，所以这里用了cerr，flush用于刷新流缓冲区。
```
#include <iostream>

int main() {
	const int image_width = 256;
	const int image_height = 256;
	std::cout << "P3\n" << image_width << ' ' << image_height << "\n255\n";

	for (int j = image_height - 1; j >= 0; --j) {
		std::cerr << "\rScanlines remainin:" << j << ' ' << std::flush;
		for (int i = 0; i < image_width; ++i) {
			auto r = double(i) / (image_width - 1);
			auto g = double(j) / (image_height - 1);
			auto b = 0.25;

			int ir = static_cast<int>(255.999 * r);
			int ig = static_cast<int>(255.999 * g);
			int ib = static_cast<int>(255.999 * b);

			std::cout << ir << ' ' << ig << ' ' << ib << '\n';
		}
	}
	
	std::cerr << "\nDone.\n";
```
编译出exe文件后，在cmd里输入```EasyRaytracing > image.ppm```, 这样程序就能将PPM内容输入到image.ppm文件中，就得到了这样一张炫酷的图片。
![好帅的图](2.jpg)

## · Vec3类
很多图形程序都会为向量、颜色设计class，而且基本上都是四维，对于向量坐标来说，出了xyz三个维度以外，还有一个齐次分量用于投影变换，对于颜色来说第四维就是alpha透明度通道，但是本项目还是从简的目的，将向量和颜色都设计为一个Vec3的类型，头文件vec3.h就是用于该类型的设计，这个做法其实还是不太好的，有时候不小心把向量和颜色加起来，程序本身不会有报错提示，因为都是同样的类型，还是那句话，本项目从简。。为了让向量和颜色有稍许的区别，我们还是在C++中给color和point3设置了各自的别名，但本质上还是vec3。
在vec3.h中，我们设置了各种运算符重载，还用到了内敛函数，因为向量的计算在渲染过程中是频繁用到的，如果每次程序都去调用函数到内存中，效率也太低了，所以用内联函数直接在函数被调用的时候进行展开，用空间换时间。有了头文件之后，我们把渲染程序精简一下，编译完再生成一下ppm图片，没啥问题就继续下一节。
```
#include "color.h"
#include "vec3.h"

#include <iostream>

int main() {
	const int image_width = 256;
	const int image_height = 256;
	std::cout << "P3\n" << image_width << ' ' << image_height << "\n255\n";

	for (int j = image_height - 1; j >= 0; --j) {
		std::cerr << "\rScanlines remainin:" << j << ' ' << std::flush;
		for (int i = 0; i < image_width; ++i) {
			color pixel_color(double(i) / (image_width - 1), double(j) / (image_height - 1), 0.25);
			write_color(std::cout, pixel_color);
		}
	}
	
	std::cerr << "\nDone.\n";
}
```

## · 光线，简单的摄像机以及背景设置
睡觉，明天再继续