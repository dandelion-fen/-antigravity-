# Antigravity 登录时报 Google API 连接超时的解决方法

## 一、问题现象

在 Windows 上使用 Antigravity 登录 Google 账号时，点击登录后会跳转到浏览器进行 Google 授权。浏览器授权流程本身可以正常打开，但回到 Antigravity 客户端后，页面提示：

![image-20260606130044966](C:\Users\27395\AppData\Roaming\Typora\typora-user-images\image-20260606130044966.png)

```text
There was an unexpected issue setting up your account.
```

具体报错可能是：

```text
Post "https://oauth2.googleapis.com/token": dial tcp xxx.xxx.xxx.xxx:443: connectex:
A connection attempt failed because the connected party did not properly respond...
```

或者：

```text
Post "https://cloudcode-pa.googleapis.com/v1internal:loadCodeAssist": dial tcp xxx.xxx.xxx.xxx:443: connectex:
A connection attempt failed because the connected party did not properly respond...
```

从报错可以看出，Antigravity 客户端在访问 Google 的 OAuth 或 Cloud Code Assist 接口时发生了超时。

需要注意的是，浏览器能正常访问 Google，并不代表 Antigravity 客户端也能正常走代理。这个问题的关键不在浏览器，而在 Antigravity 客户端本身的网络请求没有被代理接管。

## 二、问题原因

本次使用的代理工具是 Clash Verge，端口为：

```text
7897
```

最开始只开启了：

```text
系统代理
```

但是 Antigravity 并没有稳定走系统代理，仍然可能直接访问：

```text
oauth2.googleapis.com
cloudcode-pa.googleapis.com
```

因此虽然浏览器端授权页面可以打开，但 Antigravity 客户端回调后继续请求 Google API 时仍然超时。

解决思路是：

```text
让 Antigravity 客户端本身的流量也被代理接管
```

主要方法是开启 Clash Verge 的虚拟网卡模式，也就是 TUN 模式。

## 三、完整解决步骤

### 1. 确认 Clash Verge 端口

在 Clash Verge 设置中确认端口设置。本次端口为：

```text
7897
```

后续所有代理地址都使用：

```text
http://127.0.0.1:7897
```

如果你的端口不是 7897，需要改成自己的实际端口。

### 2. 开启系统代理

在 Clash Verge 设置中开启：

```text
系统代理
```

同时确认：

```text
端口设置：7897
```

如果只是浏览器访问 Google，这一步通常已经够了。但对于 Antigravity，这一步还不一定够。

### 3. 开启虚拟网卡模式，也就是 TUN 模式

进入 Clash Verge 的虚拟网卡模式设置，推荐配置如下：

```text
TUN 模式堆栈：GVisor
虚拟网卡名称：Meta
自动设置全局路由：开启
严格路由：关闭
自动选择流量出口接口：开启
DNS 劫持：any:53
最大传输单元：9000
```

设置完成后点击保存。

然后回到 Clash Verge 主设置页面，真正打开：

```text
虚拟网卡模式
```

如果旁边出现黄色感叹号，可以点击扳手图标修复服务。过程中如果弹出管理员权限提示，选择允许。

这一步是关键。只开系统代理时，Antigravity 可能仍然直连；开启 TUN 后，客户端流量会更容易被代理接管。

### 4. 建议临时关闭 IPv6

为了避免部分程序通过 IPv6 绕过代理，建议测试阶段先关闭 Clash Verge 中的：

```text
IPv6
```

后续确认 Antigravity 可以正常登录后，再根据需要决定是否重新开启。

### 5. 给 Antigravity 写入代理配置

在 PowerShell 中打开 Antigravity 的用户配置文件：

```powershell
notepad "C:\Users\27395\AppData\Roaming\Antigravity\User\settings.json"
```

如果提示是否创建新文件，选择“是”。

写入以下内容：

```json
{
  "http.proxy": "http://127.0.0.1:7897",
  "http.proxySupport": "override",
  "http.proxyStrictSSL": false
}
```

如果文件里原本已经有其他配置，不要重复写最外层的 `{}`，只需要把这三项加入原来的 JSON 中，并注意英文逗号。

### 6. 结束 Antigravity 进程

修改配置后，需要彻底关闭 Antigravity。可以在 PowerShell 中执行：

```powershell
taskkill /F /IM Antigravity.exe
taskkill /F /IM node.exe
```

如果提示没有找到 `node.exe`，可以忽略。

### 7. 测试 Google API 是否能访问

在 PowerShell 中测试代理是否能访问相关接口：

```powershell
curl.exe -x http://127.0.0.1:7897 -I https://oauth2.googleapis.com/token
```

再测试：

```powershell
curl.exe -x http://127.0.0.1:7897 -I https://cloudcode-pa.googleapis.com
```

这里不要求返回 `200`。只要不是 timeout，即使返回：

```text
403
404
405
```

也说明网络已经能连通。

开启 TUN 后，也可以测试不手动指定代理的情况：

```powershell
curl.exe -I https://cloudcode-pa.googleapis.com
```

如果这条也不再 timeout，说明虚拟网卡模式已经接管成功。

### 8. 重新登录 Antigravity

完成以上步骤后，重新打开 Antigravity。

正常流程如下：

```text
打开 Antigravity
点击登录
跳转浏览器进行 Google 授权
浏览器询问是否打开 Antigravity
点击允许
回到 Antigravity
```

如果此时不再出现：

```text
oauth2.googleapis.com timeout
cloudcode-pa.googleapis.com timeout
```

说明网络代理问题已经解决。

## 四、后续可能出现的新问题

如果网络问题解决后，出现如下提示：

![image-20260606130035888](C:\Users\27395\AppData\Roaming\Typora\typora-user-images\image-20260606130035888.png)

```text
Sorry, this account is ineligible to use Antigravity
Authentication failed.
```

这就不是代理问题了，而是账号资格问题。

这个提示说明 Antigravity 已经能够连接到 Google 后台并完成账号校验，只是当前 Google 账号不符合 Antigravity 使用条件。

常见原因包括：

```text
使用的是学校或企业 Workspace 账号
账号年龄或地区不符合要求
账号本身没有 Antigravity 使用资格
授权状态异常
```

这类问题需要换用个人 Gmail 账号，或者检查 Google 账号地区、年龄验证和第三方授权状态。

相关解决方法课查看链接：[(4 封私信 / 32 条消息) 解决google antigravity 登陆不上，账号区域限制问题 - 知乎](https://zhuanlan.zhihu.com/p/1974841556331696572)

## 五、总结

Antigravity 登录时出现：

```text
oauth2.googleapis.com timeout
cloudcode-pa.googleapis.com timeout
```

核心原因通常不是浏览器不能访问 Google，而是 Antigravity 客户端自己的网络请求没有走代理。

本次最终有效的解决方法是：

```text
Clash Verge 端口 7897
开启系统代理
开启虚拟网卡模式 / TUN 模式
关闭 IPv6 测试
写入 Antigravity 的 http.proxy 配置
杀掉 Antigravity 进程后重新登录
```

其中最关键的是：

```text
开启虚拟网卡模式 / TUN 模式
```

因为 Antigravity 可能不跟随普通系统代理，只有通过 TUN 接管网络流量后，才能避免客户端直连 Google API 导致超时。