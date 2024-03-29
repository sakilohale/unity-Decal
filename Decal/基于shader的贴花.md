
![1](https://raw.githubusercontent.com/sakilohale/unity-Decal/main/Decal/images/1.png)

主要原理：
- 在顶点着色器中，在相机坐标系（view space）得到相机中心坐标以及相机到顶点的射线，然后将两者都变换到物体的模型空间下，后者（射线向量）传入片元着色器会插值为相机到该片元的射线（同相机法重建世界坐标）。
	- 相机中心坐标可以简单设定为(0,0,0,1)。 
	- 相机到顶点的射线：先将顶点变换到ViewSpace，然后顶点 - 相机中心得到射线，再将射线变换到ModelSpace（注意在这一步，由于射线是向量，所以仿射矩阵中平移的那一部分重置为0（矩阵最后一列的xyz），因为平移不改变向量。）。假设射线为$l$，$l.xyz$为射线本身，$l.w$里面存储着顶点的视角空间下的深度值(positonVS.z或者-positionCS.w，可以这里就给转换成正的，或者后面再变)。
- 到了片元着色器，我们得到了 **物体模型空间下** 的相机中心坐标以及相机到片元的射线（其中w分量存储着相机到片元的视角空间深度）。由于我们还能通过屏幕空间UV来采样得到屏幕深度纹理，就能得到当前片元对应像素的经过深度测试和复写的真实深度值（我们会把物体，也称之为投射器，设置为不写入深度）。回顾一下，我们最终希望得到的是投射器与场景中物体相交的片元，该片元可以通过 相机中心坐标 + 相机到该相交片元的射线 来得到。而由于相似三角形原理：$$\frac{相机到该片元射线}{片元视角空间深度值}=\frac{相机到该相交片元射线}{相交片元视角空间深度值}$$
	这样就能得到物体模型空间下的实际片元了，有一些也将其称之为贴花空间(decal space)。
	为什么要在贴花空间中处理呢？这是因为我们一般用一个1 * 1 * 1的cube来当作投射器，在这个cube的模型空间下（cube模型坐标轴心在(0.5,0.5,0.5)），所有顶点的xyz分量的范围固定在了(-0.5,0.5)之间。这有什么好处呢？首先是可以当作UV坐标来采样贴图，例如使用xz两个分量来充当uv（y轴朝上），这里需要加个0.5使其变到(0,1)。然后是裁剪，我们得到的相交片元也许是不在cube内的，这种情况我们就希望裁剪掉，可以用clip(0.5 - decalpos)或者写一个mask = (abs(0.5)>decalpos.x?1:0) * (....y) * (...z)。
	到目前为止基本的贴花就完成了，下面是一些可能会出现的问题以及解决办法。




## 陡峭平面问题

首先是这种，产生原因在于投射器本身与物体产生了在物理上真实但视觉上错误的相交面。例如这里cube投射器和同方向与地面夹角小于90度的物体之间就会产生这种错误，解决办法可以旋转投射器或者不应用在这种场景。

![2](https://raw.githubusercontent.com/sakilohale/unity-Decal/main/Decal/images/2.png)


其次，陡峭的平面会导致贴花剧烈的拉伸。

![3](https://raw.githubusercontent.com/sakilohale/unity-Decal/main/Decal/images/3.png)

这是因为：
1. （次要）相交面过大，导致uv欠采样。
2. （主要）两个相交面之间夹角过小（90到180，越接近90），例如极端情况两个相交面垂直时，用于采样贴图的xz坐标在垂直的相交面上y轴方向采样的颜色值都是一样的，所以会出现这种拉伸的感觉。

怎么解决呢？这里在不对算法本质改变的前提下，只能用遮罩来避免这种拉伸的出现，并不能真正的处理拉伸。如何计算遮罩呢，通过相交片元在模型空间下的法线向量的y轴值，当90度时y轴值为0为最极端。

相交片元在模型空间下的法线向量是对应了模型空间坐标轴的z轴的，也就是模型空间下的xyz对应着tbn，那么就可以用该片元相对于xy两轴的偏导数来得到tb两向量的方向，然后做内积单位化得到相交片元的单位法向量，最后用该法向量的y分量来近似坡面坡度来生成mask。

```
		float mask=(abs(decalPos.x)<0.5?1:0)*(abs(decalPos.y)<0.5?1:0)*(abs(decalPos.z)<0.5?1:0);

                float3 decalNormal=normalize(cross(ddy(decalPos),ddx(decalPos)));

                //return float4(decalNormal,1);

                mask*=decalNormal.y>0.2*_EdgeStretchPrevent?1:0;//边缘拉伸的防止阈值
```

![4](https://raw.githubusercontent.com/sakilohale/unity-Decal/main/Decal/images/4.png)

这样坡度超过了的就会被遮住了，下面是正常的坡度，拉伸并不严重不影响观感。

![5](https://raw.githubusercontent.com/sakilohale/unity-Decal/main/Decal/images/5.png)


这种方法的好处在于简单方便，一个shader就搞定了，缺点就在于算法本质带来的陡峭平面问题，不过用在大部分以平面为主的场景还是绰绰有余了。


## 不正确遮挡问题

例如，我们希望控制人物在其上行走时，贴花不会贴到人物身上。一般而言会用到模板测试。

![6](https://raw.githubusercontent.com/sakilohale/unity-Decal/main/Decal/images/6.png)

例如给贴花shader设置如下模板测试：

```c
Stencil  
{  
    Ref[_StencilRef]   
    Comp[_StencilComp]  
}
```
当_StencilRef = 1，_StencilComp = NotEqual 时，表示该 shader 模板值为 1，值不相等则通过。

那么给人物 shdaer 设置为:
```c
Stencil  
{  
    Ref 1  
    Comp NotEqual  
    Pass Replace  
}
```
这样的话人物覆盖的地方模板值会被替换成 1，而 1 对于贴花而言不能通过，这样就实现了贴花的遮盖。




参考：

[URP下屏幕空间贴花（ScreenSpaceDecal）学习笔记 | 登峰造极者，殊途亦同归。 (lfzxb.top)](https://www.lfzxb.top/screen-space-decal-in-urp-study/)

[URP管线的自学HLSL之路 第三十三篇 屏幕空间贴花 - 哔哩哔哩 (bilibili.com)](https://www.bilibili.com/read/cv7171674)



