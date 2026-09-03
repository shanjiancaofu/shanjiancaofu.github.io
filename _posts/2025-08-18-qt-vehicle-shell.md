---
title: Qt 车机项目重构（一）：用 QStackedWidget 搭建统一外壳
date: 2025-08-18 20:30:00 +0800
categories: [C++]
tags: [Qt, Qt Widgets, QStackedWidget, 车机]
---

最近整理旧车机项目时，遇到的第一个问题并不是某个页面不能运行，而是音乐、视频、天气、摄像头等功能各自拥有入口和窗口，缺少一个稳定的主程序。功能一多，页面跳转、进程启动和硬件访问就容易纠缠在一起。

因此第一步不是立即把所有旧代码重写，而是先建立一个统一的 Qt Widgets 外壳，让旧功能能够逐步迁移。

## 页面容器的选择

车机界面通常在固定的几个主页面之间切换，并不需要同时显示所有页面。`QStackedWidget` 很适合这种场景：它保存多个子页面，但同一时间只显示其中一个。

```cpp
auto *pages = new QStackedWidget(this);
pages->addWidget(new HomePage(this));
pages->addWidget(new MusicPage(this));
pages->addWidget(new VideoPage(this));
pages->addWidget(new WeatherPage(this));

connect(tabBar, &BottomTabBar::currentChanged,
        pages, &QStackedWidget::setCurrentIndex);
```

主窗口只负责三件事：创建页面、组织布局、连接导航信号。具体业务放在页面或服务层中，主窗口不直接打开摄像头，也不直接处理媒体文件。

## 固定外壳与可替换页面

项目将整体布局分成上下两部分：上方是页面区域，下方是固定导航栏。

```text
MainWindow
├── QStackedWidget
│   ├── HomePage
│   ├── MusicPage
│   ├── VideoPage
│   ├── WeatherPage
│   ├── CameraPage
│   └── SettingsPage
└── BottomTabBar
```

这种结构的好处是页面迁移不会改变应用入口。某个功能尚未完成时，可以先使用占位页面；旧程序仍能使用时，可以放一个启动按钮；功能迁入主进程后，再将这一页替换为真正的业务页面。

## 导航栏只负责切页

底部导航栏不应该知道音乐如何播放，也不应该知道摄像头设备是 `/dev/video0` 还是 `/dev/video1`。它只维护按钮与页面下标的对应关系。

按钮使用可选中状态表示当前页面：

```css
#tabButton {
  background: transparent;
  border: none;
  color: #687589;
}

#tabButton:checked {
  color: #1769e0;
  border-top: 3px solid #1769e0;
}
```

导航和业务解耦后，即使以后把 Widgets 页面换成 QML，底层服务也不必一起重写。

## 固定分辨率不等于到处写死尺寸

当前原型以 800×480 为基准，这是常见的嵌入式屏幕尺寸。但主布局仍使用 `QVBoxLayout`，页面区域设置伸展因子，边距和间距由容器统一管理。这样既能满足当前屏幕，也为以后适配更高分辨率留下空间。

真正需要避免的是每个控件都使用绝对坐标。绝对坐标在一张演示图上很快，在字体、语言或屏幕比例变化后却很难维护。

## 阶段性重构比一次重写更稳

旧项目已经有可以运行的音乐、视频和天气程序。新外壳先保留这些成果，再逐步将业务逻辑移入服务层。这种方式可以让每一步都保持可运行状态：

1. 建立统一主窗口和导航。
2. 用占位页补齐产品结构。
3. 临时启动旧的独立程序。
4. 提取媒体、天气、摄像头服务。
5. 将页面改为直接消费服务和数据模型。

重构的目标不是让目录看起来更新，而是让系统在迁移期间仍然可验证。统一外壳建立后，后面的进程管理、平台适配和 QML 演进才有清晰的落点。
