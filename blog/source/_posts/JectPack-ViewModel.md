---
title: JectPack之ViewModel
tags: 
  - jectpack
categories:
  - [jectpack] 
top_image: https://rmt.dogedoge.com/fetch/fluid/storage/bg/dojm2h.png?w=1920&q=100&fmt=webp
excerpt: ViewMode主要用来管理和存储与UI绑定的数据，同时还与UI的生命周期相关联，viewModel可以保留之前读取到的数据不会因为Acitvity的销毁而丢失。 ViewMode的推出正是基于上述问题给出的解决方案，完美高效的将UI控制器和数据业务进行分离，UI控制器只负责UI展示相关的工作，数据业务只负责获取数据的相关工作。
---
# 概述
ViewMode主要用来管理和存储与UI绑定的数据，同时还与UI的生命周期相关联，viewModel可以保留之前读取到的数据不会因为Acitvity的销毁而丢失。

ViewMode的推出正是基于上述问题给出的解决方案，完美高效的将UI控制器和数据业务进行分离，UI控制器只负责UI展示相关的工作，数据业务只负责获取数据的相关工作。

**区别**

onSaveInstanceState()是被系统在activity stopped但没有finish调用，可以在应用被杀死的时候存储数据，而viewmodel不可以


## 使用方法

```bash
class MyViewModel: ViewModel() {
    var number:Int = 0
}
```

### 页面中调用

```bash
myViewModel = ViewModelProviders.of(this).get(MyViewModel::class.java)
```

## 源码：


```bash
/**
 * Creates a {@link ViewModelProvider}, which retains ViewModels while a scope of given Activity
 * is alive. More detailed explanation is in {@link ViewModel}.
 * <p>
 * It uses the given {@link Factory} to instantiate new ViewModels.
 *
 * @param activity an activity, in whose scope ViewModels should be retained
 * @param factory  a {@code Factory} to instantiate new ViewModels
 * @return a ViewModelProvider instance
 */
@NonNull
@MainThread
public static ViewModelProvider of(@NonNull FragmentActivity activity,
        @Nullable Factory factory) {
    Application application = checkApplication(activity);
    if (factory == null) {
        factory = ViewModelProvider.AndroidViewModelFactory.getInstance(application);
    }
    return new ViewModelProvider(activity.getViewModelStore(), factory);
}
```


每个Activity中有getViewModelStore在ViewModelStroe类中有一个Map集合用来存放viewModel
调用Providers.of方法，会调用AndroidViewModelFactory.getInstance，可以调用create方法来创建ViewModel实例。

### 总结：
1.  ViewModel 是从 ViewModelproviders 通过 viewmodelandroidfactory 工厂 和 holderfragment 获取到一个 viewmodelprovider ，调用viewmodelprovider 的 get 方法从 hashmap 获取对应的 viewmodel。 管理 hashmap 的ViewModelStore 在 holderfragment 里面，而这个holderfragment为什么不会随着 activity 的重建而不销毁呢，这是因为对应的 holderfragment 设置了setRetainInstance(true)。不会随着重建 activity 而销毁。
2. 可以创建多个 viewmodel，而 activity 和 fragment 中 holderfragment 只有一个。

### AndroidViewModel
构造函数中多传入一个参数 Application  然后在viewModel中可以通过getApplication获取，可以用于处理一些Sharedprefres存储等。

