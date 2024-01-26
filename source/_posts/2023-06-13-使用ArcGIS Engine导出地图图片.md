---
title: 使用ArcGIS Engine将地图导出为图片
date: 2023-06-13 17:55:00 +0800
categories: [ArcGIS]
tags: [arcgis]
mermaid: true
---

ArcGIS Engine本质就是在ArcObjects的基础上封装的 一组高级接口，其核心构建在ArcObjects之上，完全可扩展，也属于ArcGIS Engine的产品。开发者在桌面端或移动端应用程序中可以通过 **.COM** 的API、.NET、Java和C++调用ArcGIS Engine的GIS data，maps和geoprocessing 脚本。

先梳理下需求：

1. 两个图层：底图为影像图，另一个为采样点图层
2. 以每个点为中心点，设置比例尺为1:500，尺寸为800x500
3. 当前视图范围内仅显示导出点，其他点隐藏

安装 **ArcObjectsSDKNet** ，我这里使用的版本是10.4.1

> 注意：ArcObjectsSKDNet仅支持 VS2013和VS2015版本，如果你的VS版本过高或过低都无法安装！真的奇葩~



新建解决方案，引用程序集，在引用管理器中选择扩展，引入以下几个程序集。

![20240124162048.png (781×542) (raw.githubusercontent.com)](https://raw.githubusercontent.com/liugt34/imagegallery/main/20240124162048.png)

```
ESRI.ArcGIS.AxControls
ESRI.ArcGIS.Carto
ESRI.ArcGIS.Controls
ESRI.ArcGIS.Display
ESRI.ArcGIS.Geodatabase
ESRI.ArcGIS.Geometry
ESRI.ArcGIS.Output
ESRI.ArcGIS.System
ESRI.ArcGIS.Version
```

右键选择这几个引用，查看属性，将 **嵌入互操作类型** 修改为 ***False***

![20240124162321.png (411×560) (raw.githubusercontent.com)](https://raw.githubusercontent.com/liugt34/imagegallery/main/20240124162321.png)

> 嵌入互操作类型设定为true时，实际上就是不引入互操作集，仅编译用户代码的程序集
>
> 嵌入互操作类型设定为false时，实际就是需要从互操作程序集中获取 COM 类型的类型信息



在Program.cs文件应用程序入口添加如下代码，绑定许可。

```c#
RuntimeManager.Bind(ProductCode.EngineOrDesktop);
```

在Form1.cs中添加地图控件和license控件，其他组件不在此赘述，本文仅列出关键代码。

定位到每个点，并设置比例尺

```c#
var layer = this.mapControl.get_Layer(0) as IFeatureLayer;
var featureClass = layer.FeatureClass
//仅显示当前点过滤器，很重要
var definition = layer as IFeatureLayerDefinition;
definition.DefinitionExpression = "FID=" + i;

var feature = featureClass.GetFeature(i);
name = i.ToString("D3") + "-" + feature.Value[2] + "";
var point = feature.Shape as IPoint;

//设置比例尺
this.mapControl.MapScale = 500;
//定位中心点
this.mapControl.CenterAt(point);

var image = this.Export(this.mapControl.ActiveView, new Size(800, 500), this.mapControl.Extent);
image.Save(dir + "\\" + name + ".jpg");
image.Dispose();

```



导出方法

```C#
private Image Export(IActiveView pActiveView, Size outRect, IEnvelope pEnvelope)
{
	//赋值
	tagRECT rect = new tagRECT();
	rect.left = rect.top = 0;
	rect.right = outRect.Width;
	rect.bottom = outRect.Height;
	try
	{
		// 创建图像,为24位色
		Image image = new Bitmap(outRect.Width, outRect.Height); //, System.Drawing.Imaging.PixelFormat.Format24bppRgb);
		Graphics g = Graphics.FromImage(image);

		// 填充背景色(白色)
		g.FillRectangle(Brushes.White, 0, 0, outRect.Width, outRect.Height);

		int dpi = (int)(outRect.Width / pEnvelope.Width);

		pActiveView.Output(g.GetHdc().ToInt32(), 96, ref rect, pEnvelope, null);

		g.ReleaseHdc();

		return image;
	}
	catch (Exception ex)
	{
		return null;
	}
}
```



结果如下

![20210124164855.jpg](https://raw.githubusercontent.com/liugt34/imagegallery/main/20210124164855.jpg)



在网上搜索到另外一种方法，使用ArcEngine中ActiveView自带方法导出，代码如下

```C#
private void Export(IActiveView pView, Size outRect, string outPath)
{
	try
	{
		//参数检查
		//根据给定的文件扩展名，来决定生成不同类型的对象
		ESRI.ArcGIS.Output.IExport export = null;
		if (outPath.EndsWith(".jpg"))
		{
			export = new ESRI.ArcGIS.Output.ExportJPEGClass();
		}
		else if (outPath.EndsWith(".tiff"))
		{
			export = new ESRI.ArcGIS.Output.ExportTIFFClass();
		}
		else if (outPath.EndsWith(".bmp"))
		{
			export = new ESRI.ArcGIS.Output.ExportBMPClass();
		}
		else if (outPath.EndsWith(".emf"))
		{
			export = new ESRI.ArcGIS.Output.ExportEMFClass();
		}
		else if (outPath.EndsWith(".png"))
		{
			export = new ESRI.ArcGIS.Output.ExportPNGClass();
		}
		else if (outPath.EndsWith(".gif"))
		{
			export = new ESRI.ArcGIS.Output.ExportGIFClass();
		}

		export.ExportFileName = outPath;
		IEnvelope pEnvelope = pView.Extent;
		//导出参数           
		export.Resolution = 300;
		tagRECT exportRect = new tagRECT();
		exportRect.left = exportRect.top = 0;
		exportRect.right = outRect.Width;
		exportRect.bottom = (int)(exportRect.right * pEnvelope.Height / pEnvelope.Width);
		ESRI.ArcGIS.Geometry.IEnvelope envelope = new ESRI.ArcGIS.Geometry.EnvelopeClass();
		//输出范围
		envelope.PutCoords(exportRect.left, exportRect.top, exportRect.right, exportRect.bottom);
		export.PixelBounds = envelope;
		//可用于取消操作
		ITrackCancel pCancel = new CancelTrackerClass();
		export.TrackCancel = pCancel;
		pCancel.Reset();
		//点击ESC键时，中止转出
		pCancel.CancelOnKeyPress = true;
		pCancel.CancelOnClick = false;
		pCancel.ProcessMessages = true;
		//获取handle
		System.Int32 hDC = export.StartExporting();
		//开始转出
		pView.Output(hDC, (System.Int16)export.Resolution, ref exportRect, pEnvelope, pCancel);
		bool bContinue = pCancel.Continue();
		//捕获是否继续
		if (bContinue)
		{
			export.FinishExporting();
			export.Cleanup();
		}
		else
		{
			export.Cleanup();
		}
		bContinue = pCancel.Continue();
	}
	catch (Exception excep)
	{
		//错误信息提示
	}
}
```



