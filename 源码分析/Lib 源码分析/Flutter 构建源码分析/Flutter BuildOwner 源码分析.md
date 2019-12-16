## Flutter BuildOwner 源码分析

#### InActiveElements

```dart
/// builderOwner._inactiveElements是一个类
/// class BuildOwner {
///    ...
///    final _InactiveElements _inactiveElements = _InactiveElements();
///    ...
///	}

class _InactiveElements {
  bool _locked = false;
  final Set<Element> _elements = HashSet<Element>();

  /// unmount一个element会unmount其所有的孩子  
  void _unmount(Element element) {
    element.visitChildren((Element child) {
      _unmount(child);
    });
    element.unmount();
  }

  /// unmount _elements列表（inactive的element列表）中的所有元素  
  void _unmountAll() {
    _locked = true;
    final List<Element> elements = _elements.toList()..sort(Element._sort);
    _elements.clear();
    try {
      elements.reversed.forEach(_unmount);
    } finally {
      _locked = false;
    }
  }

  /// 递归地 deactivate 孩子   
  static void _deactivateRecursively(Element element) {
    element.deactivate();
    element.visitChildren(_deactivateRecursively);
  }

  /// 并将自身加入到 _elements 中 
  void add(Element element) {
    if (element._active)
      _deactivateRecursively(element);
    _elements.add(element);
  }

  void remove(Element element) {
    _elements.remove(element);
  }
}
```



## BuildOwner

