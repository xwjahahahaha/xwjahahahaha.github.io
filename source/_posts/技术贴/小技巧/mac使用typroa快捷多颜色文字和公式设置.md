---
title: mac使用typora快捷多颜色文字和公式设置
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2021-03-24 14:36:46
---

# mac使用typora快捷多颜色文字/公式设置

## 需要的工具:

* typora
* Alfred

typora搜索官网下载,Alfred可以[点此](https://xclient.info/s/alfred.html#versions)下载

<!-- more -->

## 设置过程:

1. 在typora上设置好内联公式设置:(不使用多颜色公式的可以跳过)

   打开偏好设置:勾选即可

   ![ACLZvi](http://xwjpics.gumptlu.work/qinniu_uPic/ACLZvi.png)

2. 打开Alfred偏好设置 :

   ![uz9Qof](http://xwjpics.gumptlu.work/qinniu_uPic/uz9Qof.png)

3. 点击左侧下面的加号,新建一个文字拓展分组:

	 ![c9eOVY](http://xwjpics.gumptlu.work/qinniu_uPic/c9eOVY.png)

   ![i85r1e](http://xwjpics.gumptlu.work/qinniu_uPic/i85r1e.png)

4. 点击右侧加号,在typora分组下新建一个文字拓展

   ![waghXp](http://xwjpics.gumptlu.work/qinniu_uPic/waghXp.png)

5. 黄颜色公式拓展示例:

   ![cdwTTW](http://xwjpics.gumptlu.work/qinniu_uPic/cdwTTW.png)
   
   ![THBN1k](http://xwjpics.gumptlu.work/qinniu_uPic/THBN1k.png)
   
   ![mURWhi](http://xwjpics.gumptlu.work/qinniu_uPic/mURWhi.png)

6. 拓展内容

   所有的公式颜色:
   
   ```shell
   $\textcolor{GreenYellow}{GreenYellow} $$\textcolor{Yellow}{Yellow}$$\textcolor{Goldenrod}{Goldenrod} $$\textcolor{Dandelion}{Dandelion}$$\textcolor{Apricot}{Apricot} $$\textcolor{Peach}{Peach}$$\textcolor{Melon}{Melon} $$\textcolor{YellowOrange}{YellowOrange}$$\textcolor{Orange}{Orange} $$\textcolor{BurntOrange}{BurntOrange}$$\textcolor{Bittersweet}{Bittersweet}$$\textcolor{RedOrange}{RedOrange} $$\textcolor{Mahogany}{Mahogany}$$\textcolor{Maroon}{Maroon} $$\textcolor{BrickRed}{BrickRed}$$\textcolor{Red}{Red} $$\textcolor{OrangeRed}{OrangeRed}$$\textcolor{RubineRed}{RubineRed}$$\textcolor{WildStrawberry}{WildStrawberry}$$\textcolor{Salmon}{Salmon}$$\textcolor{CarnationPink}{CarnationPink}$$\textcolor{Magenta}{Magenta} $$\textcolor{VioletRed}{VioletRed}$$\textcolor{Rhodamine}{Rhodamine} $$\textcolor{Mulberry}{Mulberry}$$\textcolor{RedViolet}{RedViolet} $$\textcolor{Fuchsia}{Fuchsia}$$\textcolor{Lavender}{Lavender} $$\textcolor{Thistle}{Thistle}$$\textcolor{Orchid}{Orchid} $$\textcolor{DarkOrchid}{DarkOrchid}$$\textcolor{Purple}{Purple} $$\textcolor{Plum}{Plum}$$\textcolor{Violet}{Violet} $$\textcolor{RoyalPurple}{RoyalPurple}$$\textcolor{BlueViolet}{BlueViolet}$$\textcolor{Periwinkle}{Periwinkle}$$\textcolor{CadetBlue}{CadetBlue}$$\textcolor{CornflowerBlue}{CornflowerBlue}$$\textcolor{MidnightBlue}{MidnightBlue}$$\textcolor{NavyBlue}{NavyBlue} $$\textcolor{RoyalBlue}{RoyalBlue}$$\textcolor{Blue}{Blue} $$\textcolor{Cerulean}{Cerulean}$$\textcolor{Cyan}{Cyan} $$\textcolor{ProcessBlue}{ProcessBlue}$$\textcolor{SkyBlue}{SkyBlue} $$\textcolor{Turquoise}{Turquoise}$$\textcolor{TealBlue}{TealBlue} $$\textcolor{Aquamarine}{Aquamarine}$$\textcolor{BlueGreen}{BlueGreen} $$\textcolor{Emerald}{Emerald}$$\textcolor{JungleGreen}{JungleGreen}$$\textcolor{SeaGreen}{SeaGreen} $$\textcolor{Green}{Green}$$\textcolor{ForestGreen}{ForestGreen}$$\textcolor{PineGreen}{PineGreen} $$\textcolor{LimeGreen}{LimeGreen}$$\textcolor{YellowGreen}{YellowGreen}$$\textcolor{SpringGreen}{SpringGreen}$$\textcolor{OliveGreen}{OliveGreen}$$\textcolor{RawSienna}{RawSienna} $$\textcolor{Sepia}{Sepia}$$\textcolor{Brown}{Brown} $$\textcolor{Tan}{Tan}$$\textcolor{Gray}{Gray} $$\textcolor{Black}{Black}$
   ```
   
   **文字的拓展**
   
   <font color='red'>color属性值:   </font>
   
   | 值               | 描述                                              |
   | :--------------- | :------------------------------------------------ |
   | *颜色名*         | 通过颜色名指定文本颜色（例如：“red”）             |
   | *十六进制颜色值* | 通过十六进制颜色值指定文本颜色（例如：“#ff0000”） |
   | *RGB颜色值*      | 通过RGB颜色值指定文本颜色（例如：“rgb(255,0,0)”） |
   
   ```html
   <font color='yellow'>    </font>
   ```
   
   ![EXV8M6](http://xwjpics.gumptlu.work/qinniu_uPic/EXV8M6.png)
   
   

