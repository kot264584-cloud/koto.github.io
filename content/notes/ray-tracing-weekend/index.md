---
title: "Ray Tracing in One Weekend 学习记录"
date: 2026-10-17
draft: false
description: "Ray Tracing in One Weekend 学习记录"
tags: ["光线追踪", "图形学"]
math: true
ShowToc: true
---

### 附：关于PPM格式；
> 1.如何告诉机器文件类型为ppm；   
-> std::cout<<"P3\n"<< image_width<<' '<< image_length<<"\n255\n";


## 一.生成一张简单的渐变色图片
代码：
```cpp
#include<iostream>

int main()
{
    //image
    int image_width=256;
    int image_length=256;

    //render

    std::cout<<"p3\n"<<image_width<<' '<<image_length<<"\n255\n";

    for(int j=0;j<image_length;++j){
        for(int i=0;i<image_width;++i){
            auto r=double(i)/image_width;
            auto g=double(j)/image_length;
            auto b=0.0;

            int ir=static_cast<int>(255.99*r);
            int ig=static_cast<int>(255.99*g);
            int ib=static_cast<int>(255.99*b);

            std::cout<<ir<<' '<<ig<<' '<<ib<<"\n";
        }
    }
}
```
### 附:有关三种输出方式
> cout:可被读入文件之中的输出方式;    
clog:不会被读入文件中的输出方式,可用于可视化渲染进度;   
cerr:不会被读入文件中的输出方式(具体用法暂不清楚,大致与报错相关)
## 二.有关向量(点坐标)的类创建 vec3.h
代码:
```cpp
#pragma once

#include<cmath>
#include<iostream>

class vec3{
    public:
        double e[3];

        vec3():e{0,0,0} {}
        vec3(double e0,double e1,double e2):e{e0,e1,e2} {}

        double x()const{return e[0];}
        double y()const{return e[1];}
        double z()const{return e[2];}

        vec3 operator-() const{ return vec3(-e[0],-e[1],-e[2]);}
        double operator[](int i) const{ return e[i];}
        double& operator[](int i){ return e[i];}

        vec3& operator+=(const vec3& v){
            e[0]+=v.e[0];
            e[1]+=v.e[1];
            e[2]+=v.e[2];
            return *this;
        }

        vec3& operator*=(const double t){
            e[0]*=t;
            e[1]*=t;
            e[2]*=t;
            return *this;
        }

        vec3& operator/=(double t){
            return *this *=1/t;
        }

        double length_squared() const{
            return e[0]*e[0] + e[1]*e[1] + e[2]*e[2];
        }

        double length() const{
            return std::sqrt(length_squared());
        }
    };
    using point3=vec3;

    inline std::ostream& operator<<(std::ostream& out,const vec3& v){
        return out<<v.e[0]<<" "<<v.e[1]<<" "<<v.e[2];
    }

    inline vec3 operator+(const vec3& u,const vec3& v){
        return vec3(u.e[0]+v.e[0],u.e[1]+v.e[1],u.e[2]+v.e[2]);
    }

    inline vec3 operator-(const vec3& u,const vec3& v){
        return vec3(u.e[0]-v.e[0],u.e[1]-v.e[1],u.e[2]-v.e[2]);
    }

    inline vec3 operator*(const vec3& u,const vec3& v){
        return vec3(u.e[0]*v.e[0],u.e[1]*v.e[1],u.e[2]*v.e[2]);
    }

    inline vec3 operator*(double t,const vec3& v){
        return vec3(t*v.e[0],t*v.e[1],t*v.e[2]);
    }

    inline vec3 operator*(const vec3& v,double t){
        return t*v;
    }

    inline vec3 operator/(const vec3& v,double t){
        return (1/t)*v;
    }

    inline double dot(const vec3& u,const vec3& v){
        return  u.e[0]*v.e[0]
                +u.e[1]*v.e[1]
                +u.e[2]*v.e[2];
    }

    inline vec3 cross(const vec3& u,const vec3& v){
        return vec3(u.e[1]*v.e[2]-u.e[2]*v.e[1],
                    u.e[2]*v.e[0]-u.e[0]*v.e[2],
                    u.e[0]*v.e[1]-u.e[1]*v.e[0]);
    }

    inline vec3 unit_vector(vec3 v){
        return v/v.length();
    }
```
其中相关:   
1. inline:告诉机器调用函数时建议直接插入,而不是读取函数内信息(在使用量大的时候能一定程度优化使用速度)   
#pragma once:等价于   
>#ifndef XXX_H   
>#define XXX_H   
>......   
>#endif   
>***用处:防止头文件之间读取报错***   
>***注:#pragma once更现代化,常用***
2. 该头文件用于定义新的数据类型:向量,三维点坐标;   
同时,对该数据类型的引用及相关计算做了新定义,例如:   
点乘,叉乘,取模,求单位向量......    

