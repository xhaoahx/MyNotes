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

    /// Schedule a frame to run as soon as possible, rather than waiting for
    /// the engine to request a frame in response to a system "Vsync" signal.
    ///
    /// This is used during application startup so that the first frame (which is
    /// likely to be quite expensive) gets a few extra milliseconds to run.
    ///
    /// Locks events dispatching until the scheduled frame has completed.
    ///
    /// If a frame has already been scheduled with [scheduleFrame] or
    /// [scheduleForcedFrame], this call may delay that frame.
    ///
    /// If any scheduled frame has already begun or if another
    /// [scheduleWarmUpFrame] was already called, this call will be ignored.
    ///
    /// Prefer [scheduleFrame] to update the display in normal operation.
    void scheduleWarmUpFrame() {
        if (_warmUpFrame || schedulerPhase != SchedulerPhase.idle)
            return;

        _warmUpFrame = true;
        Timeline.startSync('Warm-up frame');
        final bool hadScheduledFrame = _hasScheduledFrame;
        // We use timers here to ensure that microtasks flush in between.
        Timer.run(() {
            assert(_warmUpFrame);
            handleBeginFrame(null);
        });
        Timer.run(() {
            assert(_warmUpFrame);
            handleDrawFrame();
            // We call resetEpoch after this frame so that, in the hot reload case,
            // the very next frame pretends to have occurred immediately after this
            // warm-up frame. The warm-up frame's timestamp will typically be far in
            // the past (the time of the last real frame), so if we didn't reset the
            // epoch we would see a sudden jump from the old time in the warm-up frame
            // to the new time in the "real" frame. The biggest problem with this is
            // that implicit animations end up being triggered at the old time and
            // then skipping every frame and finishing in the new time.
            resetEpoch();
            _warmUpFrame = false;
            if (hadScheduledFrame)
                scheduleFrame();
        });

        // Lock events so touch events etc don't insert themselves until the
        // scheduled frame has finished.
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

	
    void _handleBeginFrame(Duration rawTimeStamp) {
        if (_warmUpFrame) {
            assert(!_ignoreNextEngineDrawFrame);
            _ignoreNextEngineDrawFrame = true;
            return;
        }
        handleBeginFrame(rawTimeStamp);
    }

    void _handleDrawFrame() {
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
/// Binding for the painting library.
///
/// Hooks into the cache eviction logic to clear the image cache.
///
/// Requires the [ServicesBinding] to be mixed in earlier.
mixin PaintingBinding on BindingBase, ServicesBinding {
    @override
    void initInstances() {
        super.initInstances();
        _instance = this;
        _imageCache = createImageCache();
        if (shaderWarmUp != null) {
            shaderWarmUp.execute();
        }
    }

    /// The current [PaintingBinding], if one has been created.
    static PaintingBinding get instance => _instance;
    static PaintingBinding _instance;

    /// [ShaderWarmUp] to be executed during [initInstances].
    ///
    /// If the application has scenes that require the compilation of complex
    /// shaders that are not covered by [DefaultShaderWarmUp], it may cause jank
    /// in the middle of an animation or interaction. In that case, set
    /// [shaderWarmUp] to a custom [ShaderWarmUp] before calling [initInstances]
    /// (usually before [runApp] for normal Flutter apps, and before
    /// [enableFlutterDriverExtension] for Flutter driver tests). Paint the scene
    /// in the custom [ShaderWarmUp] so Flutter can pre-compile and cache the
    /// shaders during startup. The warm up is only costly (100ms-200ms,
    /// depending on the shaders to compile) during the first run after the
    /// installation or a data wipe. The warm up does not block the main thread
    /// so there should be no "Application Not Responding" warning.
    ///
    /// Currently the warm-up happens synchronously on the GPU thread which means
    /// the rendering of the first frame on the GPU thread will be postponed until
    /// the warm-up is finished.
    ///
    /// See also:
    ///
    ///  * [ShaderWarmUp], the interface of how this warm up works.
    static ShaderWarmUp shaderWarmUp = const DefaultShaderWarmUp();

    /// The singleton that implements the Flutter framework's image cache.
    ///
    /// The cache is used internally by [ImageProvider] and should generally not
    /// be accessed directly.
    ///
    /// The image cache is created during startup by the [createImageCache]
    /// method.
    ImageCache get imageCache => _imageCache;
    ImageCache _imageCache;

    /// Creates the [ImageCache] singleton (accessible via [imageCache]).
    ///
    /// This method can be overridden to provide a custom image cache.
    @protected
    ImageCache createImageCache() => ImageCache();

    /// Calls through to [dart:ui] with [decodedCacheRatioCap] from [ImageCache].
    ///
    /// The [cacheWidth] and [cacheHeight] parameters, when specified, indicate the
    /// size to decode the image to.
    ///
    /// Both [cacheWidth] and [cacheHeight] must be positive values greater than or
    /// equal to 1 or null. It is valid to specify only one of [cacheWidth] and
    /// [cacheHeight] with the other remaining null, in which case the omitted
    /// dimension will decode to its original size. When both are null or omitted,
    /// the image will be decoded at its native resolution.
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

    /// Listenable that notifies when the available fonts on the system have
    /// changed.
    ///
    /// System fonts can change when the system installs or removes new font. To
    /// correctly reflect the change, it is important to relayout text related
    /// widgets when this happens.
    ///
    /// Objects that show text and/or measure text (e.g. via [TextPainter] or
    /// [Paragraph]) should listen to this and redraw/remeasure.
    Listenable get systemFonts => _systemFonts;
    final _SystemFontsNotifier _systemFonts = _SystemFontsNotifier();

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
/// The glue between the render tree and the Flutter engine.
mixin RendererBinding on BindingBase, ServicesBinding, SchedulerBinding, GestureBinding, SemanticsBinding, HitTestable {
    @override
    void initInstances() {
        super.initInstances();
        _instance = this;
        _pipelineOwner = PipelineOwner(
            onNeedVisualUpdate: ensureVisualUpdate,
            onSemanticsOwnerCreated: _handleSemanticsOwnerCreated,
            onSemanticsOwnerDisposed: _handleSemanticsOwnerDisposed,
        );
        window
            ..onMetricsChanged = handleMetricsChanged
            ..onTextScaleFactorChanged = handleTextScaleFactorChanged
            ..onPlatformBrightnessChanged = handlePlatformBrightnessChanged
            ..onSemanticsEnabledChanged = _handleSemanticsEnabledChanged
            ..onSemanticsAction = _handleSemanticsAction;
        initRenderView();
        _handleSemanticsEnabledChanged();
        assert(renderView != null);
        addPersistentFrameCallback(_handlePersistentFrameCallback);
        initMouseTracker();
    }

    /// The current [RendererBinding], if one has been created.
    static RendererBinding get instance => _instance;
    static RendererBinding _instance;

    @override
    void initServiceExtensions() {
        super.initServiceExtensions();

        assert(() {
            // these service extensions only work in debug mode
            registerBoolServiceExtension(
                name: 'debugPaint',
                getter: () async => debugPaintSizeEnabled,
                setter: (bool value) {
                    if (debugPaintSizeEnabled == value)
                        return Future<void>.value();
                    debugPaintSizeEnabled = value;
                    return _forceRepaint();
                },
            );
            registerBoolServiceExtension(
                name: 'debugPaintBaselinesEnabled',
                getter: () async => debugPaintBaselinesEnabled,
                setter: (bool value) {
                    if (debugPaintBaselinesEnabled == value)
                        return Future<void>.value();
                    debugPaintBaselinesEnabled = value;
                    return _forceRepaint();
                },
            );
            registerBoolServiceExtension(
                name: 'repaintRainbow',
                getter: () async => debugRepaintRainbowEnabled,
                setter: (bool value) {
                    final bool repaint = debugRepaintRainbowEnabled && !value;
                    debugRepaintRainbowEnabled = value;
                    if (repaint)
                        return _forceRepaint();
                    return Future<void>.value();
                },
            );
            registerBoolServiceExtension(
                name: 'debugCheckElevationsEnabled',
                getter: () async => debugCheckElevationsEnabled,
                setter: (bool value) {
                    if (debugCheckElevationsEnabled == value) {
                        return Future<void>.value();
                    }
                    debugCheckElevationsEnabled = value;
                    return _forceRepaint();
                },
            );
            registerSignalServiceExtension(
                name: 'debugDumpLayerTree',
                callback: () {
                    debugDumpLayerTree();
                    return debugPrintDone;
                },
            );
            return true;
        }());

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

    /// Creates a [RenderView] object to be the root of the
    /// [RenderObject] rendering tree, and initializes it so that it
    /// will be rendered when the next frame is requested.
    ///
    /// Called automatically when the binding is created.
    void initRenderView() {
        assert(renderView == null);
        renderView = RenderView(configuration: createViewConfiguration(), window: window);
        renderView.prepareInitialFrame();
    }

    /// The object that manages state about currently connected mice, for hover
    /// notification.
    MouseTracker get mouseTracker => _mouseTracker;
    MouseTracker _mouseTracker;

    /// The render tree's owner, which maintains dirty state for layout,
    /// composite, paint, and accessibility semantics
    PipelineOwner get pipelineOwner => _pipelineOwner;
    PipelineOwner _pipelineOwner;

    /// The render tree that's attached to the output surface.
    RenderView get renderView => _pipelineOwner.rootNode as RenderView;
    /// Sets the given [RenderView] object (which must not be null), and its tree, to
    /// be the new render tree to display. The previous tree, if any, is detached.
    set renderView(RenderView value) {
        assert(value != null);
        _pipelineOwner.rootNode = value;
    }

    /// Called when the system metrics change.
    ///
    /// See [Window.onMetricsChanged].
    @protected
    void handleMetricsChanged() {
        assert(renderView != null);
        renderView.configuration = createViewConfiguration();
        scheduleForcedFrame();
    }

    /// Called when the platform text scale factor changes.
    ///
    /// See [Window.onTextScaleFactorChanged].
    @protected
    void handleTextScaleFactorChanged() { }

    /// Called when the platform brightness changes.
    ///
    /// The current platform brightness can be queried from a Flutter binding or
    /// from a [MediaQuery] widget. The latter is preferred from widgets because
    /// it causes the widget to be automatically rebuilt when the brightness
    /// changes.
    ///
    /// {@tool sample}
    /// Querying [Window.platformBrightness].
    ///
    /// ```dart
    /// final Brightness brightness = WidgetsBinding.instance.window.platformBrightness;
    /// ```
    /// {@end-tool}
    ///
    /// {@tool sample}
    /// Querying [MediaQuery] directly. Preferred.
    ///
    /// ```dart
    /// final Brightness brightness = MediaQuery.platformBrightnessOf(context);
    /// ```
    /// {@end-tool}
    ///
    /// {@tool sample}
    /// Querying [MediaQueryData].
    ///
    /// ```dart
    /// final MediaQueryData mediaQueryData = MediaQuery.of(context);
    /// final Brightness brightness = mediaQueryData.platformBrightness;
    /// ```
    /// {@end-tool}
    ///
    /// See [Window.onPlatformBrightnessChanged].
    @protected
    void handlePlatformBrightnessChanged() { }

    /// Returns a [ViewConfiguration] configured for the [RenderView] based on the
    /// current environment.
    ///
    /// This is called during construction and also in response to changes to the
    /// system metrics.
    ///
    /// Bindings can override this method to change what size or device pixel
    /// ratio the [RenderView] will use. For example, the testing framework uses
    /// this to force the display into 800x600 when a test is run on the device
    /// using `flutter run`.
    ViewConfiguration createViewConfiguration() {
        final double devicePixelRatio = window.devicePixelRatio;
        return ViewConfiguration(
            size: window.physicalSize / devicePixelRatio,
            devicePixelRatio: devicePixelRatio,
        );
    }

    SemanticsHandle _semanticsHandle;

    /// Creates a [MouseTracker] which manages state about currently connected
    /// mice, for hover notification.
    ///
    /// Used by testing framework to reinitialize the mouse tracker between tests.
    @visibleForTesting
    void initMouseTracker([MouseTracker tracker]) {
        _mouseTracker?.dispose();
        _mouseTracker = tracker ?? MouseTracker(pointerRouter, renderView.hitTestMouseTrackers);
    }

    void _handleSemanticsEnabledChanged() {
        setSemanticsEnabled(window.semanticsEnabled);
    }

    /// Whether the render tree associated with this binding should produce a tree
    /// of [SemanticsNode] objects.
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

    void _handlePersistentFrameCallback(Duration timeStamp) {
        drawFrame();
        _mouseTracker.schedulePostFrameCheck();
    }

    int _firstFrameDeferredCount = 0;
    bool _firstFrameSent = false;

    /// Whether frames produced by [drawFrame] are sent to the engine.
    ///
    /// If false the framework will do all the work to produce a frame,
    /// but the frame is never sent to the engine to actually appear on screen.
    ///
    /// See also:
    ///
    ///  * [deferFirstFrame], which defers when the first frame is sent to the
    ///    engine.
    bool get sendFramesToEngine => _firstFrameSent || _firstFrameDeferredCount == 0;

    /// Tell the framework to not send the first frames to the engine until there
    /// is a corresponding call to [allowFirstFrame].
    ///
    /// Call this to perform asynchronous initialization work before the first
    /// frame is rendered (which takes down the splash screen). The framework
    /// will still do all the work to produce frames, but those frames are never
    /// sent to the engine and will not appear on screen.
    ///
    /// Calling this has no effect after the first frame has been sent to the
    /// engine.
    void deferFirstFrame() {
        assert(_firstFrameDeferredCount >= 0);
        _firstFrameDeferredCount += 1;
    }

    /// Called after [deferFirstFrame] to tell the framework that it is ok to
    /// send the first frame to the engine now.
    ///
    /// For best performance, this method should only be called while the
    /// [schedulerPhase] is [SchedulerPhase.idle].
    ///
    /// This method may only be called once for each corresponding call
    /// to [deferFirstFrame].
    void allowFirstFrame() {
        assert(_firstFrameDeferredCount > 0);
        _firstFrameDeferredCount -= 1;
        // Always schedule a warm up frame even if the deferral count is not down to
        // zero yet since the removal of a deferral may uncover new deferrals that
        // are lower in the widget tree.
        if (!_firstFrameSent)
            scheduleWarmUpFrame();
    }

    /// Call this to pretend that no frames have been sent to the engine yet.
    ///
    /// This is useful for tests that want to call [deferFirstFrame] and
    /// [allowFirstFrame] since those methods only have an effect if no frames
    /// have been sent to the engine yet.
    void resetFirstFrameSent() {
        _firstFrameSent = false;
    }

    /// Pump the rendering pipeline to generate a frame.
    ///
    /// This method is called by [handleDrawFrame], which itself is called
    /// automatically by the engine when it is time to lay out and paint a frame.
    ///
    /// Each frame consists of the following phases:
    ///
    /// 1. The animation phase: The [handleBeginFrame] method, which is registered
    /// with [Window.onBeginFrame], invokes all the transient frame callbacks
    /// registered with [scheduleFrameCallback], in registration order. This
    /// includes all the [Ticker] instances that are driving [AnimationController]
    /// objects, which means all of the active [Animation] objects tick at this
    /// point.
    ///
    /// 2. Microtasks: After [handleBeginFrame] returns, any microtasks that got
    /// scheduled by transient frame callbacks get to run. This typically includes
    /// callbacks for futures from [Ticker]s and [AnimationController]s that
    /// completed this frame.
    ///
    /// After [handleBeginFrame], [handleDrawFrame], which is registered with
    /// [Window.onDrawFrame], is called, which invokes all the persistent frame
    /// callbacks, of which the most notable is this method, [drawFrame], which
    /// proceeds as follows:
    ///
    /// 3. The layout phase: All the dirty [RenderObject]s in the system are laid
    /// out (see [RenderObject.performLayout]). See [RenderObject.markNeedsLayout]
    /// for further details on marking an object dirty for layout.
    ///
    /// 4. The compositing bits phase: The compositing bits on any dirty
    /// [RenderObject] objects are updated. See
    /// [RenderObject.markNeedsCompositingBitsUpdate].
    ///
    /// 5. The paint phase: All the dirty [RenderObject]s in the system are
    /// repainted (see [RenderObject.paint]). This generates the [Layer] tree. See
    /// [RenderObject.markNeedsPaint] for further details on marking an object
    /// dirty for paint.
    ///
    /// 6. The compositing phase: The layer tree is turned into a [Scene] and
    /// sent to the GPU.
    ///
    /// 7. The semantics phase: All the dirty [RenderObject]s in the system have
    /// their semantics updated (see [RenderObject.semanticsAnnotator]). This
    /// generates the [SemanticsNode] tree. See
    /// [RenderObject.markNeedsSemanticsUpdate] for further details on marking an
    /// object dirty for semantics.
    ///
    /// For more details on steps 3-7, see [PipelineOwner].
    ///
    /// 8. The finalization phase: After [drawFrame] returns, [handleDrawFrame]
    /// then invokes post-frame callbacks (registered with [addPostFrameCallback]).
    ///
    /// Some bindings (for example, the [WidgetsBinding]) add extra steps to this
    /// list (for example, see [WidgetsBinding.drawFrame]).
    //
    // When editing the above, also update widgets/binding.dart's copy.
    @protected
    void drawFrame() {
        assert(renderView != null);
        pipelineOwner.flushLayout();
        pipelineOwner.flushCompositingBits();
        pipelineOwner.flushPaint();
        if (sendFramesToEngine) {
            renderView.compositeFrame(); // this sends the bits to the GPU
            pipelineOwner.flushSemantics(); // this also sends the semantics to the OS.
            _firstFrameSent = true;
        }
    }

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

    @override
    void hitTest(HitTestResult result, Offset position) {
        assert(renderView != null);
        renderView.hitTest(result, position: position);
        super.hitTest(result, position);
    }

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
/// The glue between the widgets layer and the Flutter engine.
mixin WidgetsBinding on BindingBase, ServicesBinding, SchedulerBinding, GestureBinding, RendererBinding, SemanticsBinding {
    @override
    void initInstances() {
        super.initInstances();
        _instance = this;
        // Initialization of [_buildOwner] has to be done after
        // [super.initInstances] is called, as it requires [ServicesBinding] to
        // properly setup the [defaultBinaryMessenger] instance.
        _buildOwner = BuildOwner();
        buildOwner.onBuildScheduled = _handleBuildScheduled;
        window.onLocaleChanged = handleLocaleChanged;
        window.onAccessibilityFeaturesChanged = handleAccessibilityFeaturesChanged;
        SystemChannels.navigation.setMethodCallHandler(_handleNavigationInvocation);
        FlutterErrorDetails.propertiesTransformers.add(transformDebugCreator);
    }

    /// The current [WidgetsBinding], if one has been created.
    ///
    /// If you need the binding to be constructed before calling [runApp],
    /// you can ensure a Widget binding has been constructed by calling the
    /// `WidgetsFlutterBinding.ensureInitialized()` function.
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

        assert(() {
            registerBoolServiceExtension(
                name: 'debugAllowBanner',
                getter: () => Future<bool>.value(WidgetsApp.debugAllowBannerOverride),
                setter: (bool value) {
                    if (WidgetsApp.debugAllowBannerOverride == value)
                        return Future<void>.value();
                    WidgetsApp.debugAllowBannerOverride = value;
                    return _forceRebuild();
                },
            );

            // This service extension is deprecated and will be removed by 12/1/2018.
            // Use ext.flutter.inspector.show instead.
            registerBoolServiceExtension(
                name: 'debugWidgetInspector',
                getter: () async => WidgetsApp.debugShowWidgetInspectorOverride,
                setter: (bool value) {
                    if (WidgetsApp.debugShowWidgetInspectorOverride == value)
                        return Future<void>.value();
                    WidgetsApp.debugShowWidgetInspectorOverride = value;
                    return _forceRebuild();
                },
            );

            WidgetInspectorService.instance.initServiceExtensions(registerServiceExtension);

            return true;
        }());
    }

    Future<void> _forceRebuild() {
        if (renderViewElement != null) {
            buildOwner.reassemble(renderViewElement);
            return endOfFrame;
        }
        return Future<void>.value();
    }

    /// The [BuildOwner] in charge of executing the build pipeline for the
    /// widget tree rooted at this binding.
    BuildOwner get buildOwner => _buildOwner;
    // Initialization of [_buildOwner] has to be done within the [initInstances]
    // method, as it requires [ServicesBinding] to properly setup the
    // [defaultBinaryMessenger] instance.
    BuildOwner _buildOwner;

    /// The object in charge of the focus tree.
    ///
    /// Rarely used directly. Instead, consider using [FocusScope.of] to obtain
    /// the [FocusScopeNode] for a given [BuildContext].
    ///
    /// See [FocusManager] for more details.
    FocusManager get focusManager => _buildOwner.focusManager;

    final List<WidgetsBindingObserver> _observers = <WidgetsBindingObserver>[];

    /// Registers the given object as a binding observer. Binding
    /// observers are notified when various application events occur,
    /// for example when the system locale changes. Generally, one
    /// widget in the widget tree registers itself as a binding
    /// observer, and converts the system state into inherited widgets.
    ///
    /// For example, the [WidgetsApp] widget registers as a binding
    /// observer and passes the screen size to a [MediaQuery] widget
    /// each time it is built, which enables other widgets to use the
    /// [MediaQuery.of] static method and (implicitly) the
    /// [InheritedWidget] mechanism to be notified whenever the screen
    /// size changes (e.g. whenever the screen rotates).
    ///
    /// See also:
    ///
    ///  * [removeObserver], to release the resources reserved by this method.
    ///  * [WidgetsBindingObserver], which has an example of using this method.
    void addObserver(WidgetsBindingObserver observer) => _observers.add(observer);

    /// Unregisters the given observer. This should be used sparingly as
    /// it is relatively expensive (O(N) in the number of registered
    /// observers).
    ///
    /// See also:
    ///
    ///  * [addObserver], for the method that adds observers in the first place.
    ///  * [WidgetsBindingObserver], which has an example of using this method.
    bool removeObserver(WidgetsBindingObserver observer) => _observers.remove(observer);

    @override
    void handleMetricsChanged() {
        super.handleMetricsChanged();
        for (WidgetsBindingObserver observer in _observers)
            observer.didChangeMetrics();
    }

    @override
    void handleTextScaleFactorChanged() {
        super.handleTextScaleFactorChanged();
        for (WidgetsBindingObserver observer in _observers)
            observer.didChangeTextScaleFactor();
    }

    @override
    void handlePlatformBrightnessChanged() {
        super.handlePlatformBrightnessChanged();
        for (WidgetsBindingObserver observer in _observers)
            observer.didChangePlatformBrightness();
    }

    @override
    void handleAccessibilityFeaturesChanged() {
        super.handleAccessibilityFeaturesChanged();
        for (WidgetsBindingObserver observer in _observers)
            observer.didChangeAccessibilityFeatures();
    }

    /// Called when the system locale changes.
    ///
    /// Calls [dispatchLocaleChanged] to notify the binding observers.
    ///
    /// See [Window.onLocaleChanged].
    @protected
    @mustCallSuper
    void handleLocaleChanged() {
        dispatchLocalesChanged(window.locales);
    }

    /// Notify all the observers that the locale has changed (using
    /// [WidgetsBindingObserver.didChangeLocales]), giving them the
    /// `locales` argument.
    ///
    /// This is called by [handleLocaleChanged] when the [Window.onLocaleChanged]
    /// notification is received.
    @protected
    @mustCallSuper
    void dispatchLocalesChanged(List<Locale> locales) {
        for (WidgetsBindingObserver observer in _observers)
            observer.didChangeLocales(locales);
    }

    /// Notify all the observers that the active set of [AccessibilityFeatures]
    /// has changed (using [WidgetsBindingObserver.didChangeAccessibilityFeatures]),
    /// giving them the `features` argument.
    ///
    /// This is called by [handleAccessibilityFeaturesChanged] when the
    /// [Window.onAccessibilityFeaturesChanged] notification is received.
    @protected
    @mustCallSuper
    void dispatchAccessibilityFeaturesChanged() {
        for (WidgetsBindingObserver observer in _observers)
            observer.didChangeAccessibilityFeatures();
    }

    /// Called when the system pops the current route.
    ///
    /// This first notifies the binding observers (using
    /// [WidgetsBindingObserver.didPopRoute]), in registration order, until one
    /// returns true, meaning that it was able to handle the request (e.g. by
    /// closing a dialog box). If none return true, then the application is shut
    /// down by calling [SystemNavigator.pop].
    ///
    /// [WidgetsApp] uses this in conjunction with a [Navigator] to
    /// cause the back button to close dialog boxes, return from modal
    /// pages, and so forth.
    ///
    /// This method exposes the `popRoute` notification from
    /// [SystemChannels.navigation].
    @protected
    Future<void> handlePopRoute() async {
        for (WidgetsBindingObserver observer in List<WidgetsBindingObserver>.from(_observers)) {
            if (await observer.didPopRoute())
                return;
        }
        SystemNavigator.pop();
    }

    /// Called when the host tells the app to push a new route onto the
    /// navigator.
    ///
    /// This notifies the binding observers (using
    /// [WidgetsBindingObserver.didPushRoute]), in registration order, until one
    /// returns true, meaning that it was able to handle the request (e.g. by
    /// opening a dialog box). If none return true, then nothing happens.
    ///
    /// This method exposes the `pushRoute` notification from
    /// [SystemChannels.navigation].
    @protected
    @mustCallSuper
    Future<void> handlePushRoute(String route) async {
        for (WidgetsBindingObserver observer in List<WidgetsBindingObserver>.from(_observers)) {
            if (await observer.didPushRoute(route))
                return;
        }
    }

    Future<dynamic> _handleNavigationInvocation(MethodCall methodCall) {
        switch (methodCall.method) {
            case 'popRoute':
                return handlePopRoute();
            case 'pushRoute':
                return handlePushRoute(methodCall.arguments as String);
        }
        return Future<dynamic>.value();
    }

    @override
    void handleAppLifecycleStateChanged(AppLifecycleState state) {
        super.handleAppLifecycleStateChanged(state);
        for (WidgetsBindingObserver observer in _observers)
            observer.didChangeAppLifecycleState(state);
    }

    /// Called when the operating system notifies the application of a memory
    /// pressure situation.
    ///
    /// Notifies all the observers using
    /// [WidgetsBindingObserver.didHaveMemoryPressure].
    ///
    /// This method exposes the `memoryPressure` notification from
    /// [SystemChannels.system].
    void handleMemoryPressure() {
        for (WidgetsBindingObserver observer in _observers)
            observer.didHaveMemoryPressure();
    }

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

    /// Whether the first frame has finished building.
    ///
    /// This value can also be obtained over the VM service protocol as
    /// `ext.flutter.didSendFirstFrameEvent`.
    ///
    /// See also:
    ///
    ///  * [firstFrameRasterized], whether the first frame has finished rendering.
    bool get debugDidSendFirstFrameEvent => !_needToReportFirstFrame;

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

    /// When called after [deferFirstFrameReport]: tell the framework to report
    /// the frame it is building as a "useful" first frame.
    ///
    /// Deprecated. Use [deferFirstFrame]/[allowFirstFrame] to delay rendering the
    /// first frame.
    @Deprecated(
        'Use deferFirstFrame/allowFirstFrame to delay rendering the first frame. '
        'This feature was deprecated after v1.12.4.'
    )
    void allowFirstFrameReport() {
        if (!kReleaseMode) {
            allowFirstFrame();
        }
    }

    void _handleBuildScheduled() {
        // If we're in the process of building dirty elements, then changes
        // should not trigger a new frame.
        assert(() {
            if (debugBuildingDirtyElements) {
                throw FlutterError.fromParts(<DiagnosticsNode>[
                    ErrorSummary('Build scheduled during frame.'),
                    ErrorDescription(
                        'While the widget tree was being built, laid out, and painted, '
                        'a new frame was scheduled to rebuild the widget tree.'
                    ),
                    ErrorHint(
                        'This might be because setState() was called from a layout or '
                        'paint callback. '
                        'If a change is needed to the widget tree, it should be applied '
                        'as the tree is being built. Scheduling a change for the subsequent '
                        'frame instead results in an interface that lags behind by one frame. '
                        'If this was done to make your build dependent on a size measured at '
                        'layout time, consider using a LayoutBuilder, CustomSingleChildLayout, '
                        'or CustomMultiChildLayout. If, on the other hand, the one frame delay '
                        'is the desired effect, for example because this is an '
                        'animation, consider scheduling the frame in a post-frame callback '
                        'using SchedulerBinding.addPostFrameCallback or '
                        'using an AnimationController to trigger the animation.',
                    )
                ]);
            }
            return true;
        }());
        ensureVisualUpdate();
    }

    /// Whether we are currently in a frame. This is used to verify
    /// that frames are not scheduled redundantly.
    ///
    /// This is public so that test frameworks can change it.
    ///
    /// This flag is not used in release builds.
    @protected
    bool debugBuildingDirtyElements = false;

    /// Pump the build and rendering pipeline to generate a frame.
    ///
    /// This method is called by [handleDrawFrame], which itself is called
    /// automatically by the engine when when it is time to lay out and paint a
    /// frame.
    ///
    /// Each frame consists of the following phases:
    ///
    /// 1. The animation phase: The [handleBeginFrame] method, which is registered
    /// with [Window.onBeginFrame], invokes all the transient frame callbacks
    /// registered with [scheduleFrameCallback], in
    /// registration order. This includes all the [Ticker] instances that are
    /// driving [AnimationController] objects, which means all of the active
    /// [Animation] objects tick at this point.
    ///
    /// 2. Microtasks: After [handleBeginFrame] returns, any microtasks that got
    /// scheduled by transient frame callbacks get to run. This typically includes
    /// callbacks for futures from [Ticker]s and [AnimationController]s that
    /// completed this frame.
    ///
    /// After [handleBeginFrame], [handleDrawFrame], which is registered with
    /// [Window.onDrawFrame], is called, which invokes all the persistent frame
    /// callbacks, of which the most notable is this method, [drawFrame], which
    /// proceeds as follows:
    ///
    /// 3. The build phase: All the dirty [Element]s in the widget tree are
    /// rebuilt (see [State.build]). See [State.setState] for further details on
    /// marking a widget dirty for building. See [BuildOwner] for more information
    /// on this step.
    ///
    /// 4. The layout phase: All the dirty [RenderObject]s in the system are laid
    /// out (see [RenderObject.performLayout]). See [RenderObject.markNeedsLayout]
    /// for further details on marking an object dirty for layout.
    ///
    /// 5. The compositing bits phase: The compositing bits on any dirty
    /// [RenderObject] objects are updated. See
    /// [RenderObject.markNeedsCompositingBitsUpdate].
    ///
    /// 6. The paint phase: All the dirty [RenderObject]s in the system are
    /// repainted (see [RenderObject.paint]). This generates the [Layer] tree. See
    /// [RenderObject.markNeedsPaint] for further details on marking an object
    /// dirty for paint.
    ///
    /// 7. The compositing phase: The layer tree is turned into a [Scene] and
    /// sent to the GPU.
    ///
    /// 8. The semantics phase: All the dirty [RenderObject]s in the system have
    /// their semantics updated (see [RenderObject.assembleSemanticsNode]). This
    /// generates the [SemanticsNode] tree. See
    /// [RenderObject.markNeedsSemanticsUpdate] for further details on marking an
    /// object dirty for semantics.
    ///
    /// For more details on steps 4-8, see [PipelineOwner].
    ///
    /// 9. The finalization phase in the widgets layer: The widgets tree is
    /// finalized. This causes [State.dispose] to be invoked on any objects that
    /// were removed from the widgets tree this frame. See
    /// [BuildOwner.finalizeTree] for more details.
    ///
    /// 10. The finalization phase in the scheduler layer: After [drawFrame]
    /// returns, [handleDrawFrame] then invokes post-frame callbacks (registered
    /// with [addPostFrameCallback]).
    //
    // When editing the above, also update rendering/binding.dart's copy.
    @override
    void drawFrame() {
        assert(!debugBuildingDirtyElements);
        assert(() {
            debugBuildingDirtyElements = true;
            return true;
        }());

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
                buildOwner.buildScope(renderViewElement);
            super.drawFrame();
            buildOwner.finalizeTree();
        } finally {
            assert(() {
                debugBuildingDirtyElements = false;
                return true;
            }());
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

    /// The [Element] that is at the root of the hierarchy (and which wraps the
    /// [RenderView] object at the root of the rendering hierarchy).
    ///
    /// This is initialized the first time [runApp] is called.
    Element get renderViewElement => _renderViewElement;
    Element _renderViewElement;

    /// Schedules a [Timer] for attaching the root widget.
    ///
    /// This is called by [runApp] to configure the widget tree. Consider using
    /// [attachRootWidget] if you want to build the widget tree synchronously.
    @protected
    void scheduleAttachRootWidget(Widget rootWidget) {
        Timer.run(() {
            attachRootWidget(rootWidget);
        });
    }

    /// Takes a widget and attaches it to the [renderViewElement], creating it if
    /// necessary.
    ///
    /// This is called by [runApp] to configure the widget tree.
    ///
    /// See also:
    ///
    ///  * [RenderObjectToWidgetAdapter.attachToRenderTree], which inflates a
    ///    widget and attaches it to the render tree.
    void attachRootWidget(Widget rootWidget) {
        _renderViewElement = RenderObjectToWidgetAdapter<RenderBox>(
            container: renderView,
            debugShortDescription: '[root]',
            child: rootWidget,
        ).attachToRenderTree(buildOwner, renderViewElement as RenderObjectToWidgetElement<RenderBox>);
    }

    /// Whether the [renderViewElement] has been initialized.
    ///
    /// This will be false until [runApp] is called (or [WidgetTester.pumpWidget]
    /// is called in the context of a [TestWidgetsFlutterBinding]).
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