```dart

/// 用于管理 Widget 框架的管理者
///
/// 该类跟踪哪些 widget 需要重新构建，并处理应用于整个 widget 树的其他任务，
/// 例如管理树的非活动元素列表，并在调试时在热重新加载期间触发“reassemble”命令。
/// 
/// buildOwner 通常由[WidgetsBinding]拥有，并与构建/布局/绘图流水线的其他部分一起被操作系统驱动。
///
/// 可以用额外的 buildOwner 来管理离开屏幕的 widget
///
/// 要将 buildOwner 分配给树，在根element 上调用 [RootRenderObjectElement.assignOwner]方法
class BuildOwner {
  BuildOwner({ this.onBuildScheduled });

  /// 当第一个可构建 element 被标记为dirty时，都属调用这个回调。
  VoidCallback onBuildScheduled;

  final _InactiveElements _inactiveElements = _InactiveElements();

  final List<Element> _dirtyElements = <Element>[];
  bool _scheduledFlushDirtyElements = false;

  /// 是否需要重新排序[_dirtyElements]，因为在构建过程中有更多的元素变得脏了。
  ///
  /// 这对于保持[Element._sort]定义的排序顺序是必要的。
  ///
  /// 当[buildScope]不主动重建小部件树时，该字段被设置为null。
  bool _dirtyElementsNeedsResorting;

  /// 负责焦点树的对象。
  ///
  /// 很少直接被使用。相反，可以考虑使用[FocusScope]。获取给定[BuildContext]的[FocusScopeNode]。
  FocusManager focusManager = FocusManager();

  /// 将元素添加到脏元素列表中，以便在[WidgetsBinding.drawFrame]调用[buildScope]时重新构建它
  void scheduleBuildFor(Element element) {
    /// 如果试图添加一个已经处于列表中的 element，这说明那个元素需要再次被重建。处理这个情况的方法是将脏
    /// 节点列表重新排序
    if (element._inDirtyList) {
      _dirtyElementsNeedsResorting = true;
      return;
    }
    /// 调用构造对象是传入的回调，通常的回调是向引擎请求一帧  
    if (!_scheduledFlushDirtyElements && onBuildScheduled != null) {
      _scheduledFlushDirtyElements = true;
      onBuildScheduled();
    }
    /// 把节点添加到列表当中，在下一帧的垂直同步信号到来的时候，会遍历整个列表，对其中的每一个 element调
    /// 用其 rebuild 方法，由此可以重构整颗树  
    _dirtyElements.add(element);
    element._inDirtyList = true;
  }

  /// 建立一个调用[State]的区域。禁用[setState]，并调用给定的“回调”。
  ///
  /// 该机制用于确保，例如，[State]。dispose]不调用[State.setState]。
  /// 这个方法利用了断言实现了类似于锁存的机制 
  void lockState(void callback()) {
    assert(callback != null);
    assert(_debugStateLockLevel >= 0);
    assert(() {
      _debugStateLockLevel += 1;
      return true;
    }());
    try {
      callback();
    } finally {
      assert(() {
        _debugStateLockLevel -= 1;
        return true;
      }());
    }
    assert(_debugStateLockLevel >= 0);
  }

  /// 建立一个用于更新小部件树的区域，并调用给定的“回调”(如果有的话)。
  /// 然后，按深度顺序重新构建所有被[scheduleBuildFor]标记为脏的元素。
  ///
  /// 这种机制可以防止构建方法过渡地要求其他构建方法运行，从而可能导致无限循环。
  ///
  /// 脏列表在'callback'返回后处理，使用 [scheduleBuildFor] 按深度顺序构建所有标记为 dirty 的元素。
  /// 如果元素在此方法运行时被标记为 dirty，则它们必须比“context”节点更深，并且比此遍历中已经构建完成的
  /// 任何节点更深。（这意味着，在一次构建中，如果子节点再次被标记dirty的话，只能是当前构建节点的下层节
  /// 点。这会改变 脏节点列表的长度，或者是修改需要重排序的标记为true）
  /// 
  /// 要在不执行任何其他工作的情况下刷新当前脏节点列表，可以不使用回调调用此函数。这就是框架在
  /// [WidgetsBinding.drawFrame]中的每一帧所做的事情。
  /// 
  /// 一次只能激活一个[buildScope]。
  void buildScope(Element context, [ VoidCallback callback ]) {
    if (callback == null && _dirtyElements.isEmpty)
      return;
    Timeline.startSync('Build', arguments: timelineWhitelistArguments);
    try {
      _scheduledFlushDirtyElements = true;
      if (callback != null) {
        Element debugPreviousBuildTarget;
        _dirtyElementsNeedsResorting = false;
        try {
          callback();
        } finally {
        }
      }
      _dirtyElements.sort(Element._sort);
      _dirtyElementsNeedsResorting = false;
      int dirtyCount = _dirtyElements.length;
      int index = 0;
      while (index < dirtyCount) {
        try {
          _dirtyElements[index].rebuild();
        } catch (e, stack) {
        }
        index += 1;
        if (dirtyCount < _dirtyElements.length || _dirtyElementsNeedsResorting) {
          _dirtyElements.sort(Element._sort);
          _dirtyElementsNeedsResorting = false;
          dirtyCount = _dirtyElements.length;
          while (index > 0 && _dirtyElements[index - 1].dirty) {
            // 之前标记为 dirty 但是 inactive 的 widget 可以在列表中右移。
            // 因此，我们不得不把索引移到列表的左边来解决这个问题。
            // 我们不知道会有多少widget会移动。
            // 然而，我们知道，唯一的可能时是先前在左边的节点移动到了已清理的节点之后。
            index -= 1;
          }
        }
      }
      /// 这时候已经清理了所有的脏节点，然后需要清除 _inDirtyList 标记 
    } finally {
      for (Element element in _dirtyElements) {
        element._inDirtyList = false;
      }
      _dirtyElements.clear();
      _scheduledFlushDirtyElements = false;
      _dirtyElementsNeedsResorting = null;
    }
  }

  /// 通过卸载任何不再活动的 element 来完成 element 构建阶段。
  ///
  /// 被 [WidgetsBinding.drawFrame] 所调用
  ///
  ///
  /// 在当前调用堆栈展开后，将运行一个微任务，通知侦听器有关全局键的更改。
  void finalizeTree() {
    try {
      lockState(() {
        _inactiveElements._unmountAll(); // 这会注销所有的 element 中的 GlobalKey
      });
    } catch (e, stack) {
    } finally {
    }
  }

  /// 导致给定[element]的整个子树被完全重建。当应用程序代码更改并正在热加载时，开发工具将使用它，以使小
  /// widget 树立刻对更改做出响应并呈现出对应的效果
  ///
  /// 这个方法十分昂贵，除了在开发期间，不要调用它
  void reassemble(Element root) {
    try {
      root.reassemble();
    } finally {
    }
  }
}
```