## 三.有关光线的类创建
代码
```cpp
#pragma once;

#include"vec3.h"

class ray{
    public:
        ray() {}

        ray(const point3& origin,const vec3& direction):orig(origin),dir(direction){}

        const point3& origin()const{return orig};
        const vec3& direction()const{return dir};

        point3 at(double t) const{
            return orig+t*dir;
        }

    private:
        point3 orig;
        vec3 dir;
};
```
#### 相关解析:   
1. public和private:   
public是全局变量,所有文件都可以调用其中的定义类型;   
而private中定义的东西,只有改头文件内部可进行调取;
2. origin:光线的起点;   
direction:光线的方向;   
oirg+t*dir: 光线上的某一点;   
***如何调用该点坐标:ray.at(t);***

## 四.用以上新定义创建一个模拟天空蓝渐变色背景
代码
```cpp
#include"color.h"
#include"vec3.h"
#include"ray.h"

#include<iostream>

// color与point3一样,是vec3类型,只是用不同名字来区分不同用途

color ray_color(const ray& r){
    vec3 unit_direction=unit_vector(r.direction());
    auto t=0.5*(unit_direction.y()+1.0);
    return (1.0-t)*color(1.0,1.0,1.0)+t*color(0.5,0.7,1.0);
}

int main(){
    //image

    auto aspect_ratio=16.0/9.0;
    int image_width=400;

    //长宽转化,并确保长大于1
    int image_height=static_cast<int>(image_width/aspect_ratio);
    image_height=(image_height<1?1:image_height);

    //Camera

    auto focal_length=1.0;
    auto viewport_height=2.0;
    auto viewport_width=viewport_height*(static_cast<double>(image_width)/image_height);
    auto camera_center=point3(0,0,0);

    //创建窗口边界水平和垂直方向的向量
    auto viewport_u=point3(viewport_width,0,0);
    auto viewport_v=point3(0,-viewport_height,0);

    //计算向量u,v的增量
    auto pixel_delta_u=viewport_u/image_width;
    auto pixel_delta_v=viewport_v/image_height;

    //计算左上角像素坐标
    auto viewport_upper_left=camera_center-vec3(0,0,focal_length)-viewport_u/2-viewport_v/2;
    auto pixel00_loc=viewport_upper_left+0.5*(pixel_delta_v+pixel_delta_u);

    //rander

    std::cout<<"P3\n"<<image_width<<" "<<image_height<<"\n255\n";

    for(int j=0;j<image_height;++j){
        std::clog<<"\rScrean remaining:"<<(image_height-j)<<' '<<std::flush;
        for(int i=0;i<image_width;++i){
            auto pixle_center=pixel00_loc+(i*pixel_delta_u)+(j*pixel_delta_v);//读取像素坐标
            auto ray_direction=pixle_center-camera_center;

            ray r(camera_center,ray_direction);

            color pixel_color=ray_color(r);
            write_color(std::cout,pixel_color);
        }
    }
    std::clog<<"\rDone\n";
}
```
#### 实现思想细节:
1. 定义了三种视野类型:摄像中心,视窗,像素步进;   
>camera_center定位在(0,0,0);   
>视窗被分割成了400*225的小方格;   
>viewport_u为视窗的水平向量,viewport_v为垂直向量,大小与定义的视窗方格数相等;   
>定义pixel_delta_u与pixel_delta_v为水平与数值方向上,遍历像素时的步长,即像素步进;   
2. 用双重for循环配合定义的像素步进,从左到右,从上到下遍历视窗像素;   
同时,以摄像中心的光线起点,以起点到遍历的各像素的方向为光线方向,建立ray;   
最后,通过ray_color函数,将每道光线的y坐标从[-1,1]映射到[0,1],再将映射值分配给输出颜色中白色与天蓝色的占比中;   
最终实现下白上蓝的渐变图像输出

