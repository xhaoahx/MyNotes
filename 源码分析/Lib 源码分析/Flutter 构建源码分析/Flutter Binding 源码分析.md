# Flutter Binding 源码分析



## BindingBase

```dart
/// 用于混入类的基类，提供单例服务（也就是“胶水”类）
///
/// 这个类保证只会在其生命周期类被构建一次
///
/// 用于编写应用程序的最顶层将有一个继承自 [BindingBase] 的具体类，并使用所有各种 [BindingBase] 混入(如
/// [ServicesBinding] )。例如，Flutter 中的 widget 库引入了一个名为 [WidgetsFlutterBinding] 的绑定。相关
/// 库定义了如何创建此绑定。它可能是隐含的(例如 [WidgetsFlutterBinding] 是从 [runApp] 自动启用的)，或者应用
/// 程序可能需要显式地调用其构造函数。
abstract class BindingBase {
    /// 首先调用 [initInstances] 来初始化其 binding 的实例指针和其他状态。
    /// 随后调用 [initServiceExtensions] 来使得 binding 初始化它们的观测者服务拓展（如果有的话）
    BindingBase() {
        developer.Timeline.startSync('Framework initialization');

        assert(!_debugInitialized);
        /// 在 binding 类的实现中，通常会重载此函数，并在 binding 内持有一个实例
        initInstances();
        assert(_debugInitialized);

        assert(!_debugServiceExtensionsRegistered);
        initServiceExtensions();
        assert(_debugServiceExtensionsRegistered);

        developer.postEvent('Flutter.FrameworkInitialization', <String, String>{});

        developer.Timeline.finishSync();
    }

    static bool _debugInitialized = false;
    static bool _debugServiceExtensionsRegistered = false;

    /// 此胶水类绑定的视窗
    ///
    /// 有大量的 bingding 类被定义成 [BindingBase] 的额外拓展，例如, [ServicesBinding], 
    /// [RendererBinding], and [WidgetsBinding]。它们中的每一个都定义了一些与 [ui.Window] 交互的行为。
    /// 例如：
    /// [ServicesBinding] 注册了 [ui.Window.onPlatformMessage] 句柄
    /// [RendererBinding] 注册了 [ui.Window.onMetricsChanged] ,[ui.Window.onTextScaleFactorChanged], 
    /// [ui.Window.onSemanticsEnabledChanged] 和 [ui.Window.onSemanticsAction] 句柄
    ///
    /// 这些 binding 都能够独立地且静态地与 window 交互，但是，这将限制使用一个用于验证目的的伪窗口来测试这些
    /// 行为的能力。因此，[BindingBase] 暴露此 [Window] 供其他绑定使用。[BindingBase] 的一个子类，比如
    /// [TestWidgetsFlutterBinding]，可以重载这个 getter 来返回一个不同的 [Window] 实现，比如
    /// [TestWindow]。
    ui.Window get window => ui.window;

    /// 初始化方法。子类可以重载这个方法来与平台相联系或者配置它们自身的服务。子类必须调用 
    /// "super.initInstances()"
    ///
    /// 通常来说，如果服务是作为一个单例提供的，它应该以 `MixinClassName.instance` 的方式向外暴露。其实现方
    /// 法是使用一个静态域保存 MixinClassName._instance，并在此初始化方法中保存自身的静态指针到静态域上
    /// 也就是说，单例模式通常以如下的方式实现：
    /// ```dart
    /// mixin MixinClassName{
    /// ...
    /// static MixinClassName _instance;
    /// ...
    /// @override
    /// void initInstances(){
    /// 	_instance = this;
    /// }
    /// ...
    /// }
    @protected
    @mustCallSuper
    void initInstances() {
        assert(!_debugInitialized);
        assert(() {
            _debugInitialized = true;
            return true;
        }());
    }

    /// 在 binding 被初始化的时候调用，用于注册服务扩展,
    ///
    /// Bindings that want to expose service extensions should overload
    /// this method to register them using calls to
    /// [registerSignalServiceExtension],
    /// [registerBoolServiceExtension],
    /// [registerNumericServiceExtension], and
    /// [registerServiceExtension] (in increasing order of complexity).
    @protected
    @mustCallSuper
    void initServiceExtensions() {
        assert(!_debugServiceExtensionsRegistered);

        assert(() {
            registerSignalServiceExtension(
                name: 'reassemble',
                callback: reassembleApplication,
            );
            return true;
        }());

        if (!kReleaseMode && !kIsWeb) {
            registerSignalServiceExtension(
                name: 'exit',
                callback: _exitApplication,
            );
            registerServiceExtension(
                name: 'saveCompilationTrace',
                callback: (Map<String, String> parameters) async {
                    return <String, dynamic>{
                        'value': ui.saveCompilationTrace(),
                    };
                },
            );
        }

        /// 以下断言了平台的正确性
        assert(() {
            const String platformOverrideExtensionName = 'platformOverride';
            registerServiceExtension(
                name: platformOverrideExtensionName,
                callback: (Map<String, String> parameters) async {
                    if (parameters.containsKey('value')) {
                        switch (parameters['value']) {
                            case 'android':
                                debugDefaultTargetPlatformOverride = TargetPlatform.android;
                                break;
                            case 'iOS':
                                debugDefaultTargetPlatformOverride = TargetPlatform.iOS;
                                break;
                            case 'macOS':
                                debugDefaultTargetPlatformOverride = TargetPlatform.macOS;
                                break;
                            case 'fuchsia':
                                debugDefaultTargetPlatformOverride = TargetPlatform.fuchsia;
                                break;
                            case 'default':
                            default:
                                debugDefaultTargetPlatformOverride = null;
                        }
                        _postExtensionStateChangedEvent(
                            platformOverrideExtensionName,
                            defaultTargetPlatform.toString().substring('$TargetPlatform.'.length),
                        );
                        await reassembleApplication();
                    }
                    return <String, dynamic>{
                        'value': defaultTargetPlatform
                            .toString()
                            .substring('$TargetPlatform.'.length),
                    };
                },
            );
            return true;
        }());
        assert(() {
            _debugServiceExtensionsRegistered = true;
            return true;
        }());
    }

    /// 是否 [lockEvents] 方法目前锁定了事件
    ///
    /// 触发事件的 binding 子类应该首先检查这一点，如果设置了绑定子类，则应该对事件进行排队，而不是立即触发它
    /// 们。
    ///
    /// 事件应该在调用 [unlock] 时刷新。
    @protected
    bool get locked => _lockCount > 0;
    int _lockCount = 0;

    /// 锁定异步事件的派发，直到给定的 Future 回调完成
    ///
    /// 这将导致输入延迟，因此应该尽可能避免这种情况。它主要用于非用户交互时间，比如允许
    /// [reassembleApplication] 在遍历树时阻塞输入(它部分是异步执行的)。
    @protected
    Future<void> lockEvents(Future<void> callback()) {
        developer.Timeline.startSync('Lock events');

        assert(callback != null);
        _lockCount += 1;
        final Future<void> future = callback();
        assert(future != null, 
               'The lockEvents() callback returned null; it should return a' 
               'Future<void> that completes when the lock is to expire.'
              );
        future.whenComplete(() {
            _lockCount -= 1;
            if (!locked) {
                developer.Timeline.finishSync();
                unlocked();
            }
        });
        return future;
    }

    /// 被 [lockEvents] 调用，用于解锁事件
    ///
    /// 此方法应该冲刷任何在 [locked] 期间被阻塞的事件
    @protected
    @mustCallSuper
    void unlocked() {
        assert(!locked);
    }

    /// 使得整个应用发生重建，例如，在热重载之后
    ///
    /// 这个方法是十分昂贵的，应该只在开发期间调用
    ///
    /// 这个方法运行的时候，事件是被锁定的 （例如，指针事件是不会被派发的）
    ///
    /// 子类 (binding classes) 应该重载 [performReassemble] 来响应此方法的调用，而不是重载这个方法
    Future<void> reassembleApplication() {
        return lockEvents(performReassemble);
    }

    /// 此方法被 [reassembleApplication] 来实际响应应用的组装过程（例如，在热重载之后）
    ///
    /// binding 子类应该通过此方法 重新注册 任何使用闭包的内容，这样它们就不会一直指向旧代码，并刷新以前计算的
    /// 任何缓存值，以防新代码以不同的方式计算它们。例如，当调用该层时，渲染层将触发整个应用程序重新绘制。
    @mustCallSuper
    @protected
    Future<void> performReassemble() {
        FlutterError.resetErrorCount();
        return Future<void>.value();
    }

    /// Registers a service extension method with the given name (full
    /// name "ext.flutter.name"), which takes no arguments and returns
    /// no value.
    ///
    /// Calls the `callback` callback when the service extension is called.
    ///
    /// {@macro flutter.foundation.bindingBase.registerServiceExtension}
    @protected
    void registerSignalServiceExtension({
        @required String name,
        @required AsyncCallback callback,
    }) {
        assert(name != null);
        assert(callback != null);
        registerServiceExtension(
            name: name,
            callback: (Map<String, String> parameters) async {
                await callback();
                return <String, dynamic>{};
            },
        );
    }

    /// Registers a service extension method with the given name (full
    /// name "ext.flutter.name"), which takes a single argument
    /// "enabled" which can have the value "true" or the value "false"
    /// or can be omitted to read the current value. (Any value other
    /// than "true" is considered equivalent to "false". Other arguments
    /// are ignored.)
    ///
    /// Calls the `getter` callback to obtain the value when
    /// responding to the service extension method being called.
    ///
    /// Calls the `setter` callback with the new value when the
    /// service extension method is called with a new value.
    ///
    /// {@macro flutter.foundation.bindingBase.registerServiceExtension}
    @protected
    void registerBoolServiceExtension({
        @required String name,
        @required AsyncValueGetter<bool> getter,
        @required AsyncValueSetter<bool> setter,
    }) {
        assert(name != null);
        assert(getter != null);
        assert(setter != null);
        registerServiceExtension(
            name: name,
            callback: (Map<String, String> parameters) async {
                if (parameters.containsKey('enabled')) {
                    await setter(parameters['enabled'] == 'true');
                    _postExtensionStateChangedEvent(name, await getter() ? 'true' : 'false');
                }
                return <String, dynamic>{'enabled': await getter() ? 'true' : 'false'};
            },
        );
    }

    /// Registers a service extension method with the given name (full
    /// name "ext.flutter.name"), which takes a single argument with the
    /// same name as the method which, if present, must have a value
    /// that can be parsed by [double.parse], and can be omitted to read
    /// the current value. (Other arguments are ignored.)
    ///
    /// Calls the `getter` callback to obtain the value when
    /// responding to the service extension method being called.
    ///
    /// Calls the `setter` callback with the new value when the
    /// service extension method is called with a new value.
    ///
    /// {@macro flutter.foundation.bindingBase.registerServiceExtension}
    @protected
    void registerNumericServiceExtension({
        @required String name,
        @required AsyncValueGetter<double> getter,
        @required AsyncValueSetter<double> setter,
    }) {
        assert(name != null);
        assert(getter != null);
        assert(setter != null);
        registerServiceExtension(
            name: name,
            callback: (Map<String, String> parameters) async {
                if (parameters.containsKey(name)) {
                    await setter(double.parse(parameters[name]));
                    _postExtensionStateChangedEvent(name, (await getter()).toString());
                }
                return <String, dynamic>{name: (await getter()).toString()};
            },
        );
    }

    /// Sends an event when a service extension's state is changed.
    ///
    /// Clients should listen for this event to stay aware of the current service
    /// extension state. Any service extension that manages a state should call
    /// this method on state change.
    ///
    /// `value` reflects the newly updated service extension value.
    ///
    /// This will be called automatically for service extensions registered via
    /// [registerBoolServiceExtension], [registerNumericServiceExtension], or
    /// [registerStringServiceExtension].
    void _postExtensionStateChangedEvent(String name, dynamic value) {
        postEvent(
            'Flutter.ServiceExtensionStateChanged',
            <String, dynamic>{
                'extension': 'ext.flutter.$name',
                'value': value,
            },
        );
    }

    /// All events dispatched by a [BindingBase] use this method instead of
    /// calling [developer.postEvent] directly so that tests for [BindingBase]
    /// can track which events were dispatched by overriding this method.
    @protected
    void postEvent(String eventKind, Map<String, dynamic> eventData) {
        developer.postEvent(eventKind, eventData);
    }

    /// Registers a service extension method with the given name (full name
    /// "ext.flutter.name"), which optionally takes a single argument with the
    /// name "value". If the argument is omitted, the value is to be read,
    /// otherwise it is to be set. Returns the current value.
    ///
    /// Calls the `getter` callback to obtain the value when
    /// responding to the service extension method being called.
    ///
    /// Calls the `setter` callback with the new value when the
    /// service extension method is called with a new value.
    ///
    /// {@macro flutter.foundation.bindingBase.registerServiceExtension}
    @protected
    void registerStringServiceExtension({
        @required String name,
        @required AsyncValueGetter<String> getter,
        @required AsyncValueSetter<String> setter,
    }) {
        assert(name != null);
        assert(getter != null);
        assert(setter != null);
        registerServiceExtension(
            name: name,
            callback: (Map<String, String> parameters) async {
                if (parameters.containsKey('value')) {
                    await setter(parameters['value']);
                    _postExtensionStateChangedEvent(name, await getter());
                }
                return <String, dynamic>{'value': await getter()};
            },
        );
    }

    /// Registers a service extension method with the given name (full name
    /// "ext.flutter.name").
    ///
    /// The given callback is called when the extension method is called. The
    /// callback must return a [Future] that either eventually completes to a
    /// return value in the form of a name/value map where the values can all be
    /// converted to JSON using `json.encode()` (see [JsonEncoder]), or fails. In
    /// case of failure, the failure is reported to the remote caller and is
    /// dumped to the logs.
    ///
    /// The returned map will be mutated.
    ///
    /// {@template flutter.foundation.bindingBase.registerServiceExtension}
    /// A registered service extension can only be activated if the vm-service
    /// is included in the build, which only happens in debug and profile mode.
    /// Although a service extension cannot be used in release mode its code may
    /// still be included in the Dart snapshot and blow up binary size if it is
    /// not wrapped in a guard that allows the tree shaker to remove it (see
    /// sample code below).
    ///
    /// {@tool sample}
    /// The following code registers a service extension that is only included in
    /// debug builds.
    ///
    /// ```dart
    /// void myRegistrationFunction() {
    ///   assert(() {
    ///     // Register your service extension here.
    ///     return true;
    ///   }());
    /// }
    /// ```
    /// {@end-tool}
    ///
    /// {@tool sample}
    /// A service extension registered with the following code snippet is
    /// available in debug and profile mode.
    ///
    /// ```dart
    /// void myRegistrationFunction() {
    ///   // kReleaseMode is defined in the 'flutter/foundation.dart' package.
    ///   if (!kReleaseMode) {
    ///     // Register your service extension here.
    ///   }
    /// }
    /// ```
    /// {@end-tool}
    ///
    /// Both guards ensure that Dart's tree shaker can remove the code for the
    /// service extension in release builds.
    /// {@endtemplate}
    @protected
    void registerServiceExtension({
        @required String name,
        @required ServiceExtensionCallback callback,
    }) {
        assert(name != null);
        assert(callback != null);
        final String methodName = 'ext.flutter.$name';
        developer.registerExtension(methodName, (String method, Map<String, String> parameters) async {
            assert(method == methodName);
            assert(() {
                if (debugInstrumentationEnabled)
                    debugPrint('service extension method received: $method($parameters)');
                return true;
            }());

            // VM service extensions are handled as "out of band" messages by the VM,
            // which means they are handled at various times, generally ASAP.
            // Notably, this includes being handled in the middle of microtask loops.
            // While this makes sense for some service extensions (e.g. "dump current
            // stack trace", which explicitly doesn't want to wait for a loop to
            // complete), Flutter extensions need not be handled with such high
            // priority. Further, handling them with such high priority exposes us to
            // the possibility that they're handled in the middle of a frame, which
            // breaks many assertions. As such, we ensure they we run the callbacks
            // on the outer event loop here.
            await debugInstrumentAction<void>('Wait for outer event loop', () {
                return Future<void>.delayed(Duration.zero);
            });

            dynamic caughtException;
            StackTrace caughtStack;
            Map<String, dynamic> result;
            try {
                result = await callback(parameters);
            } catch (exception, stack) {
                caughtException = exception;
                caughtStack = stack;
            }
            if (caughtException == null) {
                result['type'] = '_extensionType';
                result['method'] = method;
                return developer.ServiceExtensionResponse.result(json.encode(result));
            } else {
                FlutterError.reportError(FlutterErrorDetails(
                    exception: caughtException,
                    stack: caughtStack,
                    context: ErrorDescription('during a service extension callback for "$method"'),
                ));
                return developer.ServiceExtensionResponse.error(
                    developer.ServiceExtensionResponse.extensionError,
                    json.encode(<String, String>{
                        'exception': caughtException.toString(),
                        'stack': caughtStack.toString(),
                        'method': method,
                    }),
                );
            }
        });
    }

    @override
    String toString() => '<$runtimeType>';
}

