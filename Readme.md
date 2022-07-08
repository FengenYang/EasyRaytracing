# EasyRayTracing Project

该项目为一个简单的光追项目，整个流程参考了教程[Ray tracing in one weekend](https://raytracing.github.io/books/RayTracingInOneWeekend.html)，同时记录一下完成项目的过程中一些自己的想法，如果哪里想错了欢迎指正，也希望能够真的如标题所说一个周末完成，搞完这个项目就接着回去看opengl，换换口味。整个教程大概12节的内容，那就每完成一节就总结一下好了。

因为这个教程用最基础的C++功能完成了整个渲染流程，也没有借助别的图形API，所以整个工作环境也就用了Visual studio 2019，建个C++项目就可以开搞了，那就走起。

## · 创建图片文件
为了简化渲染流程，我们最终渲染得到的结果为一张PPM格式的图片，PPM的内容如下所示：<br />
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
编译出exe文件后，在cmd里输入```EasyRaytracing > image.ppm```, 这样程序就能将PPM内容输入到image.ppm文件中，就得到了这样一张炫酷的图片。<br />
![好帅的图](2.jpg)

## · Vec3类
很多图形程序都会为向量、颜色设计class，而且基本上都是四维，对于向量坐标来说，出了xyz三个维度以外，还有一个齐次分量用于投影变换，对于颜色来说第四维就是alpha透明度通道，但是本项目还是从简的目的，将向量和颜色都设计为一个Vec3的类型，头文件vec3.h就是用于该类型的设计，这个做法其实还是不太好的，有时候不小心把向量和颜色加起来，程序本身不会有报错提示，因为都是同样的类型，还是那句话，本项目从简。。为了让向量和颜色有稍许的区别，我们还是在C++中给color和point3设置了各自的别名，但本质上还是vec3。<br />
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
光追最关键的就是光线的定义，我们将光线视为一个向量，用函数表示为 P(t)=A+tb，其中A为光线的起点，b为光线的方向，t为我们自己定义的一个数值，其代表这个向量的长度，更重要的是t的正负值可以让我们的光线正向反向进行传播。头文件ray.h定义了这样一个光线的类型。<br />
然后我们将光线射向场景中，然后把光线击中的所有像素的颜色加起来就能得到最终颜色，光线追踪的过程主要分为三步：首先，计算从视角出发的光线到像素的整个路径，本项目中摄像机的位置就是视角的位置；然后计算出路径中光线接触到了哪些物体；最后将光线和各个物体接触点的颜色相加。那我们定义了光线之后，就来把main函数改写一下：
```
color ray_color(const ray& r) {
	vec3 unit_direction = unit_vector(r.direction());
	auto t = 0.5 * (unit_direction.y() + 1.0);
	return (1.0 - t) * color(1.0, 1.0, 1.0) + t * color(0.5, 0.7, 1.0);
}

int main() {
	//image
	const auto aspect_ratio = 16.0 / 9.0;
	const int image_width = 400;
	const int image_height = static_cast<int>(image_width/aspect_ratio);
	
	//camera
	auto viewport_height = 2.0;
	auto viewport_width = aspect_ratio * viewport_height;
	auto focal_length = 1.0;

	auto origin = point3(0, 0, 0);
	auto horizontal = vec3(viewport_width, 0, 0);
	auto vertical = vec3(0, viewport_height, 0);
	auto lower_left_corner = origin - horizontal / 2 - vertical / 2 - vec3(0, 0, focal_length);

	//render
	std::cout << "P3\n" << image_width << ' ' << image_height << "\n255\n";

	for (int j = image_height - 1; j >= 0; --j) {
		std::cerr << "\rScanlines remainin:" << j << ' ' << std::flush;
		for (int i = 0; i < image_width; ++i) {
			auto u = double(i) / (image_width - 1);
			auto v = double(j) / (image_height - 1);
			ray r(origin, lower_left_corner + u * horizontal + v * vertical - origin);
			color pixel_color = ray_color(r);
			write_color(std::cout, pixel_color);
		}
	}
	
	std::cerr << "\nDone.\n";
}
```
在main函数之前，我们定义了一个write_color函数，参数为我们当前视角到某个像素的一条光线，首先我们取光线在y轴方向上的标准化的值，其范围为(-1,1)，那t的范围就是(0,1)，这个t就相当于一个权重，用来计算我当前像素的颜色值中白色和蓝色的比重，可以看到t越大蓝色值占比越高，在渲染出来的图片中可以看到就是一个渐变的蓝色图片。<br />
那在main函数中，我们将图片长宽比设为16比9，然后循环遍历每个像素计算颜色值，u和v类似于NDC坐标，它的值被限制到0到1的范围，然后我们用图片左下角的位置计算便宜，得到像素在图片中的位置，也就是r变量构造函数的参数所作的事情。图片左下角的位置我们用变量lower_left_corner来表示，可以看到origin代表我摄像机的位置，它的xy也就是图片的中心，然后在z轴方向有一个focal_length的距离。编译运行，得到最终渲染结果。<br />

![blue](3.png)

## · 加个球~
现在我们在场景中添加一个球来直观地表现光线击中物体得到不同颜色的情况。首先来复习一下高中数学，我们假设球心的坐标为(C<sub>x</sub>,C<sub>y</sub>,C<sub>z</sub>)，半径为r，那就能得到这个球的函数：<br />
$$(x-C_x)^2+(y-C_y)^2+(z-C_z)^2=r^2$$
现在我们有了光线的函数，假设光线与这个球相交或者相切，也就是光线P上至少有一个点到球心的距离等于半径，即：
$$(P-C)^2=r^2$$
我们有光线P关于t的函数式，带入并展开可以得到一个关于t的二元一次方程：
$$t^2b·b+2tb·(A-C)+(A-C)^2-R^2=0$$
这个式子看似复杂，其实只有一个未知量t，所以我们用b<sup>2</sup>-4ac的正负或零来判断这个方程是否有解，式子大于0有两个解也就是光线与球相交，等于0一个解代表光线与球面相切，小于0那就是光线与球没有交点。知道了原理，那就用代码实现一下。
```
bool hit_sphere(const point3& center, double radius, const ray& r) {
	vec3 oc = r.origin() - center;
	auto a = dot(r.direction(), r.direction());
	auto b = 2.0 * dot(oc, r.direction());
	auto c = dot(oc, oc) - radius * radius;
	auto discriminant = b * b - 4 * a * c;
	return (discriminant > 0);
}
```
我们用这个函数判断光线是否与球面相交，值得注意的是在二元一次方程中，A为光线的起点，也就是当前场景的原点，C为球心，所以A-C用代码表示为原点到球心的向量，所以是变量oc。r.direction()是二次方程中的b，因为代表了光线的方向。随后我们修改一下确定光线颜色的函数：
```
color ray_color(const ray& r) {
	if (hit_sphere(point3(0, 0, -1), 0.5, r))
		return color(1, 0, 0);
	vec3 unit_direction = unit_vector(r.direction());
	auto t = 0.5 * (unit_direction.y() + 1.0);
	return (1.0 - t) * color(1.0, 1.0, 1.0) + t * color(0.5, 0.7, 1.0);
}
```
在最前面加了一个if判断，也就是在光线与球心在(0,0,-1)半径为-1的球有交点时，该像素点颜色为红色，否则还是之前的渐变蓝色。有一个问题是，还记得之前t的正负值代表光线的正向和反向吗，目前t是有可能为负值的，也就是说哪怕这个球的位置在光线起点的后面，也是会得到交点，我们会在后续修复这个问题。程序没有问题的情况下得到最终效果：<br />
![red ball](4.png)

## · 表明的法向和多个物体