***这段代码的要点:***
1. 先知道什么是光追:光追就是在光线穿过每个像素时,计算出每个像素应该输出的对应颜色;
2. 因此,我们大致分三步走:   
1.计算视点到像素之间的光线;   
2.求出与光线相交的物品(具体实现在下一章节);   
3.计算交点处的颜色(即ray_color所实现的功能);
3. 本次实践用以了解步骤1与3;

## 五.在画面中添加一个交互球
代码
```cpp
//判断与球体相交的光线
bool hit_sphere(const point3& center,double radius,const ray& r){
    vec3 oc=center-r.oigin();
    auto a=dot(r.direction(),r.direction());
    auto b=dot(r.direction(),oc);
    auto c=dot(oc,oc)-radius*radius;
    auto delta=b*b-4*a*c;
    return (delta>=0);
}

color ray_color(const ray& r){
    if(hit_sphere(point3(0,0,-1),0.5,r)){
        return color(1,0,0);
    }
    vec3 unit_direction=unit_vector(r.direction());
    auto t=0.5*(unit_direction.y()+1.0);
    return (1.0-t)*color(1.0,1.0,1.0)+t*color(0.5,0.7,1.0);
}
```
#### ***修改内容:添加了光线是否与球体相交的函数hit_sphere;在染色函数里对于通过球体的光线做输出红色处理;***

## 六.法向和多物体交互
### 1.法向可视化实现
```cpp
double hit_sphere(const point3& center,double radius,const ray& r){
    vec3 oc=center-r.oigin();
    auto a=r.direction().length_squared();
    auto h=dot(r.direction(),oc);
    auto c=oc.length_squared()-radius*radius;
    auto delta=h*h-a*c;
    if(delta<0){
        return -1.0;
    }else{
        return (h-std::sqrt(delta))/a;
    }
}

color ray_color(const ray& r){
    auto a=hit_sphere(point3(0,0,-1),0.5,r);
    if(a>0.0){
        vec3 N=unit_vector(r.at(a)-vec3(0,0,-1));
        return 0.5*color(N.x()+1, N.y()+1, N.z()+1);
    }
    
    vec3 unit_direction=unit_vector(r.direction());
    auto t=0.5*(unit_direction.y()+1.0);
    return (1.0-t)*color(1.0,1.0,1.0)+t*color(0.5,0.7,1.0);
}
```

### 2.创建与可视物体有关的类
```cpp
#pragma once

#include"ray.h"

class hit_record{
    public:
        point3 p;
        vec3 normal;
        double t;
        
};

class hittable{
    public:
        virtual ~hittable()=default;

        virtual bool hit(const ray& r,double ray_tmin,double ray_tmax,hit_record& rec) const=0;
};
```
### 3.创建相关操作函数(判断是否可视,并录入相关物体信息,包括:光线与物体交点,该点法向,与光线t值)
```cpp
#pragma once

#include "hittable.h"
#include "vec3.h"

class sphere : public hittable {
  public:
    sphere(const point3& center, double radius) : center(center), radius(std::fmax(0,radius)) {}

    bool hit(const ray& r, double ray_tmin, double ray_tmax, hit_record& rec) const override {
        vec3 oc = center - r.origin();
        auto a = r.direction().length_squared();
        auto h = dot(r.direction(), oc);
        auto c = oc.length_squared() - radius*radius;

        auto delta = h*h - a*c;
        if (delta < 0)
            return false;

        auto sqrtd = std::sqrt(delta);

        // Find the nearest root that lies in the acceptable range.
        auto root = (h - sqrtd) / a;
        if (root <= ray_tmin || ray_tmax <= root) {
            root = (h + sqrtd) / a;
            if (root <= ray_tmin || ray_tmax <= root)
                return false;
        }

        rec.t = root;
        rec.p = r.at(rec.t);
        rec.normal = (rec.p - center) / radius;

        return true;
    }

  private:
    point3 center;
    double radius;
};
```