```



## GestureBinding

```dart

/// 手势系统的 binding
///
/// ## 指针的生命周期和手势竞技场
///
/// ### [PointerDownEvent] （指针落下事件）
///
/// 当指针落下事件 [PointerDownEvent] 被 [GestureBinding] 接受(来自于 [Window.onPointerDataPacket], 被
/// [PointerEventConverter] 所拦截)到的时候，[hitTest] 将会进行，并且决定哪一些 [HitTestTarget] 节点会受到
/// 影响(其他的 binding 应该实现 [hitTest] 以延迟给 [HitTestable] 对象。
///
/// 随后，受到影响的节点被要求处理事件([dispatchEvent] 调用每一个受影响的节点的  
/// [HitTestTarget.handleEvent])，如果有任何相关的 [GestureRecognizer]s, 它们调用  
/// [GestureRecognizer.addPointer]. 这通常会导致识别器向 [PointerRouter] 注册，以接收有关指针的通知。
///
/// 一旦点击测试和调度逻辑完成，事件就会被传递到前面提到的[PointerRouter]，它会将事件传递给对该事件感兴趣的任
/// 何对象。
///
/// 最终，[gestureArena]对给定的指针([GestureArenaManager.close])关闭，它开始选择一个手势来赢得该指针
///
/// ### 其他事件
///
/// [PointerEvent.down] 可能发出更多的事件，例如 [PointerMoveEvent], [PointerUpEvent] 或者
/// [PointerCancelEvent]。它们被发送到与接收到 down 事件时相同的 [HitTestTarget] 节点(即使它们已经被释放;这
/// 些物体有责任意识到这种可能性)。
///
/// 然后，事件被路由到[PointerRouter]的表中任何仍然注册的参与者，以获取该指针。
///
/// 当接收到 [PointerUpEvent] 时， [GestureArenaManager.sweep] 方法强制必要时终止手势域逻辑。
/// 
/// 注意，此混入类实现了 HitTestable, HitTestDispatcher, HitTestTarget
mixin GestureBinding on BindingBase implements HitTestable, HitTestDispatcher, HitTestTarget {
    @override
    void initInstances() {
        super.initInstances();
        _instance = this;
        window.onPointerDataPacket = _handlePointerDataPacket;
    }

    @override
    void unlocked() {
        super.unlocked();
        _flushPointerEventQueue();
    }

    /// 单例保存
    static GestureBinding get instance => _instance;
    static GestureBinding _instance;

    final Queue<PointerEvent> _pendingPointerEvents = Queue<PointerEvent>();

    /// 处理指针事件包
    /// 当用户触发（如触发、滑动屏幕）指针事件的时候，平台会向 Flutter 端发送一个打包好的数据包，其中包括了指
    /// 针的各种信息（例如，偏移，按压力度等等），此时，window 会接收此数据包，并将其作为参数调用此回调
    void _handlePointerDataPacket(ui.PointerDataPacket packet) {
        // 我们将指针数据转换成逻辑像素，这样就可以以一种与设备无关的方式定义触摸
        /// 注意这里，将转换完成的指针事件添加到了 _pendingPointerEvents 队列中，等待 unlock（如果当前处于 
        /// lock 状态下的话）之后冲刷队列，否则理解冲刷队列
        _pendingPointerEvents.addAll(
            PointerEventConverter.expand(packet.data, window.devicePixelRatio)
        );
        if (!locked)
            _flushPointerEventQueue();
    }

    /// 派发对于给定指针的 [PointerCancelEvent] 
    ///
    /// 指针事件将在下一个指针事件之前和微任务结束之前分派，但不在此函数调用中分派。
    void cancelPointer(int pointer) {
        if (_pendingPointerEvents.isEmpty && !locked)
            scheduleMicrotask(_flushPointerEventQueue);
        _pendingPointerEvents.addFirst(PointerCancelEvent(pointer: pointer));
    }

    /// 冲刷指针事件队列。即对指针事件队列顺序出列，并对出列的每一个事件调用 _handlePointerEvent
    /// 在处于 lock 状态时，会暂停冲刷事件队列，直到 unlcok 被调用
    void _flushPointerEventQueue() {
        assert(!locked);
        while (_pendingPointerEvents.isNotEmpty)
            _handlePointerEvent(_pendingPointerEvents.removeFirst());
    }

    /// 引导从引擎接收到的所有指针事件的路由器
    final PointerRouter pointerRouter = PointerRouter();

    /// 用于消除一系列指针事件歧义的手势竞技场
    final GestureArenaManager gestureArena = GestureArenaManager();

    /// 用于确定哪个小部件处理指针信号事件的解析器？？
    final PointerSignalResolver pointerSignalResolver = PointerSignalResolver();

    /// 当前落下的所有指针的状态，是一个从指针 id 到 hitTestResult 的映射
    ///
    /// 不跟踪悬停指针的状态，因为这需要对每一帧进行命中测试。
    final Map<int, HitTestResult> _hitTests = <int, HitTestResult>{};

    /// 处理指针事件，只能在 unlock 的状态下被调用 
    void _handlePointerEvent(PointerEvent event) {
        assert(!locked);
        HitTestResult hitTestResult;
        /// 如果是指针落下事件或者指针信号事件
        /// 注意，对于每一个落下的指针，引擎都会为其分配一个指针 id
        if (event is PointerDownEvent || event is PointerSignalEvent) {
            /// 显然，_hitTests 不能包含已经落下的指针 id
            assert(!_hitTests.containsKey(event.pointer));
            /// 新建一个 HitTestResult 并将其作为 hitTest 的入参。RenderingBinding 对 hitTest 的重载会引
            /// 导整棵 渲染对象树 进行点击测试
            hitTestResult = HitTestResult();
            hitTest(hitTestResult, event.position);
            /// 得到一个 hitTestResult 结果后，将其放入 _hitTests 中
            if (event is PointerDownEvent) {
                _hitTests[event.pointer] = hitTestResult;
            }
        /// 如果是指针抬起或者指针取消事件，则需要从 _hitTests 中移除对应指针 id 和其 hitTestResult
        } else if (event is PointerUpEvent || event is PointerCancelEvent) {
            hitTestResult = _hitTests.remove(event.pointer);
        /// 发出此事件的时候指针已经落下（例如指针移动事件）
        } else if (event.down) {
            // 发出此事件的时候指针已经落下（例如指针移动事件），则此事件必须派发到它们初始指针落下的位置（即
            // 初始 PointerDownEvent 的结果）。因此，我们需要重用我们已经找到的路径，而不是重新做一次点击测
            // 试 
            hitTestResult = _hitTests[event.pointer];
        }
        if (/// 如果得到了点击测试的结果
            hitTestResult != null ||
            /// PointerHoverEvent 是指针（通常是鼠标）悬浮在设备上（即没有与设备接触）的情况下派发的事件
            event is PointerHoverEvent ||
            /// PointerHoverEvent 是指针（通常是鼠标）进入设备范围且悬浮在设备上
            event is PointerAddedEvent ||
            /// PointerHoverEvent 是指针（通常是鼠标）从悬浮状态并离开设备范围
            event is PointerRemovedEvent) {
            /// 在以上情况下都需要派发事件
            dispatchEvent(event, hitTestResult);
        }
    }

    /// 确定在给定的 positon 下，那些 [HitTestTarget] 受到了影响
    @override // from HitTestable
    void hitTest(HitTestResult result, Offset position) {
        /// 把自身添加到结果中是为了调用自身的 handleEvent 
        result.add(HitTestEntry(this));
    }

    /// 沿着点击测试结果的路径派发事件
    ///
    /// 这将给定的事件发送到给定的 [HitTestResult] 整体中的每个 [HitTestTarget]，并捕获任何句柄抛出的异常。
    /// 注意：[hitTestResult] 参数只能在 [PointerHoverEvent]、[PointerAddedEvent] 或
    /// [PointerRemovedEvent] 事件中为空。
    @override // from HitTestDispatcher
    void dispatchEvent(PointerEvent event, HitTestResult hitTestResult) {
        assert(!locked);
        // 没有命中测试信息意味着这是一个悬停或指针添加/删除事件。
        if (hitTestResult == null) {
            try {
                pointerRouter.route(event);
            } catch (exception, stack) {
               ...
            }
            return;
        }
        // 否则，对于 hitTestResult.path 中的每一个 HitTestEntry 调用 handleEvent 来处理转换过（即把全局坐
        // 标转换成本地坐标）的指针事件
        for (HitTestEntry entry in hitTestResult.path) {
            try {
                entry.target.handleEvent(event.transformed(entry.transform), entry);
            } catch (exception, stack) {
                ...
            }
        }
    }

    /// 由于 hitTest 的递归性，此 handleEvent 会在最后被调用
    @override 
    void handleEvent(PointerEvent event, HitTestEntry entry) {
        pointerRouter.route(event);
        /// 如果是指针落下事件，竞技场对于给定的指针 id 关闭，并尝试决出一个获胜者
        if (event is PointerDownEvent) {
            gestureArena.close(event.pointer);
        /// 如果是指针抬起事件，则会强制清扫竞技场，并决出一个获胜者
        } else if (event is PointerUpEvent) {
            gestureArena.sweep(event.pointer);
        } else if (event is PointerSignalEvent) {
            pointerSignalResolver.resolve(event);
        }
    }
}
```



## ServiceBinding

```dart
/// 监听平台的消息并且将它们导向 [defaultBinaryMessenger].
///
/// [ServicesBinding] 也注册了一个 [LicenseEntryCollector]，它提供了 "License" 文件
mixin ServicesBinding on BindingBase {
    @override
    void initInstances() {
        super.initInstances();
        _instance = this;
        _defaultBinaryMessenger = createBinaryMessenger();
        window
            ..onPlatformMessage = defaultBinaryMessenger.handlePlatformMessage;
        initLicenses();
        SystemChannels.system.setMessageHandler(handleSystemMessage);
    }

    static ServicesBinding get instance => _instance;
    static ServicesBinding _instance;

    /// 默认的 [BinaryMessenger] 实例
    ///
    /// 此实例被用于从 Flutter 应用向平台发送消息, 并跟踪每个通道上已注册的处理程序，以便将传入消息分派给注册
    /// 的处理程序。
    BinaryMessenger get defaultBinaryMessenger => _defaultBinaryMessenger;
    BinaryMessenger _defaultBinaryMessenger;

    /// 创建默认的 [BinaryMessenger] 
    @protected
    BinaryMessenger createBinaryMessenger() {
        return const _DefaultBinaryMessenger._();
    }

    /// 处理从 [SystemChannels.system] 接收到的平台消息
    ///
    /// 其他 bindings 可以重载此方法来响应平台消息
    @protected
    @mustCallSuper
    Future<void> handleSystemMessage(Object systemMessage) async { }
	
    /// 略 license 相关
    ...

    @override
    void initServiceExtensions() {
        super.initServiceExtensions();

        assert(() {
            registerStringServiceExtension(
                // ext.flutter.evict value=foo.png will cause foo.png to be evicted from
                // the rootBundle cache and cause the entire image cache to be cleared.
                // This is used by hot reload mode to clear out the cache of resources
                // that have changed.
                name: 'evict',
                getter: () async => '',
                setter: (String value) async {
                    evict(value);
                },
            );
            return true;
        }());
    }

    /// Called in response to the `ext.flutter.evict` service extension.
    ///
    /// This is used by the `flutter` tool during hot reload so that any images
    /// that have changed on disk get cleared from caches.
    @protected
    @mustCallSuper
    void evict(String asset) {
        rootBundle.evict(asset);
    }
}
```



## SchedulerBinding

```dart
/// 调度器 binding，调度如下操作:
///
/// * _Transient callbacks_, 被 [Window.onBeginFrame] 所触发，同步
///   callback, 将应用程序的行为与系统的显示同步。例如，[Ticker] 和 [AnimationController] Transient 
///   callbacks 触发
///
/// * _Persistent callbacks_, 被 [Window.onDrawFrame] 所触发，用于在 transient callbacks 之后更新系统
///   的显示。例如，渲染成利用此回调来驱动渲染流水线
///
/// * _Post-frame callbacks_, 在 Persistent callbacks 完成后，在 [Window.onDrawFrame] 返回之前触发
///   也就是所谓的帧尾回调
///
/// * 非渲染任务，在帧之间运行。它们被赋予优先级，并根据 [schedulingStrategy] 按优先级顺序执行。
mixin SchedulerBinding on BindingBase, ServicesBinding {
    @override
    void initInstances() {
        super.initInstances();
        _instance = this;
        SystemChannels.lifecycle.setMessageHandler(_handleLifecycleMessage);
        readInitialLifecycleStateFromNativeWindow();

        if (!kReleaseMode) {
            int frameNumber = 0;
            addTimingsCallback((List<FrameTiming> timings) {
                for (FrameTiming frameTiming in timings) {
                    frameNumber += 1;
                    _profileFramePostEvent(frameNumber, frameTiming);
                }
            });
        }
    }

    final List<TimingsCallback> _timingsCallbacks = <TimingsCallback>[];

    /// 添加一个可来接收来自引擎的 [FrameTiming] 的 [TimingsCallback]
    ///
    /// 例如，它可以用于监视发布模式下的性能，或者在第一帧光栅化时获取信号。
    ///
    /// 这比使用 [Window.onReportTimings] 更好，因为 [addTimingsCallback] 允许多个回调。
    ///
    /// 如果同一个回调被添加两次，那么它将被执行两次。
    void addTimingsCallback(TimingsCallback callback) {
        _timingsCallbacks.add(callback);
        /// 当 _timingsCallbacks 中至少有一个回调的时候，将 _executeTimingsCallbacks 作为
        /// window.onReportTimings
        if (_timingsCallbacks.length == 1) {
            assert(window.onReportTimings == null);
            window.onReportTimings = _executeTimingsCallbacks;
        }
        assert(window.onReportTimings == _executeTimingsCallbacks);
    }

    /// 移除 _timingsCallbacks 中的一个回调
    void removeTimingsCallback(TimingsCallback callback) {
        assert(_timingsCallbacks.contains(callback));
        _timingsCallbacks.remove(callback);
        if (_timingsCallbacks.isEmpty) {
            window.onReportTimings = null;
        }
    }

    /// 这个方法通常会被作为 window.onReportFrame 回调（当 timingsCallbacks 不为空的时候）
    /// 关于 window.onReportFrame ：
    /// 它是一个被调用来报告最近栅格化的帧的 [FrameTiming] 的回调。
    /// 这可以用来查看应用程序是否丢失了帧(通过 [FrameTiming] )，或较高的延迟(通过 
    /// [FrameTiming.totalSpan])。
    ///
    /// 与 [Timeline] 不同，这里的计时信息在发布模式中可用(除了概要文件和调试模式之外)。因此，这可以用于监控
    /// 应用程序的性能。
    void _executeTimingsCallbacks(List<FrameTiming> timings) {
        /// 这里首先对 _timingsCallbacks 进行了一次（浅）拷贝
        final List<TimingsCallback> clonedCallbacks =
            List<TimingsCallback>.from(_timingsCallbacks);
        /// 遍历 clonedCallbacks 中的每一个回调，如果此时回调还处于 _timingsCallbacks 中的话，则触发此回调
        /// 这是为了防止异步对 _timingsCallbacks 带来的影响
        for (TimingsCallback callback in clonedCallbacks) {
            try {
                if (_timingsCallbacks.contains(callback)) {
                    callback(timings);
                }
            } catch (exception, stack) {
                ...
            }
        }
    }

    static SchedulerBinding get instance => _instance;
    static SchedulerBinding _instance;

    @override
    void initServiceExtensions() {
        super.initServiceExtensions();

        if (!kReleaseMode) {
            registerNumericServiceExtension(
                name: 'timeDilation',
                getter: () async => timeDilation,
                setter: (double value) async {
                    timeDilation = value;
                },
            );
        }
    }

    /// 当前应用的生命周期状态
    ///
    /// 被 [handleAppLifecycleStateChanged] 当 [SystemChannels.lifecycle] 通知被派发的时候
    ///
    /// 推荐使用 [WidgetsBindingObserver.didChangeAppLifecycleState] 来观测此字段的变化
    AppLifecycleState get lifecycleState => _lifecycleState;
    AppLifecycleState _lifecycleState;

    /// Initializes the [lifecycleState] with the [initialLifecycleState] from the
    /// window.
    ///
    /// Once the [lifecycleState] is populated through any means (including this
    /// method), this method will do nothing. This is because the
    /// [initialLifecycleState] may already be stale and it no longer makes sense
    /// to use the initial state at dart vm startup as the current state anymore.
    ///
    /// The latest state should be obtained by subscribing to
    /// [WidgetsBindingObserver.didChangeAppLifecycleState].
    @protected
    void readInitialLifecycleStateFromNativeWindow() {
        if (   _lifecycleState == null
            && _parseAppLifecycleMessage(window.initialLifecycleState) != null
        ) {
            _handleLifecycleMessage(window.initialLifecycleState);
        }
    }

    /// Called when the application lifecycle state changes.
    ///
    /// Notifies all the observers using
    /// [WidgetsBindingObserver.didChangeAppLifecycleState].
    ///
    /// This method exposes notifications from [SystemChannels.lifecycle].
    @protected
    @mustCallSuper
    void handleAppLifecycleStateChanged(AppLifecycleState state) {
        assert(state != null);
        _lifecycleState = state;
        switch (state) {
            case AppLifecycleState.resumed:
            case AppLifecycleState.inactive:
                _setFramesEnabledState(true);
                break;
            case AppLifecycleState.paused:
                _setFramesEnabledState(false);
                break;
            default:
                break;
        }
    }

    Future<String> _handleLifecycleMessage(String message) async {
        // TODO(chunhtai): remove the workaround once the issue is fixed
        // https://github.com/flutter/flutter/issues/39832
        if (message == 'AppLifecycleState.detached')
            return null;

        handleAppLifecycleStateChanged(_parseAppLifecycleMessage(message));
        return null;
    }

    /// 将平台提供的应用生命周期状态转换为 dart 枚举
    static AppLifecycleState _parseAppLifecycleMessage(String message) {
        switch (message) {
            case 'AppLifecycleState.paused':
                return AppLifecycleState.paused;
            case 'AppLifecycleState.resumed':
                return AppLifecycleState.resumed;
            case 'AppLifecycleState.inactive':
                return AppLifecycleState.inactive;
        }
        return null;
    }

    /// The strategy to use when deciding whether to run a task or not.
    ///
    /// Defaults to [defaultSchedulingStrategy].
    SchedulingStrategy schedulingStrategy = defaultSchedulingStrategy;

    static int _taskSorter (_TaskEntry<dynamic> e1, _TaskEntry<dynamic> e2) {
        return -e1.priority.compareTo(e2.priority);
    }
    final PriorityQueue<_TaskEntry<dynamic>> _taskQueue = HeapPriorityQueue<_TaskEntry<dynamic>>(_taskSorter);

    /// Schedules the given `task` with the given `priority` and returns a
    /// [Future] that completes to the `task`'s eventual return value.
    ///
    /// The `debugLabel` and `flow` are used to report the task to the [Timeline],
    /// for use when profiling.
    ///
    /// ## Processing model
    ///
    /// Tasks will be executed between frames, in priority order,
    /// excluding tasks that are skipped by the current
    /// [schedulingStrategy]. Tasks should be short (as in, up to a
    /// millisecond), so as to not cause the regular frame callbacks to
    /// get delayed.
    ///
    /// If an animation is running, including, for instance, a [ProgressIndicator]
    /// indicating that there are pending tasks, then tasks with a priority below
    /// [Priority.animation] won't run (at least, not with the
    /// [defaultSchedulingStrategy]; this can be configured using
    /// [schedulingStrategy]).
    Future<T> scheduleTask<T>(
        TaskCallback<T> task,
        Priority priority, {
            String debugLabel,
            Flow flow,
        }) {
        final bool isFirstTask = _taskQueue.isEmpty;
        final _TaskEntry<T> entry = _TaskEntry<T>(
            task,
            priority.value,
            debugLabel,
            flow,
        );
        _taskQueue.add(entry);
        if (isFirstTask && !locked)
            _ensureEventLoopCallback();
        return entry.completer.future;
    }

    @override
    void unlocked() {
        super.unlocked();
        if (_taskQueue.isNotEmpty)
            _ensureEventLoopCallback();
    }

    // Whether this scheduler already requested to be called from the event loop.
    bool _hasRequestedAnEventLoopCallback = false;

    // Ensures that the scheduler services a task scheduled by
    // [SchedulerBinding.scheduleTask].
    void _ensureEventLoopCallback() {
        assert(!locked);
        assert(_taskQueue.isNotEmpty);
        if (_hasRequestedAnEventLoopCallback)
            return;
        _hasRequestedAnEventLoopCallback = true;
        Timer.run(_runTasks);
    }

    // Scheduled by _ensureEventLoopCallback.
    void _runTasks() {
        _hasRequestedAnEventLoopCallback = false;
        if (handleEventLoopCallback())
            _ensureEventLoopCallback(); // runs next task when there's time
    }

    /// Execute the highest-priority task, if it is of a high enough priority.
    ///
    /// Returns true if a task was executed and there are other tasks remaining
    /// (even if they are not high-enough priority).
    ///
    /// Returns false if no task was executed, which can occur if there are no
    /// tasks scheduled, if the scheduler is [locked], or if the highest-priority
    /// task is of too low a priority given the current [schedulingStrategy].
    ///
    /// Also returns false if there are no tasks remaining.
    @visibleForTesting
    bool handleEventLoopCallback() {
        if (_taskQueue.isEmpty || locked)
            return false;
        final _TaskEntry<dynamic> entry = _taskQueue.first;
        if (schedulingStrategy(priority: entry.priority, scheduler: this)) {
            try {
                _taskQueue.removeFirst();
                entry.run();
            } catch (exception, exceptionStack) {
                StackTrace callbackStack;
                assert(() {
                    callbackStack = entry.debugStack;
                    return true;
                }());
                FlutterError.reportError(FlutterErrorDetails(
                    exception: exception,
                    stack: exceptionStack,
                    library: 'scheduler library',
                    context: ErrorDescription('during a task callback'),
                    informationCollector: (callbackStack == null) ? null : () sync* {
                        yield DiagnosticsStackTrace(
                            '\nThis exception was thrown in the context of a scheduler callback. '
                            'When the scheduler callback was _registered_ (as opposed to when the '
                            'exception was thrown), this was the stack',
                            callbackStack,
                        );
                    },
                ));
            }
            return _taskQueue.isNotEmpty;
        }
        return false;
    }

    int _nextFrameCallbackId = 0; // positive
    
    /// 瞬时帧回调列表（其实是从一个 int 到 _FrameCallbackEntry 的映射）
    ///
    Map<int, _FrameCallbackEntry> _transientCallbacks = <int, _FrameCallbackEntry>{};
    /// 已经移除的帧回调 id
    final Set<int> _removedIds = HashSet<int>();

    /// 当前已经调度的瞬时帧回调的数量
    ///
    /// 此字段在所有的瞬时帧回调被调用之后重新设置成 0，也就是在每一帧开始（即持久帧回调阶段）的时候
    ///
    /// This number is primarily exposed so that tests can verify that
    /// there are no unexpected transient callbacks still registered
    /// after a test's resources have been gracefully disposed.
    int get transientCallbackCount => _transientCallbacks.length;

    /// 调度给定的瞬时帧回调
    ///
    /// 把给定的回调添加到会回调列表当中，并且确保调度一帧
    ///
    /// 如果这是一次性注册，请忽略“重新调度”参数。
    ///
    /// 如果这是一个回调，它将在每次触发时重新注册，那么当你重新注册回调时，将“重新调度”参数设置为true。
    /// 这在发行版本中没有影响，但是在调试构建中，它确保为这个回调存储的堆栈跟踪是注册回调时的原始堆栈跟踪，而
    /// 不是重新注册回调时的堆栈跟踪。这使得跟踪特定回调被调用的原始原因变得更加容易。如果“重调度”为真，则调用
    /// 必须在帧回调的上下文中进行。
    ///
    /// 使用 [cancelFrameCallbackWithId] 取消已经注册的回调
    int scheduleFrameCallback(FrameCallback callback, { bool rescheduling = false }) {
        scheduleFrame();
        _nextFrameCallbackId += 1;
        _transientCallbacks[_nextFrameCallbackId] = 
            _FrameCallbackEntry(callback, rescheduling: rescheduling);
        return _nextFrameCallbackId;
    }

    /// 取消给定的 [id] 对应的瞬时帧回调
    ///
    /// 这个方法会移除已经注册的回调。如果因为添加回调而调度了一帧，那么那一帧不会被取消
    void cancelFrameCallbackWithId(int id) {
        assert(id > 0);
        _transientCallbacks.remove(id);
        _removedIds.add(id);
    }

    /// 永久帧回调列表
    final List<FrameCallback> _persistentCallbacks = <FrameCallback>[];

    /// 添加一个持久帧弗蒂奥
    ///
    /// *绝对*不要请求新的一帧。概念上来说，持久帧回调是 "begin frame" 事件的观测者。因为它们在瞬时帧回调
    /// 之后被触发，因此，它们能够驱动渲染流水线
    ///
    /// 持久帧回调一旦注册就不能被取消，它们会在应用的整个生命周期中的每一帧被回调
    void addPersistentFrameCallback(FrameCallback callback) {
        _persistentCallbacks.add(callback);
    }
    
    /// 帧尾回调列表
    final List<FrameCallback> _postFrameCallbacks = <FrameCallback>[];

    /// 添加一个在帧尾的回调
    ///
    /// *绝对*不要请求新的一帧。
    ///
    /// 此回调在永久帧回调之后，且在一帧结束之前，被触发。如果，在一帧进行的时候向帧尾回调列表中添加一个回调，
    /// 它会被延迟到下一帧执行
    ///
    /// 回调一定会按照被添加的顺序执行
    ///
    /// 帧尾回调不可以被取消，它们也只会被调用一次
    void addPostFrameCallback(FrameCallback callback) {
        _postFrameCallbacks.add(callback);
    }

    /// 
    Completer<void> _nextFrameCompleter;

    /// 返回一个在一帧结束之后立刻完成的 Future
    ///
    /// 如果这个 getter 在两帧之间调用，那么它会立刻向引擎调度一帧；否则，返回一个 Futrue
    ///
    /// 如果设备的屏幕被关闭的话，那么，此 Future 会等待一个较长的时间，直到屏幕恢复使用
    Future<void> get endOfFrame {
        if (_nextFrameCompleter == null) {
            if (schedulerPhase == SchedulerPhase.idle)
                scheduleFrame();
            _nextFrameCompleter = Completer<void>();
            addPostFrameCallback((Duration timeStamp) {
                _nextFrameCompleter.complete();
                _nextFrameCompleter = null;
            });
        }
        return _nextFrameCompleter.future;
    }

    /// 时候已经调度了一帧，也就是意味着 [handleBeginFrame] 马上就要被调用
    bool get hasScheduledFrame => _hasScheduledFrame;
    bool _hasScheduledFrame = false;

    /// 当前的调度阶段
    /// 见 [SchedulerPhase]
    SchedulerPhase get schedulerPhase => _schedulerPhase;
    SchedulerPhase _schedulerPhase = SchedulerPhase.idle;

    /// 当 [scheduleFrame] 被调用的时候，是否可以调度一帧
    ///
    /// 此字段依赖于 [lifecycleState].
    bool get framesEnabled => _framesEnabled;

    bool _framesEnabled = true;
    void _setFramesEnabledState(bool enabled) {
        if (_framesEnabled == enabled)
            return;
        _framesEnabled = enabled;
        if (enabled)
            scheduleFrame();
    }

    /// 确保 window 绑定了回调
    /// 这个方法在每一帧都会被调用，确保 window 单例绑定了正确的的方法
    @protected
    void ensureFrameCallbacksRegistered() {
        window.onBeginFrame ??= _handleBeginFrame;
        window.onDrawFrame ??= _handleDrawFrame;
    }

    /// 如果当前没有在进行一帧的话，就调用 [scheduleFrame] 调度一帧
    void ensureVisualUpdate() {
        switch (schedulerPhase) {
            case SchedulerPhase.idle:
            case SchedulerPhase.postFrameCallbacks:
                scheduleFrame();
                return;
            case SchedulerPhase.transientCallbacks:
            case SchedulerPhase.midFrameMicrotasks:
            case SchedulerPhase.persistentCallbacks:
                return;
        }
    }

    /// 如果有必要的话，通过调用 [Window.scheduleFrame] 来调度一帧
    ///
    /// 在这个调用之后，引擎将会 (且最终) 会回调 [handleBeginFrame] (此回调可能会延迟一段时间。例如设备的屏
    /// 幕熄灭。直到设备被重新打开为止，不会调用 [handleBeginFrame]）。在帧期间调用此回调会导致另外的一帧被调
    /// 度，即是当前的帧还没有被完成。
    void scheduleFrame() {
        if (_hasScheduledFrame || !_framesEnabled)
            return;
        /// 确保 window 绑定了正确的回调
        ensureFrameCallbacksRegistered();
        window.scheduleFrame();
        _hasScheduledFrame = true;
    }

    /// 通过调用 [Window.scheduleFrame]，强制调度一帧
    ///
    /// 在此调用之后，引擎会强制回调 [handleBeginFrame]（例如，在屏幕熄灭的时候）
    ///
    /// 框架使用这一方法来强制应用渲染正确的大小（例如，熄灭屏幕并且旋转（即改变了设备的垂直方向）了设备，
    /// 那么，框架调用此方法来确保在屏幕恢复后能够正常显示）
    ///
    /// 希望尽快地更新一帧的时候，考虑使用 [scheduleWarmUpFrame]
    void scheduleForcedFrame() {
        if (_hasScheduledFrame)
            return;
        window.scheduleFrame();
        _hasScheduledFrame = true;
    }

    bool _warmUpFrame = false;

    /// 尽可能快地调度一帧，而不是等待引擎响应垂直同步信号
    ///
    /// 则通常在应用开始的第一帧时（这一帧相对来说是及其昂贵的），被调用，让它获取额外的时间来运行
    ///
    /// 锁定事件派发，直到帧调度完成
    ///
    /// 如果 [scheduleFrame] 或者 [scheduleForcedFrame] 已经被调用了，这个调用将会延迟已经调度了的帧
    ///
    /// 如果任何其他的预热帧已经被调度或者正在进行一帧，此调用将会被忽略
    void scheduleWarmUpFrame() {
        if (_warmUpFrame || schedulerPhase != SchedulerPhase.idle)
            return;

        _warmUpFrame = true;
        Timeline.startSync('Warm-up frame');
        final bool hadScheduledFrame = _hasScheduledFrame;
        // 在这里使用 Timer 来确保微任务队列进行
        Timer.run(() {
            assert(_warmUpFrame);
            handleBeginFrame(null);
        });
        Timer.run(() {
            assert(_warmUpFrame);
            handleDrawFrame();
            resetEpoch();
            _warmUpFrame = false;
            if (hadScheduledFrame)
                scheduleFrame();
        });

        lockEvents(() async {
            await endOfFrame;
            Timeline.finishSync();
        });
    }

    Duration _firstRawTimeStampInEpoch;
    Duration _epochStart = Duration.zero;
    Duration _lastRawTimeStamp = Duration.zero;

    /// Prepares the scheduler for a non-monotonic change to how time stamps are
    /// calculated.
    ///
    /// Callbacks received from the scheduler assume that their time stamps are
    /// monotonically increasing. The raw time stamp passed to [handleBeginFrame]
    /// is monotonic, but the scheduler might adjust those time stamps to provide
    /// [timeDilation]. Without careful handling, these adjusts could cause time
    /// to appear to run backwards.
    ///
    /// The [resetEpoch] function ensures that the time stamps are monotonic by
    /// resetting the base time stamp used for future time stamp adjustments to the
    /// current value. For example, if the [timeDilation] decreases, rather than
    /// scaling down the [Duration] since the beginning of time, [resetEpoch] will
    /// ensure that we only scale down the duration since [resetEpoch] was called.
    ///
    /// Setting [timeDilation] calls [resetEpoch] automatically. You don't need to
    /// call [resetEpoch] yourself.
    void resetEpoch() {
        _epochStart = _adjustForEpoch(_lastRawTimeStamp);
        _firstRawTimeStampInEpoch = null;
    }

    /// Adjusts the given time stamp into the current epoch.
    ///
    /// This both offsets the time stamp to account for when the epoch started
    /// (both in raw time and in the epoch's own time line) and scales the time
    /// stamp to reflect the time dilation in the current epoch.
    ///
    /// These mechanisms together combine to ensure that the durations we give
    /// during frame callbacks are monotonically increasing.
    Duration _adjustForEpoch(Duration rawTimeStamp) {
        final Duration rawDurationSinceEpoch = _firstRawTimeStampInEpoch == null ? Duration.zero : rawTimeStamp - _firstRawTimeStampInEpoch;
        return Duration(microseconds: (rawDurationSinceEpoch.inMicroseconds / timeDilation).round() + _epochStart.inMicroseconds);
    }

    /// The time stamp for the frame currently being processed.
    ///
    /// This is only valid while between the start of [handleBeginFrame] and the
    /// end of the corresponding [handleDrawFrame], i.e. while a frame is being
    /// produced.
    Duration get currentFrameTimeStamp {
        assert(_currentFrameTimeStamp != null);
        return _currentFrameTimeStamp;
    }
    Duration _currentFrameTimeStamp;

    /// The raw time stamp as provided by the engine to [Window.onBeginFrame]
    /// for the frame currently being processed.
    ///
    /// Unlike [currentFrameTimeStamp], this time stamp is neither adjusted to
    /// offset when the epoch started nor scaled to reflect the [timeDilation] in
    /// the current epoch.
    ///
    /// On most platforms, this is a more or less arbitrary value, and should
    /// generally be ignored. On Fuchsia, this corresponds to the system-provided
    /// presentation time, and can be used to ensure that animations running in
    /// different processes are synchronized.
    Duration get currentSystemFrameTimeStamp {
        assert(_lastRawTimeStamp != null);
        return _lastRawTimeStamp;
    }

	
    /// 当调度一帧之后，引擎会回调此方法（此方法被绑定到 window.onBeginFrame）
    void _handleBeginFrame(Duration rawTimeStamp) {
        /// 这里区别时候是预热帧，如果是，把 _ignoreNextEngineDrawFrame 设置为 true
        if (_warmUpFrame) {
            assert(!_ignoreNextEngineDrawFrame);
            _ignoreNextEngineDrawFrame = true;
            return;
        }
        handleBeginFrame(rawTimeStamp);
    }

    /// 当调度一帧之后，引擎会在回调 _handleBeginFrame 之后，回调此方法。（此方法被绑定到 
    /// window.onDrawFrame ）
    void _handleDrawFrame() {
        /// 因为是预热帧，这里不进行实际的构建和绘制
        if (_ignoreNextEngineDrawFrame) {
            _ignoreNextEngineDrawFrame = false;
            return;
        }
        handleDrawFrame();
    }

    /// Called by the engine to prepare the framework to produce a new frame.
    ///
    /// This function calls all the transient frame callbacks registered by
    /// [scheduleFrameCallback]. It then returns, any scheduled microtasks are run
    /// (e.g. handlers for any [Future]s resolved by transient frame callbacks),
    /// and [handleDrawFrame] is called to continue the frame.
    ///
    /// If the given time stamp is null, the time stamp from the last frame is
    /// reused.
    ///
    /// To have a banner shown at the start of every frame in debug mode, set
    /// [debugPrintBeginFrameBanner] to true. The banner will be printed to the
    /// console using [debugPrint] and will contain the frame number (which
    /// increments by one for each frame), and the time stamp of the frame. If the
    /// given time stamp was null, then the string "warm-up frame" is shown
    /// instead of the time stamp. This allows frames eagerly pushed by the
    /// framework to be distinguished from those requested by the engine in
    /// response to the "Vsync" signal from the operating system.
    ///
    /// You can also show a banner at the end of every frame by setting
    /// [debugPrintEndFrameBanner] to true. This allows you to distinguish log
    /// statements printed during a frame from those printed between frames (e.g.
    /// in response to events or timers).
    void handleBeginFrame(Duration rawTimeStamp) {
        Timeline.startSync('Frame', arguments: timelineWhitelistArguments);
        _firstRawTimeStampInEpoch ??= rawTimeStamp;
        _currentFrameTimeStamp = _adjustForEpoch(rawTimeStamp ?? _lastRawTimeStamp);
        if (rawTimeStamp != null)
            _lastRawTimeStamp = rawTimeStamp;
        _hasScheduledFrame = false;
        try {
            // TRANSIENT FRAME CALLBACKS
            Timeline.startSync('Animate', arguments: timelineWhitelistArguments);
            _schedulerPhase = SchedulerPhase.transientCallbacks;
            final Map<int, _FrameCallbackEntry> callbacks = _transientCallbacks;
            _transientCallbacks = <int, _FrameCallbackEntry>{};
            callbacks.forEach((int id, _FrameCallbackEntry callbackEntry) {
                if (!_removedIds.contains(id))
                    _invokeFrameCallback(callbackEntry.callback, _currentFrameTimeStamp, callbackEntry.debugStack);
            });
            _removedIds.clear();
        } finally {
            _schedulerPhase = SchedulerPhase.midFrameMicrotasks;
        }
    }

    void handleDrawFrame() {
        assert(_schedulerPhase == SchedulerPhase.midFrameMicrotasks);
        Timeline.finishSync(); // end the "Animate" phase
        try {
            // PERSISTENT FRAME CALLBACKS
            _schedulerPhase = SchedulerPhase.persistentCallbacks;
            for (FrameCallback callback in _persistentCallbacks)
                _invokeFrameCallback(callback, _currentFrameTimeStamp);

            // POST-FRAME CALLBACKS
            _schedulerPhase = SchedulerPhase.postFrameCallbacks;
            final List<FrameCallback> localPostFrameCallbacks =
                List<FrameCallback>.from(_postFrameCallbacks);
            _postFrameCallbacks.clear();
            for (FrameCallback callback in localPostFrameCallbacks)
                _invokeFrameCallback(callback, _currentFrameTimeStamp);
        } finally {
            _schedulerPhase = SchedulerPhase.idle;
            Timeline.finishSync(); // end the Frame
            assert(() {
                if (debugPrintEndFrameBanner)
                    debugPrint('▀' * _debugBanner.length);
                _debugBanner = null;
                return true;
            }());
            _currentFrameTimeStamp = null;
        }
    }

    void _profileFramePostEvent(int frameNumber, FrameTiming frameTiming) {
        postEvent('Flutter.Frame', <String, dynamic>{
            'number': frameNumber,
            'startTime': frameTiming.timestampInMicroseconds(FramePhase.buildStart),
            'elapsed': frameTiming.totalSpan.inMicroseconds,
            'build': frameTiming.buildDuration.inMicroseconds,
            'raster': frameTiming.rasterDuration.inMicroseconds,
        });
    }

    // Calls the given [callback] with [timestamp] as argument.
    //
    // Wraps the callback in a try/catch and forwards any error to
    // [debugSchedulerExceptionHandler], if set. If not set, then simply prints
    // the error.
    void _invokeFrameCallback(FrameCallback callback, Duration timeStamp, [ StackTrace callbackStack ]) {
        try {
            callback(timeStamp);
        } catch (exception, exceptionStack) {
            ...
        }
    }
}
```



## PaintingBinding

```dart
/// 绘制库的 binding
///
/// 涉及到缓存回收逻辑，以清除图像缓存。
///
/// 要求 [ServicesBinding] 被提前使用（混入）
mixin PaintingBinding on BindingBase, ServicesBinding {
    @override
    void initInstances() {
        super.initInstances();
        _instance = this;
        /// 在初始化的时候创建了图片缓存
        _imageCache = createImageCache();
        if (shaderWarmUp != null) {
            shaderWarmUp.execute();
        }
    }
    
    static PaintingBinding get instance => _instance;
    static PaintingBinding _instance;

    /// [initInstances] 期间执行的 [ShaderWarmUp]
    ///
    /// 如果应用程序有需要编译没有被 [DefaultShaderWarmUp] 所包含的复杂的着色器的场景，它可能会导致在动画或
    /// 交互时的卡顿。在这种情况下，在调用 [initInstances]之前( 通常在[runApp]之前，在
    /// [enableFlutterDriverExtension] 之前，为 Flutter 驱动程序测试之前)，将 [ShaderWarmUp] 设置为自定义
    /// 的 [ShaderWarmUp]。在自定义 [ShaderWarmUp] 中绘制场景，这样 Flutter 可以在启动时预编译和缓存着色
    /// 器。在此期间，预热的代价是昂贵的(100ms-200ms，取决于要编译的着色器)，所以应该不会出现“应用程序没有响
    /// 应”的警告。
    ///
    /// 目前，预热过程与 GPU 线程同步进行，也就是意味着，GPU 上第一帧的渲染会被推迟到预热完成之后进行
    static ShaderWarmUp shaderWarmUp = const DefaultShaderWarmUp();

    /// 实现了 Flutetr 框架图片缓存的单例
    ///
    /// 此缓存被 [ImageProvider] 内部使用，且通常情况下，不应被直接访问
    ///
    /// 此缓存被 [createImageCache] 方法所创建
    ImageCache get imageCache => _imageCache;
    ImageCache _imageCache;

    /// 创建图片缓存单例 [ImageCache] (可以通过 [imageCache] 来访问).
    ///
    /// 此方法能够被重载以提供一个自定义的图片缓存
    @protected
    ImageCache createImageCache() => ImageCache();

    /// 通过 [ImageCache ]中的 [decodedCacheRatioCap] 调用 [dart:ui]。
    ///
    /// 当指定 [cacheWidth] 和 [cacheHeight] 参数时，表示了要解码图像的大小。
    ///
    /// [cacheWidth] 和 [cacheHeight] 必须是大于或等于 1 的正值或者 null。只指定 [cacheWidth] 和
    /// [cacheHeight] 的一者，而另一个值为 null，在这种情况下，省略的维度将解码为其原始大小。
    /// 当两者都为 null 或省略时，图像将被解码成其原始大小
    Future<ui.Codec> instantiateImageCodec(Uint8List bytes, {
        int cacheWidth,
        int cacheHeight,
    }) {
        assert(cacheWidth == null || cacheWidth > 0);
        assert(cacheHeight == null || cacheHeight > 0);
        return ui.instantiateImageCodec(
            bytes,
            targetWidth: cacheWidth,
            targetHeight: cacheHeight,
        );
    }

    @override
    void evict(String asset) {
        super.evict(asset);
        imageCache.clear();
    }

    /// 系统字体改变的可监听（即当系统字体变化时，通知所有的监听者）
    ///
    /// 当系统添加或者移除字体的时候，系统字体可能发生改变。在这种情况下，为了能够正确地反映变化，必须重新布局
    /// 绘制与文字相关的组件
    ///
    /// 展示文字的对象，例如 [TextPainter] 或者 [Paragraph] 应该监听此字段，并且在必要的时候进行重绘
    Listenable get systemFonts => _systemFonts;
    final _SystemFontsNotifier _systemFonts = _SystemFontsNotifier();

    /// 当系统发送字体变化的消息时，触发监听者回调
    @override
    Future<void> handleSystemMessage(Object systemMessage) async {
        await super.handleSystemMessage(systemMessage);
        final Map<String, dynamic> message = systemMessage as Map<String, dynamic>;
        final String type = message['type'] as String;
        switch (type) {
            case 'fontsChange':
                _systemFonts.notifyListeners();
                break;
        }
        return;
    }
}
```



## SematicsBinding 略



## RenderingBinding

```dart
/// 渲染 Binding
/// 
/// 这个类联合了渲染对象和 Flutter 引擎
/// 注意以下混入的顺序要求了：使用此混入之前必须先混入（实现、继承）BindingBase,ServicesBinding, 
/// SchedulerBinding,GestureBinding,SemanticsBinding,HitTestable
mixin RendererBinding 
   on BindingBase, 
      ServicesBinding, 
      SchedulerBinding, 
      GestureBinding, 
      SemanticsBinding, 
      HitTestable 
{
    @override
    void initInstances() {
        super.initInstances();
        _instance = this;
        /// 创建一个渲染流水线持有者单例
        _pipelineOwner = PipelineOwner(
            onNeedVisualUpdate: ensureVisualUpdate,
            onSemanticsOwnerCreated: _handleSemanticsOwnerCreated,
            onSemanticsOwnerDisposed: _handleSemanticsOwnerDisposed,
        );
        /// 为 window 绑定了处理平台变化的方法
        window
            ..onMetricsChanged = handleMetricsChanged
            ..onTextScaleFactorChanged = handleTextScaleFactorChanged
            ..onPlatformBrightnessChanged = handlePlatformBrightnessChanged
            ..onSemanticsEnabledChanged = _handleSemanticsEnabledChanged
            ..onSemanticsAction = _handleSemanticsAction;
        /// 初始化了 RenderView，即创建了
        initRenderView();
        _handleSemanticsEnabledChanged();
        assert(renderView != null);
        /// 在 SchedulerBinding 中的 _persistentFrameCallback 列表中添加了驱动渲染流水线的回调
        addPersistentFrameCallback(_handlePersistentFrameCallback);
        /// 初始化鼠标追踪器
        initMouseTracker();
    }

    static RendererBinding get instance => _instance;
    static RendererBinding _instance;

    @override
    void initServiceExtensions() {
        super.initServiceExtensions();
        if (!kReleaseMode) {
            // these service extensions work in debug or profile mode
            registerSignalServiceExtension(
                name: 'debugDumpRenderTree',
                callback: () {
                    debugDumpRenderTree();
                    return debugPrintDone;
                },
            );

            registerSignalServiceExtension(
                name: 'debugDumpSemanticsTreeInTraversalOrder',
                callback: () {
                    debugDumpSemanticsTree(DebugSemanticsDumpOrder.traversalOrder);
                    return debugPrintDone;
                },
            );

            registerSignalServiceExtension(
                name: 'debugDumpSemanticsTreeInInverseHitTestOrder',
                callback: () {
                    debugDumpSemanticsTree(DebugSemanticsDumpOrder.inverseHitTest);
                    return debugPrintDone;
                },
            );
        }
    }

    /// 创建一个 [RenderView] 作为渲染对象树的树根，并且初始化它。因此，在下一帧的时候，它将会被渲染。
    void initRenderView() {
        assert(renderView == null);
        /// 通过配置的 ViewConfiguration 来创建一个 RenderView
        renderView = RenderView(configuration: createViewConfiguration(), window: window);
        /// 为第一帧做准备，调用了以下两个回调
        /// scheduleInitialLayout();
    	/// scheduleInitialPaint(_updateMatricesAndCreateNewRootLayer());
        renderView.prepareInitialFrame();
    }

    /// 鼠标跟踪器，它维护了当前鼠标的状态（例如，派发鼠标的悬停消息）
    MouseTracker get mouseTracker => _mouseTracker;
    MouseTracker _mouseTracker;

    /// 渲染了对象树的持有者，它维护了布局、绘制、语义的状态
    PipelineOwner get pipelineOwner => _pipelineOwner;
    PipelineOwner _pipelineOwner;

    /// 渲染对象树的根节点
    /// 注意到，这个 mixin 并不直接持有 renderView，而是直接持有 pipelineOwner 来间接访问 renderView
    RenderView get renderView => _pipelineOwner.rootNode as RenderView;
    set renderView(RenderView value) {
        assert(value != null);
        _pipelineOwner.rootNode = value;
    }

    /// 当屏幕像素大小发生改变（例如，键盘弹出）的时候的回调
    @protected
    void handleMetricsChanged() {
        assert(renderView != null);
        renderView.configuration = createViewConfiguration();
        /// 强制刷新一帧
        scheduleForcedFrame();
    }

    /// 字体大小因子发生改变时的回调
    @protected
    void handleTextScaleFactorChanged() { }

    /// 平台亮度发生改变时的回调
    ///
    /// 当前的平台亮度可以通过此字段或者 MediaQuery 直接获取。后者会在变化的时候重建与之相关的 widget
    @protected
    void handlePlatformBrightnessChanged() { }

    /// 返回一个基于当前环境配置的 [ViewConfiguration]
    ///
    ViewConfiguration createViewConfiguration() {
        final double devicePixelRatio = window.devicePixelRatio;
        return ViewConfiguration(
            size: window.physicalSize / devicePixelRatio,
            devicePixelRatio: devicePixelRatio,
        );
    }

    SemanticsHandle _semanticsHandle;

    /// 创建一个管理当前鼠标状态的 [MouseTracker]
    @visibleForTesting
    void initMouseTracker([MouseTracker tracker]) {
        _mouseTracker?.dispose();
        _mouseTracker = tracker ?? MouseTracker(pointerRouter, renderView.hitTestMouseTrackers);
    }

    /// 语义启用下的回调
    void _handleSemanticsEnabledChanged() {
        setSemanticsEnabled(window.semanticsEnabled);
    }

    /// 语义相关。。。
    void setSemanticsEnabled(bool enabled) {
        if (enabled) {
            _semanticsHandle ??= _pipelineOwner.ensureSemantics();
        } else {
            _semanticsHandle?.dispose();
            _semanticsHandle = null;
        }
    }

    void _handleSemanticsAction(int id, SemanticsAction action, ByteData args) {
        _pipelineOwner.semanticsOwner?.performAction(
            id,
            action,
            args != null ? const StandardMessageCodec().decodeMessage(args) : null,
        );
    }

    void _handleSemanticsOwnerCreated() {
        renderView.scheduleInitialSemantics();
    }

    void _handleSemanticsOwnerDisposed() {
        renderView.clearSemantics();
    }

    /// 处理持久帧回调
    /// 这个回调在此混入类被初始化的时候加入到了 SchedulerBinding 的持久帧回调当中，也就是说，在每一帧，此方
    /// 法都会被回调一次
    void _handlePersistentFrameCallback(Duration timeStamp) {
        /// 绘制一帧，WidgetBinding 重载了这个方法
        drawFrame();
        /// 在帧尾检测鼠标的状态
        _mouseTracker.schedulePostFrameCheck();
    }

    int _firstFrameDeferredCount = 0;
    bool _firstFrameSent = false;

    /// 被 [drawFrame] 产生的帧是否应该提交给引擎
    ///
    /// 如果为 false，框架依旧生产一帧，但不会将它提交给引擎去渲染
    bool get sendFramesToEngine => _firstFrameSent || _firstFrameDeferredCount == 0;

    /// 告诉引擎不要将第一帧送入引擎去渲染，直到相应的 [allowFirstFrame] 被调用
    ///
    /// 调用此方法来在首帧渲染之前完成异步初始化的工作 (在开屏页之前)。框架依旧会完成所有的工作来产生一帧（也
    /// 就是说，会完成构建、布局、绘制等流程），当不会将这一帧提交给引擎层去显示
    void deferFirstFrame() {
        assert(_firstFrameDeferredCount >= 0);
        _firstFrameDeferredCount += 1;
    }

    /// 在 [deferFirstFrame] 调用之后，告诉引擎现在可以将第一帧提交给引擎去渲染
    ///
    /// 在 [schedulerPhase] == [SchedulerPhase.idle] 的时候，调用此方法的性能是最佳的 
    ///
    /// 此方法只能调用与对应 [deferFirstFrame] 相应的次数（类似于栈，deferFirstFrame 如同入栈， 
    /// allowFirstFrame 则是出栈。栈为空时才会开始第一帧的渲染
    void allowFirstFrame() {
        assert(_firstFrameDeferredCount > 0);
        _firstFrameDeferredCount -= 1;
        // 即使延迟计数尚未降至零，也要始终安排一个预热帧，因为取消延迟可能会发现组件树中更低的新延迟.
        if (!_firstFrameSent)
            scheduleWarmUpFrame();
    }

    /// 调用此方法来假装还没有向引擎发送一帧。（即把下一帧视为送到引擎的第一帧）
    ///
    /// 此方法可以用于测试 [deferFirstFrame] 和 [allowFirstFrame]，因为它们只在第一帧还没有送到引擎里的时候
    /// 有效
    void resetFirstFrameSent() {
        _firstFrameSent = false;
    }
          
    /// 驱动渲染流水线绘制一帧（即完成布局、绘制等操作
    /// 此方法被 widgetsBinding 所重载，在绘制布局之前完成了 widget 树的构建
    @protected
    void drawFrame() {
        assert(renderView != null);
        /// 冲刷布局
        pipelineOwner.flushLayout();
        pipelineOwner.flushCompositingBits();
        /// 冲刷绘制
        pipelineOwner.flushPaint();
        if (sendFramesToEngine) {
            /// 将合成完的 scence 提交给引擎层，显示一帧
            renderView.compositeFrame();
            pipelineOwner.flushSemantics(); // this also sends the semantics to the OS.
            _firstFrameSent = true;
        }
    }

    /// 应用的重组，此方法被 hot reload 调用，仅在测试模式下有效
    @override
    Future<void> performReassemble() async {
        await super.performReassemble();
        Timeline.startSync('Dirty Render Tree', arguments: timelineWhitelistArguments);
        try {
            renderView.reassemble();
        } finally {
            Timeline.finishSync();
        }
        scheduleWarmUpFrame();
        await endOfFrame;
    }

    /// 点击测试
    /// 对渲染树的根节点进行点击测试，然后调用超类（GesutureBinding.hitTest) 的点击测试
    @override
    void hitTest(HitTestResult result, Offset position) {
        assert(renderView != null);
        renderView.hitTest(result, position: position);
        super.hitTest(result, position);
    }

    /// 强制重绘
    Future<void> _forceRepaint() {
        RenderObjectVisitor visitor;
        visitor = (RenderObject child) {
            child.markNeedsPaint();
            child.visitChildren(visitor);
        };
        instance?.renderView?.visitChildren(visitor);
        return endOfFrame;
    }
}

