# view事件分发工作流程

![event_dispatch](.\img\event_dispatch.png)

- action_down事件是事件序列的开始，会清除之前事件序列的状态。
- 一个触控点的序列一般情况下只给一个view处理，当一个view消费了一个触控点的down事件后，该触控点的事件序列后续事件都会交给他处理。
- 当一个view消费了一个触控点的down事件后，viewgroup可以拦截子view的后续事件。
- 子view可以通过requestDisallowInterceptTouchEvent方法强制要求viewGroup不要拦截事件，viewGroup中会设置一个FLAG_DISALLOW_INTERCEPT标志表示不拦截事件。
- 当viewgroup决定拦截事件后，后续事件将默认交给它处理，并不再调用onInterceptTouchEvent方法。
- 某个view一旦开始处理事件，如果它不消耗action_down事件，那么后续事件就不会再交给它处理。
- 如果view不消耗出action_down 以外的其他事件，那么这个点击事件会消失，此时父元素的onTouchEvent不会被调用，最终这些消失的点击事件会交给Activity处理。
- 默认情况下，viewGroup是支持多点触控的分发，但view是不支持多点触控的，需要自己去重写 `dispatchTouchEvent` 方法来支持多点触控。