### 4.加入判断法向与光线方向(同向则说明光线来源于物体内部,反向则相反)
代码
```cpp
class hit_record{
    public:
        point3 p;
        vec3 normal;
        double t;
        bool front_face;

        void set_face_normal(const ray& r,const vec3& outward_normal){
            front_face=dot(r.direction(),outward_normal)<0;
            normal=front_face?outward_normal:-outward_normal;
        }
        
};
--------------------------------------------------------------------------------------
bool hit(const ray& r, double ray_tmin, double ray_tmax, hit_record& rec) const override {
        vec3 oc = center - r.origin();
        auto a = r.direction().length_squared();
        auto h = dot(r.direction(), oc);
        auto c = oc.length_squared() - radius*radius;

        auto delta = h*h - a*c;
        if (delta < 0)
            return false;

        auto sqrtd = std::sqrt(delta);

        // Find the nearest root that lies in the acceptable range.
        auto root = (h - sqrtd) / a;
        if (root <= ray_tmin || ray_tmax <= root) {
            root = (h + sqrtd) / a;
            if (root <= ray_tmin || ray_tmax <= root)
                return false;
        }

        rec.t = root;
        rec.p = r.at(rec.t);
        vec3 outward_normal=(rec.p-center)/radius;
        rec.set_face_normal(r,outward_normal);

        return true;
    }
```

### 5.将可视物体统合成一类(创建一个场景管理器)
```cpp
#pragma once

#include"hittable.h"

#include<memory>
#include<vector>

using std::make_shared;
using std::shared_ptr;

class hittable_list:public hittable{
    public:
        std::vector<shared_ptr<hittable>> objects;

        hittable_list() {}
        hittable_list(shared_ptr<hittable> object) {add(object);}

        void clear(){objects.clear();}

        void add(shared_ptr<hittable> object){
            objects.push_back(object);
        }

        bool hit(const ray& r,double ray_tmin,double ray_tmax,hit_record& rec) const override{
            hit_record temp_rec;
            bool hit_anyting=false;
            auto closest_so_far=ray_max;

            for(const auto& object:objects){
                if(object->hit(r,ray_tmin,closest_so_far,temp_rec)){
                    hit_anyting=true;
                    closest_so_far=temp_rec.t;
                    rec=temp_rec;
                }
            }
            return hit_anyting;
        }
};
```

### 6.创建一个用于存放常用数学常亮(及相关函数)的头文件
```cpp
#pragma once

#include<cmath>
#include<iostream>
#include<limits>
#include<memory>

using std::make_shared;
using std::shared_ptr;

const double infinity=std::numeric_limits<double>::infinity();
const double pi=3.1415926535897932385;

inline double degrees_to_radians(double degrees){
    return degrees*pi/180.0;
}

#include"color.h"
#include"ray.h"
#include"vec3.h"
```

