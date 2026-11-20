# 微信 4.1.15 适配说明

> 本文件记录本仓库在 **Windows 11 + 微信 PC 版 4.1.15** 环境下的实际部署经验与踩坑。
> 上游 README 面向的是微信 4.0，直接照搬会失败，原因见下。

---

## 一、上游 `find_all_keys.py` 在 4.1.15 上失效

实测在本机运行上游脚本：

```
找到 20 个数据库, 20 个不同的salt
[+] Weixin.exe PID=22440 (415MB)
[+] 可读内存: 38 区域, 1200MB
扫描 x'<hex>' 缓存密钥...
结果: 0/20 salts 找到密钥
```

**2 秒扫完，一个 salt 都没匹配上。**

原因：上游用的是**扫描内存中的 `x'<64hex_enc_key><32hex_salt>'` 明文字符串**。
而微信从 **4.1.11 起不再以明文形式缓存这个字符串**，内存里只保留一个 WCDB passphrase 对象。
正则匹配不到任何东西，于是直接空手而归。

> 上游 README 里也提到过这一点：微信 4.1.11+ 需要 hook 才能拿到，
> 但 hook 意味着注入，与本项目的「纯只读」原则冲突。

## 二、可用的替代路径：Config.Cipher 对象链

不注入、不 hook、不修改微信进程任何字节，纯只读扫描：

```
1. 在进程内存中找到字符串 "com.Tencent.WCDB.Config.Cipher"
2. 找到引用它的 std::string 节点（+0x10 指针、+0x18 长度）
3. 从节点 +0x28 取 config 对象指针
4. config+0x88 是 {data_ptr, data_len, ...} 结构，取出 blob
5. blob 用 27 字节循环 XOR 掩码解码 → 得到 x'<hex>' 形式的密钥串
6. 从中切出 64 位 hex 作为 enc_key 候选
7. 用数据库第一页的 HMAC-SHA512 校验判定真伪
```

实测结果：**20/20 salt 全部命中，耗时 5 秒**。

> 第 7 步的 HMAC 校验是**唯一可靠的判定方式**。
> password 和 enc_key 都是 32 字节，长度上完全无法区分，
> 只能靠「拿数据库首页的 HMAC 钢印去比对，对上了才算数」。

## 三、两种输出格式完全兼容

上游与本适配方案输出的 JSON 结构一致：

```json
{
  "<db 相对路径>": {
    "enc_key": "<64 位 hex>",
    "salt": "<32 位 hex>"
  }
}
```

因此**密钥文件可以直接互换**，后续解密链路（`decrypt_db.py` / `monitor.py` / `monitor_web.py` / `mcp_server.py`）**无需任何改动**。

## 四、部署清单

| 文件 | 说明 | 是否入库 |
|---|---|---|
| `all_keys.json` | 密钥映射表，由适配方案产出 | ❌ 已 gitignore |
| `config.json` | `db_dir` 指向本机 `db_storage` 路径 | ❌ 已 gitignore |
| `decrypted/` | 全量解密产物（明文聊天内容） | ❌ 已 gitignore |
| `decoded_images/` | 解码后的图片 | ❌ 已 gitignore |

## 五、密钥会过期 —— salt 变了就得重提

微信重建数据库时 salt 会变化，**旧密钥立刻失效**。判断方法：

```python
page1 = open(db_path, 'rb').read(4096)
print(page1[:16].hex() == keys[rel]['salt'])   # False → 需重新提取
```

本机实测：10 月 30 日提取的密钥，到 11 月 20 日 salt 已从 `87d9…` 变为 `21e8…`。
重新提取一次即可，**10~30 秒，无持续成本**。

## 六、密钥相关的密码学要点

```
enc_key = PBKDF2-HMAC-SHA512(password, salt, 256000, 32)   ← password 不等于 enc_key
mac_key = PBKDF2-HMAC-SHA512(enc_key, salt^0x3A, 2, 32)
```

拿到 32 字节 `password` 直接当 AES 密钥用，必然全页解密失败 —— 这是最大的坑。

## 七、安全须知

- `all_keys.json` 等价于**你全部聊天记录的解密权限**
- `decrypted/` 是**明文聊天内容**
- 两者都已在 `.gitignore` 中，**切勿提交或外传**
- 所有密钥提取工具仅限本人设备上的本人账号使用
