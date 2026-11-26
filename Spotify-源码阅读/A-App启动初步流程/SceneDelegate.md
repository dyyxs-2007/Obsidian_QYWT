Share方法不是说它是个单例，而是让其他地方可以直接通过SceneDelegate类获取当前SceneDelegate
这样方便切换控制器



主要方法中用到的SPTheme
这个类提供一些类方法给背景等内容初始化，在SPTheme内部，其实是先调用了UIColor的一个动态染色方法，SPTheme会提供深浅色，以后切换黑夜白天自动渲染


 **BOOL** isSelfTest = [NSProcessInfo.processInfo.arguments containsObject:@"-SPSelfTest"];
这里的 NSProcessInfo 是获取系统进程，拿到里面的由开发时传入的参数确定的是否测试

接着判断情况，进入新手引导或者登录注册或者首页

最后一个if处理是为了等动画过去再开始启动测试流程