```



## WidgetsBinding

```dart
/// 组件 Binding
///
/// 此混入类联合了 Flutter 组件层和引擎层
mixin WidgetsBinding 
   on BindingBase, 
      ServicesBinding, 
      SchedulerBinding, 
      GestureBinding, 
      RendererBinding, 
      SemanticsBinding 
{
    @override
    void initInstances() {
        super.initInstances();
        _instance = this;
        // 初始化 BuildOwner 必须在 [super.initInstances] 被调用之后，因为它要求 [ServicesBinding] 在之前
        // 已经创建了 [defaultBinaryMessenger] 实例
        _buildOwner = BuildOwner();
        /// 为 buildOwner 绑定调度方法
        buildOwner.onBuildScheduled = _handleBuildScheduled;
        /// 本地化改变回调
        window.onLocaleChanged = handleLocaleChanged;
        /// 可访问性改变回调
        window.onAccessibilityFeaturesChanged = handleAccessibilityFeaturesChanged;
        /// 系统路由回调（加入、退出路由）
        SystemChannels.navigation.setMethodCallHandler(_handleNavigationInvocation);
        FlutterErrorDetails.propertiesTransformers.add(transformDebugCreator);
    }
          
    /// 如果此 Binding 在 [runApp] 之前就被构建，可以直接调用 [WidgetsFlutterBinding.ensureInitialized]
    static WidgetsBinding get instance => _instance;
    static WidgetsBinding _instance;

    @override
    void initServiceExtensions() {
        super.initServiceExtensions();

        if (!kReleaseMode) {
            registerSignalServiceExtension(
                name: 'debugDumpApp',
                callback: () {
                    debugDumpApp();
                    return debugPrintDone;
                },
            );

            if (!kIsWeb) {
                registerBoolServiceExtension(
                    name: 'showPerformanceOverlay',
                    getter: () =>
                    Future<bool>.value(WidgetsApp.showPerformanceOverlayOverride),
                    setter: (bool value) {
                        if (WidgetsApp.showPerformanceOverlayOverride == value)
                            return Future<void>.value();
                        WidgetsApp.showPerformanceOverlayOverride = value;
                        return _forceRebuild();
                    },
                );
            }

            registerServiceExtension(
                name: 'didSendFirstFrameEvent',
                callback: (_) async {
                    return <String, dynamic>{
                        // This is defined to return a STRING, not a boolean.
                        // Devtools, the Intellij plugin, and the flutter tool all depend
                        // on it returning a string and not a boolean.
                        'enabled': _needToReportFirstFrame ? 'false' : 'true',
                    };
                },
            );

            // This returns 'true' when the first frame is rasterized, and the trace
            // event 'Rasterized first useful frame' is sent out.
            registerServiceExtension(
                name: 'didSendFirstFrameRasterizedEvent',
                callback: (_) async {
                    return <String, dynamic>{
                        // This is defined to return a STRING, not a boolean.
                        // Devtools, the Intellij plugin, and the flutter tool all depend
                        // on it returning a string and not a boolean.
                        'enabled': firstFrameRasterized ? 'true' : 'false',
                    };
                },
            );

            // Register the ability to quickly mark elements as dirty.
            // The performance of this method may be improved with additional
            // information from https://github.com/flutter/flutter/issues/46195.
            registerServiceExtension(
                name: 'fastReassemble',
                callback: (Map<String, Object> params) async {
                    final String className = params['class'] as String;
                    void markElementsDirty(Element element) {
                        if (element == null) {
                            return;
                        }
                        if (element.widget?.runtimeType?.toString()?.startsWith(className) ?? false) {
                            element.markNeedsBuild();
                        }
                        element.visitChildElements(markElementsDirty);
                    }
                    markElementsDirty(renderViewElement);
                    return <String, String>{'Success': 'true'};
                },
            );

            // Expose the ability to send Widget rebuilds as [Timeline] events.
            registerBoolServiceExtension(
                name: 'profileWidgetBuilds',
                getter: () async => debugProfileBuildsEnabled,
                setter: (bool value) async {
                    if (debugProfileBuildsEnabled != value)
                        debugProfileBuildsEnabled = value;
                },
            );
        }
    }

    /// 强制重建
    Future<void> _forceRebuild() {
        if (renderViewElement != null) {
            buildOwner.reassemble(renderViewElement);
            return endOfFrame;
        }
        return Future<void>.value();
    }

    /// 掌管了构建流水线的 buildOwner
    BuildOwner get buildOwner => _buildOwner;
    BuildOwner _buildOwner;

    /// 焦点树管理帧，即管理焦点树的对象
    ///
    /// 很少被直接使用。通常，给定 context 且调用 [FocusScope.of] 来获取到 [FocusScopeNode]
    FocusManager get focusManager => _buildOwner.focusManager;

    final List<WidgetsBindingObserver> _observers = <WidgetsBindingObserver>[];

    /// 注册一个给定的 WidgetsBindingObserver 
    /// 在各种应用事件发生的时候，观测者会被通知（例如，系统本地化改变）通常情况下，组件树只有一个组件将它自身
    /// 作为 WidgetsBindingObserver，并且将系统的状态转换成一个 inherited widgets.
    ///
    /// 例如，[WidgetsApp] 注册了一个观测者，并且在每次构建的时候将屏幕大小传递给了 [MediaQuery] 组件，这使
    /// 得每一个调用了 [MediaQuery.of] 方法的组件能够获取到屏幕大小（例如屏幕旋转）改变的通知 
    void addObserver(WidgetsBindingObserver observer) => _observers.add(observer);
          
    /// 注销给定的观察者，应当谨慎使用这一办法，因为它的代价相对较高(O(N))。
    bool removeObserver(WidgetsBindingObserver observer) => _observers.remove(observer);

    /// 处理屏幕大小改变，通知观测者
    @override
    void handleMetricsChanged() {
        super.handleMetricsChanged();
        for (WidgetsBindingObserver observer in _observers)
            observer.didChangeMetrics();
    }

    /// 处理字体因子改变，通知观测者
    @override
    void handleTextScaleFactorChanged() {
        super.handleTextScaleFactorChanged();
        for (WidgetsBindingObserver observer in _observers)
            observer.didChangeTextScaleFactor();
    }

    /// 处理平台亮度改变，通知观测者
    @override
    void handlePlatformBrightnessChanged() {
        super.handlePlatformBrightnessChanged();
        for (WidgetsBindingObserver observer in _observers)
            observer.didChangePlatformBrightness();
    }

    /// 处理可访问性改变，通知观测者
    @override
    void handleAccessibilityFeaturesChanged() {
        super.handleAccessibilityFeaturesChanged();
        for (WidgetsBindingObserver observer in _observers)
            observer.didChangeAccessibilityFeatures();
    }

    /// 处理系统本地化改变。通过调用 [dispatchLocalesChanged] 来通知观测者
    @protected
    @mustCallSuper
    void handleLocaleChanged() {
        dispatchLocalesChanged(window.locales);
    }

    /// 用给定的 locales 来通知观测者
    @protected
    @mustCallSuper
    void dispatchLocalesChanged(List<Locale> locales) {
        for (WidgetsBindingObserver observer in _observers)
            observer.didChangeLocales(locales);
    }

    @protected
    @mustCallSuper
    void dispatchAccessibilityFeaturesChanged() {
        for (WidgetsBindingObserver observer in _observers)
            observer.didChangeAccessibilityFeatures();
    }

    /// 系统退出当前路由时的回调
    ///
    /// 首先，这个调用会遍历观测者，调用它们的 [WidgetsBindingObserver.didPopRoute] 方法，直到它们中的一个
    /// 返回了 true（这意味着它有能力处理路由退出事件，例如，弹出一个对话框）。如果没有观测者返回 true，那么将
    /// 会调用 [SystemNavigator.pop] 关闭应用
    @protected
    Future<void> handlePopRoute() async {
        for (WidgetsBindingObserver observer in List<WidgetsBindingObserver>.from(_observers)) {
            if (await observer.didPopRoute())
                return;
        }
        SystemNavigator.pop();
    }

    /// 宿主告诉此应用需要推送一个路由
    ///
    /// 首先，这个调用会遍历观测者，调用它们的 [WidgetsBindingObserver.didPopRoute] 方法，直到它们中的一个
    /// 返回了 true（这意味着它有能力处理路由退出事件，例如，弹出一个对话框）。如果没有观测者返回 true，那么什
    /// 么也不会发生
    @protected
    @mustCallSuper
    Future<void> handlePushRoute(String route) async {
        for (WidgetsBindingObserver observer in List<WidgetsBindingObserver>.from(_observers)) {
            if (await observer.didPushRoute(route))
                return;
        }
    }

    /// 此方法作为系统导航的监听回调
    Future<dynamic> _handleNavigationInvocation(MethodCall methodCall) {
        switch (methodCall.method) {
            /// 路由退出
            case 'popRoute':
                return handlePopRoute();
            /// 插入路由
            case 'pushRoute':
                return handlePushRoute(methodCall.arguments as String);
        }
        return Future<dynamic>.value();
    }

    /// 通知观测者，应用的什么周期发生了改变
    @override
    void handleAppLifecycleStateChanged(AppLifecycleState state) {
        super.handleAppLifecycleStateChanged(state);
        for (WidgetsBindingObserver observer in _observers)
            observer.didChangeAppLifecycleState(state);
    }

    /// 内存压力回调，通知观测者
    void handleMemoryPressure() {
        for (WidgetsBindingObserver observer in _observers)
            observer.didHaveMemoryPressure();
    }

    /// 处理平台消息，只处理内存压力
    @override
    Future<void> handleSystemMessage(Object systemMessage) async {
        await super.handleSystemMessage(systemMessage);
        final Map<String, dynamic> message = systemMessage as Map<String, dynamic>;
        final String type = message['type'] as String;
        switch (type) {
            case 'memoryPressure':
                handleMemoryPressure();
                break;
        }
        return;
    }

    bool _needToReportFirstFrame = true;

    final Completer<void> _firstFrameCompleter = Completer<void>();

    /// Whether the Flutter engine has rasterized the first frame.
    ///
    /// {@macro flutter.frame_rasterized_vs_presented}
    ///
    /// See also:
    ///
    ///  * [waitUntilFirstFrameRasterized], the future when [firstFrameRasterized]
    ///    becomes true.
    bool get firstFrameRasterized => _firstFrameCompleter.isCompleted;

    /// A future that completes when the Flutter engine has rasterized the first
    /// frame.
    ///
    /// {@macro flutter.frame_rasterize_vs_presented}
    ///
    /// See also:
    ///
    ///  * [firstFrameRasterized], whether this future has completed or not.
    Future<void> get waitUntilFirstFrameRasterized => _firstFrameCompleter.future;

    /// Tell the framework not to report the frame it is building as a "useful"
    /// first frame until there is a corresponding call to [allowFirstFrameReport].
    ///
    /// Deprecated. Use [deferFirstFrame]/[allowFirstFrame] to delay rendering the
    /// first frame.
    @Deprecated(
        'Use deferFirstFrame/allowFirstFrame to delay rendering the first frame. '
        'This feature was deprecated after v1.12.4.'
    )
    void deferFirstFrameReport() {
        if (!kReleaseMode) {
            deferFirstFrame();
        }
    }

    /// 调度一次构建（通常是 [State.setState]）之后，会触发视觉更新
    void _handleBuildScheduled() {        
        ensureVisualUpdate();
    }

    /// 绘制一帧，此方法重载了 RenderingBinding.drawFrame
    @override
    void drawFrame() {
        TimingsCallback firstFrameCallback;
        if (_needToReportFirstFrame) {
            assert(!_firstFrameCompleter.isCompleted);

            firstFrameCallback = (List<FrameTiming> timings) {
                assert(sendFramesToEngine);
                if (!kReleaseMode) {
                    developer.Timeline.instantSync('Rasterized first useful frame');
                    developer.postEvent('Flutter.FirstFrame', <String, dynamic>{});
                }
                SchedulerBinding.instance.removeTimingsCallback(firstFrameCallback);
                firstFrameCallback = null;
                _firstFrameCompleter.complete();
            };
            // Callback is only invoked when [Window.render] is called. When
            // [sendFramesToEngine] is set to false during the frame, it will not
            // be called and we need to remove the callback (see below).
            SchedulerBinding.instance.addTimingsCallback(firstFrameCallback);
        }

        try {
            if (renderViewElement != null)
                /// 对 buildOnwer 中包含的标记 dirty 的节点进行一次构建（renderViewElement 只用作断言）
                buildOwner.buildScope(renderViewElement);
            /// 调用 super.drawFrame 完成布局、绘制等流程
            super.drawFrame();
            /// 构建完成后，将没有被复用的 element 销毁
            buildOwner.finalizeTree();
        } finally {
            ...
        }
        if (!kReleaseMode) {
            if (_needToReportFirstFrame && sendFramesToEngine) {
                developer.Timeline.instantSync('Widgets built first useful frame');
            }
        }
        _needToReportFirstFrame = false;
        if (firstFrameCallback != null && !sendFramesToEngine) {
            SchedulerBinding.instance.removeTimingsCallback(firstFrameCallback);
        }
    }

    /// Element 树的根节点（即 [RenderObjectToWidgetElement]）
    Element get renderViewElement => _renderViewElement;
    Element _renderViewElement;

    /// 使用 Timer 异步地完成 树根节点的安装
    @protected
    void scheduleAttachRootWidget(Widget rootWidget) {
        Timer.run(() {
            attachRootWidget(rootWidget);
        });
    }

    /// 将给定的 widget 作为根 widget（RenderObjectToWidgetAdapter）的孩子，创建三颗树的树根
    void attachRootWidget(Widget rootWidget) {
        /// 注意：以下调用顺序按是向调用 RenderObjectToWidgetAdapter 的构造函数
        /// 随后调用其 [attachToRenderTree] 方法，将返回值提供给 _renderViewElement
        _renderViewElement = RenderObjectToWidgetAdapter<RenderBox>(
            /// 渲染对象树的根节点
            container: renderView,
            debugShortDescription: '[root]',
            /// runApp 的参数 widget
            child: rootWidget,
        ).attachToRenderTree(
            buildOwner, 
            renderViewElement as RenderObjectToWidgetElement<RenderBox>
        );
    }

    /// 是否 [renderViewElement] 被初始化
    bool get isRootWidgetAttached => _renderViewElement != null;

    @override
    Future<void> performReassemble() {
        assert(() {
            WidgetInspectorService.instance.performReassemble();
            return true;
        }());

        if (renderViewElement != null)
            buildOwner.reassemble(renderViewElement);
        return super.performReassemble();
    }
}
```



## WidgetsFlutterBinding

```dart
class WidgetsFlutterBinding 
    extends BindingBase 
       with GestureBinding, 
            ServicesBinding, 
            SchedulerBinding, 
            PaintingBinding, 
            SemanticsBinding, 
            RendererBinding, 
            WidgetsBinding 
{

  /// 返回 [WidgetsBinding] 的实例，在必要时创建并初始化它。如果创建了一个，则返回的是一个
  /// [WidgetsFlutterBinding]。如果之前初始化已经初始化了，那么它至少会实现 [WidgetsBinding]。
  ///
  /// 只有在需要在调用 [runApp] 之前初始化绑定时，才需要调用这个方法。
  ///
  /// 在“flutter_test”框架中，[testWidgets] 将绑定实例初始化为 [TestWidgetsFlutterBinding]，而不是
  /// [WidgetsFlutterBinding]。
  static WidgetsBinding ensureInitialized() {
    if (WidgetsBinding.instance == null)
      WidgetsFlutterBinding();
    return WidgetsBinding.instance;
  }
}

```

