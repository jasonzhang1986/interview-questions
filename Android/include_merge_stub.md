本篇是对官方文档的简单翻译，英文水平好的建议看原文 [官方文档链接](https://developer.android.com/training/improving-layouts/reusing-layouts.html#Include)

## 通过 < include / > 重用布局
虽然 Android 提供了众多的 widget 来给小型的和可重用的交互使用，但你仍然有一些需求是需要使用特殊布局的较大组件。为了更加有效的重复使用完整的特殊布局，你可以使用 < inculde /> 和 < merge />标签在当前布局中嵌入另一个布局。

重用布局功能强大，它允许你创建可重复使用的复杂布局。例如，一个“是/否”按钮面板，或包含描述文本的自定义进度条。这也意味着你的应用程序中在多个布局共有的任何元素都可以提取出来单独管理，然后包含在各个布局中。

### 创建一个可重复使用的布局
如果你已经知道这个布局需要重复使用，那么新建一个 xml 文件然后定义这个布局。举个例子，这个布局内容是定义一个 title bar，然后可以被每个 activity 包含使用( titlebar.xml ):
```xml
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:background="@color/titlebar_bg"
    tools:showIn="@layout/activity_main" >

    <ImageView android:layout_width="wrap_content"
               android:layout_height="wrap_content"
               android:src="@drawable/gafricalogo" />
</FrameLayout>
```
根视图应该可以适配需要用到的每个布局，可以在每个布局中正常展示。

**Note**：tools:showIn 是一个特殊的属性，它仅仅在 AndroidStudio 的 design-time 使用，当编译的时候会被 remove 掉。它的作用是指定一个布局来包含当前视图，这样你可以通过 AndroiStudio 的预览功能看到嵌入后的效果。

### 使用< include /> 标签
在需要添加可重用组件的布局中添加 < include /> 标签。
