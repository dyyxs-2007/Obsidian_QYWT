# iOS 沙盒目录结构


## 核心概念

### 沙盒常见目录及用途
- **Bundle Container (`*.app`)**：应用可执行文件与只读资源文件。
- **Data Container**
  - `Documents/`：用户生成的重要数据，会被 iTunes/iCloud 备份。
  - `Library/Preferences/`：`UserDefaults` 对应的 plist 存储位置，会被备份。
  - `Library/Caches/`：缓存数据（如图片、音频缓存），不会被备份，系统空间不足时可能被清理。
  - `tmp/`：临时文件，不会被备份，应用退出或系统清理时可能被删除。

---

## 在本项目中的实际应用
- 源码位置：
- 缓存目录选择与文件管理方式：
- 清理/容量控制策略：

---

## 参考资料
- 