### 7.创建一个法向可视化球和一片空地(空地用绿色表示)
```cpp
#include"rtweekend.h"

#include"hittable.h"
#include"hittable.list.h"
#include"sphere.h"

// color与point3一样,是vec3类型,只是用不同名字来区分不同用途


color ray_color(const ray& r,const hittable & world){
    hit_record rec;
    if(world.hit(r,0,infinity,rec)){
        return 0.5*(rec.normal+color(1,1,1));
    }
    
    vec3 unit_direction=unit_vector(r.direction());
    auto t=0.5*(unit_direction.y()+1.0);
    return (1.0-t)*color(1.0,1.0,1.0)+t*color(0.5,0.7,1.0);
}

int main(){
    //image

    auto aspect_ratio=16.0/9.0;
    int image_width=400;

    //长宽转化,并确保长大于1
    int image_height=static_cast<int>(image_width/aspect_ratio);
    image_height=(image_height<1?1:image_height);

    //world

    hittable_list world;

    world.add(make_shared<sphere>(point3(0,0,-1),0.5));
    world.add(make_shared<sphere>(point3(0,-100.5,-1),100));

    //Camera

    auto focal_length=1.0;
    auto viewport_height=2.0;
    auto viewport_width=viewport_height*(static_cast<double>(image_width)/image_height);
    auto camera_center=point3(0,0,0);

    //创建窗口边界水平和垂直方向的向量
    auto viewport_u=point3(viewport_width,0,0);
    auto viewport_v=point3(0,-viewport_height,0);

    //计算向量u,v的增量
    auto pixel_delta_u=viewport_u/image_width;
    auto pixel_delta_v=viewport_v/image_height;

    //计算左上角像素坐标
    auto viewport_upper_left=camera_center-vec3(0,0,focal_length)-viewport_u/2-viewport_v/2;
    auto pixel00_loc=viewport_upper_left+0.5*(pixel_delta_v+pixel_delta_u);

    //rander

    std::cout<<"P3\n"<<image_width<<" "<<image_height<<"\n255\n";

    for(int j=0;j<image_height;++j){
        std::clog<<"\rScrean remaining:"<<(image_height-j)<<' '<<std::flush;
        for(int i=0;i<image_width;++i){
            auto pixle_center=pixel00_loc+(i*pixel_delta_u)+(j*pixel_delta_v);//读取像素坐标
            auto ray_direction=pixle_center-camera_center;

            ray r(camera_center,ray_direction);

            color pixel_color=ray_color(r,world);
            write_color(std::cout,pixel_color);
        }
    }
    std::clog<<"\rDone\n";
}
```

***操作整合:***
1. 已对基础的向量,坐标,光线的基础集合以及相关计算函数进行了明确的定义;   
2. 同时,由球体交互为引,定义了光线与物体交互的存在判定,和法向等信息的计算读取与存储;   
3. 最后,推广至物体类,创建了场景管理器hittable_list.h;   
4. 同时,将运用最广的一些头文件整合到了rtweekend.h中,并加入了一些常用的数学常量与函数,来简化其他文件的头文件输入

***后续操作:尝试实现抗锯齿效果***

### 8.创建区间类

***目的:限定渲染区间,而不是将所有物体都进行渲染***

```cpp
#pragma once

#include"rtweekend.h"

class interval{
    public:
        double min;
        double max;

        interval():min(+infinity),max(-infinity){}

        interval(double min,double max) : min(min),max(max){}

        double size() const{
            return max-min;
        }

        bool contains(double x) const{
            return x>=min&& x<=max;
        }

        bool surrounds(double x) const{
            return x>min&&x<max;
        }

        static const interval empty,universe;
};

const interval interval::empty=interval(+infinity,-infinity);
const interval interval::universe=interval(-infinity,+infinity);
```

