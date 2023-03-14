---
title: Use C# WinForms Chart Component
date: 2023-03-13 15:52:00 +0800
categories: [C#, WinForms]
tags: [winform, chart]
mermaid: true
---

今天接到一个临时需求，客户要求做一个桌面小程序用来计算水库库容，里面要有水位库容对照曲线。之前做图表都是用ECharts,WinForms图表没用过，所以临时抱佛脚去网上找了一些案例，现记录如下。
# 创建WinForms项目

打开Visual Studio， 新建项目，选择Windows窗体应用（.NET Framework）.
ps:由于不确定客户Windows版本，所以还是稳当点，选.NET Framework版，框架选用 .NET Framework 4.5.2

![新建项目](https://raw.githubusercontent.com/liugt34/imagegallery/main/20230314092722.jpg)



使用拖拽大法将chart组件拖到Form中



![拖拽Chart组件到Form](https://raw.githubusercontent.com/liugt34/imagegallery/main/20230314093512.jpg)



F5运行一下，空白Form。

![blank form](https://raw.githubusercontent.com/liugt34/imagegallery/main/20230314094802.jpg)



# 添加模拟数据

我们以一周气温为例，X轴为周一到周日，Y轴为温度。

```C#
var days = new string[] { "周一", "周二", "周三", "周四", "周五", "周六", "周日" };
var temperatures = new int[7] { 20, 22, 25, 19, 20, 19, 21};

//设置X轴名称
chart1.ChartAreas[0].AxisX.Title = "日期";
//设置Y轴名称
chart1.ChartAreas[0].AxisY.Title = "温度(°)";

//设置图例名称
chart1.Legends[0].Title = "一周温度曲线图";
chart1.Legends[0].Alignment = StringAlignment.Center;
//设置上下左右
chart1.Legends[0].Docking = Docking.Top;

chart1.Series[0].Name = "温度";
//设置图表类型为曲线
chart1.Series[0].ChartType = SeriesChartType.Spline;
//绑定数据
chart1.Series[0].Points.DataBindXY(days, temperatures);
```

再次 F5 运行，温度曲线显示

![spline](https://raw.githubusercontent.com/liugt34/imagegallery/main/20230314101544.jpg)



# 设置样式

我们发现曲线已经显示，但是还比较简陋，对应日期的数据未能明显标识出来，此时我们需要添加对应的点我们在设置图标类型的下面为曲线设置点标记。

```C#
//设置图表类型为曲线
chart1.Series[0].ChartType = SeriesChartType.Spline;
//在点位上显示具体值
chart1.Series[0].IsValueShownAsLabel = true;
//设置标记样式
chart1.Series[0].MarkerStyle = MarkerStyle.Circle;
chart1.Series[0].MarkerSize = 10;
chart1.Series[0].MarkerColor = Color.Orange;
```

再次F5运行，对应点位已标记出来并显示了对应的值。

![](https://raw.githubusercontent.com/liugt34/imagegallery/main/20230314103355.jpg)


# 多个曲线设置

新增一个需求，显示一天中最高温度和最低温度曲线，这次我们使用图形界面设置另一条曲线

选中Chart组件右键属性，在弹出的属性列表中找到Series,点击Series弹出设置框，新增一个Series并设置各种属性如图

![](https://raw.githubusercontent.com/liugt34/imagegallery/main/20230314104239.jpg)



新增最低温度模拟数据并绑定到Series中

```C#
var min = new int[7] { 16, 15, 20, 14, 16, 14, 18 };

//绑定数据
chart1.Series[1].Name = "最低温度";
chart1.Series[1].Points.DataBindXY(days, min);
```

F5运行，成功显示最低温度曲线。是不是更简单 O(∩_∩)O 。

![20230314105351.jpg (928×516) (raw.githubusercontent.com)](https://raw.githubusercontent.com/liugt34/imagegallery/main/20230314105351.jpg)

# 新增其他要素曲线

客户说了：我还要显示空气湿度曲线。 思考一下，这时如果通过再加一条Series是否可行。答案是否定的。因为当前Y轴为温度，虽然通过新增Series可以显示出来，但是数值比例全都是错误的，此时我们需要新增一个Y轴来显示湿度。

通过图形界面，添加一个新的Series,并设置参数

```c#
var humidities = new double[7] { 0.54, 0.52, 0.5, 0.63, 0.61, 0.59, 0.51 };

//设置Y2轴名称
chart1.ChartAreas[0].AxisY2.Title = "空气湿度";
//设置轴线为虚线，跟温度区分开
chart1.ChartAreas[0].AxisY2.MajorGrid.LineDashStyle = ChartDashStyle.Dot;
chart1.ChartAreas[0].AxisY2.LineDashStyle = ChartDashStyle.Solid;
chart1.Series[2].Name = "湿度";
//重点:设置为辅助轴
chart1.Series[2].YAxisType = AxisType.Secondary;
chart1.Series[2].Points.DataBindXY(days, humidities);
```

成果如下：

![](https://raw.githubusercontent.com/liugt34/imagegallery/main/20230314114933.jpg)

完整代码如下

```C#
using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.Data;
using System.Drawing;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Windows.Forms;
using System.Windows.Forms.DataVisualization.Charting;

namespace WindowsFormsApp1
{
    public partial class Form1 : Form
    {
        public Form1()
        {
            InitializeComponent();

            this.Load += Form1_Load;
        }

        private void Form1_Load(object sender, EventArgs e)
        {
            var days = new string[] { "周一", "周二", "周三", "周四", "周五", "周六", "周日" };
            var max = new int[7] { 20, 22, 25, 19, 20, 19, 21};
            var min = new int[7] { 16, 15, 20, 14, 16, 14, 18 };
            var humidities = new double[7] { 0.54, 0.52, 0.5, 0.63, 0.61, 0.59, 0.51 };
            //设置X轴名称
            chart1.ChartAreas[0].AxisX.Title = "日期";
            chart1.ChartAreas[0].AxisX.MajorGrid.LineDashStyle = ChartDashStyle.NotSet;
            chart1.ChartAreas[0].AxisX.LineDashStyle = ChartDashStyle.Dash;
            //设置Y轴名称
            chart1.ChartAreas[0].AxisY.Title = "温度(°)";

            //设置图例名称
            chart1.Legends[0].Title = "一周温度曲线图";
            chart1.Legends[0].Alignment = StringAlignment.Center;
            //设置上下左右
            chart1.Legends[0].Docking = Docking.Top;

            chart1.Series[0].Name = "最高温度";
            //设置图表类型为曲线
            chart1.Series[0].ChartType = SeriesChartType.Spline;
            //在点位上显示具体值
            chart1.Series[0].IsValueShownAsLabel = true;
            chart1.Series[0].MarkerStyle = MarkerStyle.Circle;
            chart1.Series[0].MarkerSize = 10;
            chart1.Series[0].MarkerColor = Color.Orange;
            //绑定数据
            chart1.Series[0].Points.DataBindXY(days, max);

            chart1.Series[1].Name = "最低温度";
            chart1.Series[1].Points.DataBindXY(days, min);
            
            //设置Y2轴名称
            chart1.ChartAreas[0].AxisY2.Title = "空气湿度";
            //设置轴线为虚线，跟温度区分开
            chart1.ChartAreas[0].AxisY2.MajorGrid.LineDashStyle = ChartDashStyle.Dot;
            chart1.ChartAreas[0].AxisY2.LineDashStyle = ChartDashStyle.Solid;
            chart1.Series[2].Name = "湿度";
            //重点:设置为辅助轴
            chart1.Series[2].YAxisType = AxisType.Secondary;
            chart1.Series[2].Points.DataBindXY(days, humidities);
        }
    }
}

```





