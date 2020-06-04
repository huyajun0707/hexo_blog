---
title: JectPack之LiveData
tags: 
  - jectpack
categories:
  - [jectpack] 
top_image: https://rmt.dogedoge.com/fetch/fluid/storage/bg/dojm2h.png?w=1920&q=100&fmt=webp
excerpt: LiveData是一个数据持有类，可以添加观察者处理变更事务，不同于普通的观察者，它最重要的特性就是遵从应用程序的生命周期，如在Activity中如果数据更新了但Activity已经是destroy状态，LivaeData就不会通知Activity(observer)。当然。LiveData的优点还有很多，如不会造成内存泄漏等。
---
LiveData是一个数据持有类，可以添加观察者处理变更事务，不同于普通的观察者，它最重要的特性就是遵从应用程序的生命周期，如在Activity中如果数据更新了但Activity已经是destroy状态，LivaeData就不会通知Activity(observer)。当然。LiveData的优点还有很多，如不会造成内存泄漏等。

LiveData通常会配合ViewModel来使用，ViewModel负责触发数据的更新，更新会通知到LiveData，然后LiveData再通知活跃状态的观察者。
## 使用：

```bash
nameViewModel=ViewModelProviders.of(this,SavedStateVMFactory(this)).get(NameViewModel::class.java)

// Create the observer which updates the UI.
val nameObserver = object : Observer<Int> {
    override fun onChanged(@Nullable newName: Int) {
        // Update the UI, in this case, a TextView.
        tvGet.setText(newName)
    }
}

// Observe the LiveData, passing in this activity as the LifecycleOwner and the observer.
nameViewModel.number.observe(this, nameObserver)
```

## 源码：
ViewModel中持有LiveData
viewmodel.livedata.observe(this,observer)

### 添加观察者
lifecycle是Activity的一个观察者持有类，可以添加新的观者者，并且可以获取到当前生命周期所处的状态

```bash
@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
    assertMainThread("observe");
    //判断当前生命周期状态，如果是已销毁不进行下一步操作
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // ignore
        return;
    }
    //新创建lifecycleBoundObserver将owner和observer保存起来
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
   //将observer和wrapper分别作为key和value存入Map中，
   //putIfAbsent()方法会判断如果 value 已经能够存在，就返回，否则返回null。
   // 如果返回existing为null，说明以前没有添加过这个观察者，就将 LifecycleBoundObserver 作为 owner 生命周期的观察者

     ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    owner.getLifecycle().addObserver(wrapper);
}
```

#### observeForever()
当数据发生改变时不管组件处于什么状态都会收到回调，除非手动将观察者移除。
#### observer()
当组件处于STARTED和RESUMED状态下才会收到

### 通知观察者更新

```bash
@MainThread
protected void setValue(T value) {
    //必须在主线程setValue
    assertMainThread("setValue");
    mVersion++;
    mData = value;
    //分发给各个观察者
    dispatchingValue(null);
}
 //分发 如果传入null则通知所有观察者，反之只通知传入的观察者
void dispatchingValue(@Nullable ObserverWrapper initiator) {
    if (mDispatchingValue) {
        mDispatchInvalidated = true;
        return;
    }
    mDispatchingValue = true;
    do {
        mDispatchInvalidated = false;
        if (initiator != null) {
            considerNotify(initiator);
            initiator = null;
        } else {
            //遍历所有的观察者，通知
            for (Iterator<Map.Entry<Observer<? super T>, ObserverWrapper>> iterator =
                    mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                        //具体方法
                considerNotify(iterator.next().getValue());
                if (mDispatchInvalidated) {
                    break;
                }
            }
        }
    } while (mDispatchInvalidated);
    mDispatchingValue = false;
}

private void considerNotify(ObserverWrapper observer) {
    //判断是否是活跃的
    if (!observer.mActive) {
        return;
    }
    // Check latest state b4 dispatch. Maybe it changed state but we didn't get the event yet.
    //
    // we still first check observer.active to keep it as the entrance for events. So even if
    // the observer moved to an active state, if we've not received that event, we better not
    // notify for a more predictable notification order.
    if (!observer.shouldBeActive()) {
        observer.activeStateChanged(false);
        return;
    }
    if (observer.mLastVersion >= mVersion) {
        return;
    }
    observer.mLastVersion = mVersion;
    //noinspection unchecked
    //回调观察者的方法
    observer.mObserver.onChanged((T) mData);
}
```

##  Transformations
使用LiveData时，有时候我们想改变LiveData的值在LiveData通知其Observers之前，或者在有些时候我们的数据源Repository会返回不同的LiveData实例，我们需要一种数据转换的功能帮助我们去转换不同数据层的数据，而Transformations就是这样一个工具。
### switchMap
可以生成liveData对象，并添加指定条件
### map
可以转化LiveData对象
