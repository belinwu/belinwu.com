---
layout: post
title: Android ImageView的scaleType属性
date: 2017-01-05 20:51
categories: Android
---

`ImageView` 的 `android:scaleType` 属性的含义如下：

*控制如何调整图片大小和图片位移以匹配 `ImageView` 的大小（即宽和高）。*

它有以下几个值：

| 属性值 | 代码值 |
|----------------|-------------------------------------|
| `center`       | `ImageView.ScaleType.CENTER`        |
| `centerCrop`   | `ImageView.ScaleType.CENTER_CROP`   |
| `centerInside` | `ImageView.ScaleType.CENTER_INSIDE` |
| `fitCenter`    | `ImageView.ScaleType.FIT_CENTER`    |
| `fitStart`     | `ImageView.ScaleType.FIT_START`     |
| `fitEnd`       | `ImageView.ScaleType.FIT_END`       |
| `fitXY`        | `ImageView.ScaleType.FIT_XY`        |
| `matrix`       | `ImageView.ScaleType.MATRIX`        |

默认值为：`fitCenter`。

## 1. 精解

以下使用两个宽高均为：`400x400px` 的 `ImageView` 控件来分别展示宽高为：`125x127px` 和 `678x674px` 的图片。

`ImageView` 控件的背景色均为红色，他们的父视图背景色为灰色。

### 1.1 `center`

图片保持原始宽高。

将图片居中展示在 `ImageView` 里，超出 `ImageView` 宽高的部分将被截去。

![image_scale_center](/assets/img/image_scale_center.png)

### 1.2 `centerCrop`

等比例缩放图片使得图片的宽高都分别 `>=` `ImageView` 的宽高（减去 `padding`），接着截取缩放后的图片的中间部分。

![image_scale_center_crop.png](/assets/img/image_scale_center_crop.png)

### 1.3 `centerInside`

图片居中展示在 `ImageView` 里，若图片大小 `>` `ImageView` 大小，则将图片等比列缩小，使得整张图片都能完全展示在 
`ImageView` 里，也就是说图片水平和垂直方向至少有一方向上的边与 `ImageView` 的边对齐。

![image_scale_center_inside.png](/assets/img/image_scale_center_inside.png)

### 1.4 `fitCenter` (默认值)

等比列缩放图片使得图片水平和垂直方向至少有一方向上的边与 `ImageView` 的边对齐，最后将图片居中显示在 `ImageView` 里。

![image_scale_fit_center.png](/assets/img/image_scale_fit_center.png)

### 1.5 `fitStart`

与 `fitCenter` 缩放规则相同，最后使图片的上边和左边都分别与 `ImageView` 的上边和左边对齐。

![image_scale_fit_start.png](/assets/img/image_scale_fit_start.png)

### 1.6 `fitEnd`

与 `fitCenter` 缩放规则相同，最后使图片的下边和右边都分别与 `ImageView` 的下边和右边对齐。

![image_scale_fit_end.png](/assets/img/image_scale_fit_end.png)

### 1.7 `fitXY`

不等比例缩放图片，使得图片的四边与 `ImageView` 的四边都对齐。

![image_scale_fit_xy.png](/assets/img/image_scale_fit_xy.png)

### 1.8 `matrix`

矩阵被用于在图形绘制时操作画布。对于 `ImageView` 而言，矩阵可以用于位移图片、翻转图片、旋转图片等，尤其是涉及用户手势相关的操作。

待续

![image_scale_matrix.png](/assets/img/image_scale_matrix.png)

## 2. 总结

- 当图片比 `ImageView` 小时，`center` 和 `centerInside` 的展示效果是相同的。
- 当图片比 `ImageView` 大时，`centerInside` 和 `fitCenter` 的展示效果是相同的。
- `fitCenter`、`fitStart` 和 `fitEnd` 这三个的缩放规则是一样的，区别在于缩放后图片位移不同。
- 当希望匹配图片的宽高比，尤其是在使用 `fitXY` 这一属性值时，可以设置 `adjustViewBounds=true`。

## 3. 参考资料

- [Working with the ImageView](https://github.com/codepath/android_guides/wiki/Working-with-the-ImageView)
- [Tanis.7x's Answer](http://stackoverflow.com/a/31788451/2905470)