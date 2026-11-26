**SPMusicRepository**
**SPAuthService**
从NSUserDefault里面拿到存的数据源和用户信息，区分默认账户，默认音源

上面这俩类是单例
 [SPMusicRepository shared].source = [SPMockMusicSource new];
 这句先初始化单例，然后注册依赖
 所以说
 