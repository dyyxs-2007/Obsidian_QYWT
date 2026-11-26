# Keychain 与 Plist 安全存储

## 核心概念与对比

| 维度 | Keychain（钥匙串） | Plist / UserDefaults |
| :--- | :--- | :--- |
| **安全性** | 系统级加密存储，支持硬件/Secure Enclave 保护 | 明文存储于沙盒 `Library/Preferences` 或自定义路径 |
| **生命周期** | 卸载 App 后数据默认保留（受系统策略约束） | 随 App 卸载一同删除 |
| **适用场景** | Token、密码、Cookie、私钥等敏感凭证 | 非敏感的用户偏好设置、功能开关、轻量状态 |

---

## 在本项目中的实际应用
- Keychain 封装与使用位置：
- Plist / UserDefaults 使用位置：
- 存储内容分类与注意事项：

---

## 参考资料
- 