## 七.摄像机移动
### 1. 创建摄像机类
```cpp
#pragma once

#include"hittable.h"

class camera{
    public:
        double aspect_ratio=1.0;
        int     image_width=100;

        void render(const hittable& world) {
            initialize();

            std::cout<<"P3\n"<<image_width<<" "<<image_height<<"\n255\n";

            for(int j=0;j<image_height;++j){
                std::clog<<"\rScrean remaining:"<<(image_height-j)<<' '<<std::flush;
                for(int i=0;i<image_width;++i){
                    auto pixle_center=pixel00_loc+(i*pixel_delta_u)+(j*pixel_delta_v);//读取像素坐标
                    auto ray_direction=pixle_center-center;

                    ray r(center,ray_direction);

                    color pixel_color=ray_color(r,world);
                    write_color(std::cout,pixel_color);
                }
            }

            std::clog<<"\nDone.\n";
        }



    private:
        int image_height;
        point3 center;
        point3 pixel00_loc;
        vec3 pixel_delta_u;
        vec3 pixel_delta_v;

        void initialize(){
            image_height=static_cast<int>(image_width/aspect_ratio);
            image_height=(image_height<1?1:image_height);

            center=point3(0,0,0);

            auto focal_length=1.0;
            auto viewport_height=2.0;
            auto viewport_width=viewport_height*(static_cast<double>(image_width)/image_height);

            auto viewport_u=vec3(viewport_width,0,0);
            auto viewport_v=vec3(0,-viewport_height,0);

            pixel_delta_u=viewport_u/image_width;
            pixel_delta_v=viewport_v/image_height;

            auto viewport_upper_left=center-vec3(0,0,focal_length)-viewport_u/2-viewport_v/2;
            pixel00_loc=viewport_upper_left+0.5*(pixel_delta_u+pixel_delta_v);
        }
        
        color ray_color(const ray& r,const hittable & world) const{
            hit_record rec;
            if(world.hit(r,interval(0,infinity),rec)){
                return 0.5*(rec.normal+color(1,1,1));
            }
            
            vec3 unit_direction=unit_vector(r.direction());
            auto t=0.5*(unit_direction.y()+1.0);
            return (1.0-t)*color(1.0,1.0,1.0)+t*color(0.5,0.7,1.0);
        }

};
```
### 2.简化main函数
```cpp
#include"rtweekend.h"

#include"hittable.h"
#include"hittable_list.h"
#include"sphere.h"
#include"camera.h"

// color与point3一样,是vec3类型,只是用不同名字来区分不同用途

int main(){
    hittable_list world;

    world.add(make_shared<sphere>(point3(0,0,-1),0.5));
    world.add(make_shared<sphere>(point3(0,-100.5,-1),100));

    camera cam;

    cam.aspect_ratio=16.0/9.0;
    cam.image_width=400;

    cam.render(world);
}
```

## 八.抗锯齿的实现

### 1.抗锯齿的实现思路

#### 无限分辨率与采样
    现实世界是连续的,拥有无限分辨率的.为达成这种效果,我们可以通过***对每个像素进行多次采样提取并求平均值***的方式来模拟

#### 了解**点采样**问题
    * **什么是点采样:** 我们为每一个像素发射一条光线穿过其几何中心,最后返回颜色时会将这一块像素内的颜色都赋予其几何中心处的颜色;
    * **点采样的弊端** 1. 显而易见,用一块区域中心颜色来来填充整块区域的话,会使边缘呈现出"阶梯状",也就是常说的"锯齿";    
        2. 举个例子,我们要渲染一个8*8的黑白棋盘,但我们只发射了4条射线去读取信息,如果4条射线均指向白/黑色,那么返回的结果就是一个纯白/纯黑棋盘,这样会丢失大量信息
        3. 采样栅栏或远处重复的复杂图案时,会出现摩尔纹
#### 结合"无限分辨率"与"点采样"
    * 提出合理的解决方案---***超采样***
    * **核心思路:** 将读取点信息转化为读取区域信息后积分(求平均值);
    * **实现思路:**     
        1. *增加样本量*:向一个像素发射n(n>1)条射线;
        2. *射线偏移*:将射线随机偏移,以防读取到同一点上的信息;
        3. *取平均值*:读取每条光线返回的颜色,求和取平均值;

#### 代码实现
1. 随机数工具函数
```cpp
#include<random>
#include<cstdio>

inline double random_double(){
    //返回[0,1)之间的随机数
    return std::rand()/(RAND_MAX+1.0);
}

inline double random_double(double min,double max){
    static std::uniform_real_distribution<double> distribution(0.0,1.0);
    static std::mt19937 generator;
    return distribution(generator)*(max-min)+min;
}

inline double random_double(double min,double max){
    //返回[min,max)之间的随机数
    return min+(max-min)*random_double();
}
```
#### 代码解析:
    1. **static的用法:** 将*distribution* 和*generator* 声明为*static*能确保它们只被初始化一次,避免每次调用函数时都对它们初始化而获得重复序列,确保下一次的调用沿用上一次使用的状态;
    2. **mt19937:**  这是**梅森旋转算法** 相比*rand()* 它的随机性更好,周期更长;
    3. **std::uniform_real_distribution<double>(x,y)** :其作用是将*mt19937 generator*产生的原始整数映射到指定区间[x,y](初始为0-1)中,并确保区间内每个数字出现的概率相等;




