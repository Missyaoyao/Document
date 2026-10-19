## 手动打开浏览器登录后，playwright继续操作
### playwright 使用 cdp 方式自动化脚本接管手工打开的 chrome 页面继续操作

背景： 有些网站的登录流程比较复杂，或者安全级别比较高，或者需要验证码、短信验证等，这些操作无法通过自动化脚本完成。此时可以使用 Playwright 的持久化上下文功能，手动登录一次网站，然后在后续的自动化测试中继续操作。


**1、 环境准备**

将chrome配置系统环境变量，确保可以在命令行中直接使用chrome命令启动浏览器。

**2、原理**
> 原理：Chrome 启动时开启`--remote-debugging-port=9222`，暴露 CDP 调试端口；Playwright 用`connect_over_cdp`接入这个**已经跑起来的 Chrome 实例**，直接复用当前所有 tab、Cookie、登录状态、localStorage。
> 
> ✅ 重点：**不能直接双击打开 Chrome**，必须命令行带参数启动 Chrome，这个 Chrome 才会开放 CDP 端口，Playwright 才能接管。

#### 步骤 1：命令行启动 Chrome（带远程调试端口，Windows）

> 
> ⚠️ 先**全部关闭所有 Chrome 窗口 + 后台 chrome 进程**，否则会冲突！

```
# Windows cmd 执行
"chrome.exe --remote-debugging-port=12345 --user-data-dir="E:\Work\user_data_chrome"
```

参数说明：

1. `--remote-debugging-port=12345`：开放 CDP 端口（固定 123456）
2. `--user-data-dir="E:\Work\user_data_chrome"`：独立用户目录（保存你的登录缓存，**不要直接使用默认 Default 目录，会锁**）

执行完命令 → Chrome 自动弹出窗口，**你手动在这个 Chrome 里登录网站、打开页面**，之后脚本接管这个浏览器继续操作。

> 
> 验证是否成功开启 CDP：浏览器访问 `http://127.0.0.1:12345`，能看到调试页面，代表端口正常。

#### 步骤 2：Python 脚本 connect_over_cdp 接管 Chrome

**3、 代码实现**

    
    ```python
    from playwright.sync_api import sync_playwright

    with sync_playwright() as p:
        browser = p.chromium.connect_over_cdp('http://localhost:12345/')
        # 获取page对象
        page = browser.contexts[0].pages[0]

    print(page.url)
    print(page.title())

    page.locator("#right-top").get_by_role("link", name="查看报告").click()
    print(page.url)
    print(page.title())


**注意： 通过cmd执行命令后打开的浏览器不要关闭，playwright 就是接管这个浏览器继续操作。**
