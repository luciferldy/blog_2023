# ViewModel 如何在屏幕旋转后不重建

我们在 Android 开发中使用 MVVM 架构时一定会接触到 ViewModel ，使用 ViewModel 的好处是一方面它是 jetpack 提供了下 ViewModel 和 Fragment/Activity 以及 Corountine 比较丰富的扩展方法，比如 `viewModels()` 和 `activityViewModels()` 可以用来创建 ViewModel，而 `viewModel.viewModelScope` 可以用来启动和 ViewModel 本身生命周期绑定的协程。

在屏幕发生旋转时，如果 AndroidManifest.xml 没有声明 activity `configChanges` 关于屏幕旋转相关的属性，就会导致 activity 的重建，通过日志或者断点可以发现，重建前后的 activity `hashCode` 是不一致的。如果我们打印 ViewModel 的 hashCode ，就会发现 ViewModel 是没有发生变化的。下面解释下这个原因：

我们在使用 `viewModels` 或者 `activityViewModels` 创建 ViewModel 时，ViewModel 本身会存储在一个 `ViewModelStore` 的类中
![](/4_12/pic1.png)
在触发销毁重建的逻辑后，会走到 `getLastNonConfigurateionInstance()` 的逻辑中，保证拿到的 `ViewModelStore` 和重建前的一致，从 ViewModelStore 获取到的 ViewModel 也就和原来保持一致了
![](/4_12/pic2.png)

# LiveData 如何在 Fragment/Activity 销毁后自动解绑

LiveData 在调用 `observe(lifecyclerOwner, Observer)` 方法时，会把 Observer 封装成一个 `LifecycleBoundObserver` 并和 lifecycleOwner 的状态进行绑定，在 lifecycleOwner 的状态为 destroy 时，会进行 `removeObserver`
![](/4_12/pic3.png)

而在将 LifecycleBoundObserver 绑定到 LifecycleRegistery 时，封装了 `ObserverWithState`
![](/4_12/pic4.png)

LifecycleRegistery 是 fragment/activity 持有的一个对象。当 fragment/activity 状态发生变化是，会触发 LifecycleRegsitery 的 `moveToState` 以及 `sync` 方法，从而开始状态的流转。在 `sync` 中，会对比当前的状态与 `observerMap` 中元素的状态，决定向前遍历 `forwardPass` 还是向后遍历 `backwardPass` ，无论哪种遍历方式，最终都会走到 `observer.dispatchEvent()` 方法中
![](/4_12/pic5.png)

在 `dispatchEvent` 中，可以看到是调用了 `mLifecycleObserver.onStateChanged(owner, event)` 方法，而 `mLifecycleObserver` 是在 `ObserverWithState` 构造方法里得到的。
![](/4_12/pic6.png)

我们看下 `lifecycleEventObserver` 的实现，由于我们传入的 observer(object) 本身是一个 `LifecycleBoundObserver` 即 `LifecycleEventObserver` ，所以会直接返回自己，而在调用 `onStateChange` 时，也会回调到
![](/4_12/pic7.png)

即如果 currentState 是 DESTORIED 状态，那么直接回 remove 掉 mObserver 解除绑定
![](/4_12/pic8.png)
