# playwright 如何启动 chrome 浏览器加载本地缓存实现免登录

# ✅ Playwright 免登录两种方案

> 
> 官方**不推荐直接挂载你日常 Chrome 的 Default 文件夹**（新版 Chrome 有锁、隔离机制，就是你刚才踩的坑）
> 最佳思路：**新建独立 profile 文件夹，用 playwright 自带 Chromium，打开浏览器手动登录一次，之后每次启动都会保留登录态**

## 方案 A：持久化上下文 launch_persistent_context（你想要的：本地缓存免登录，推荐）

原理：`launch_persistent_context(user_data_dir=xxx)` 会把 Cookie、LocalStorage、IndexedDB 全部保存在你指定的文件夹，关闭浏览器不会丢失，下次启动自动加载。

> 
> `user_data_dir` 传的是**整个 profile 目录**，不要混淆 Chrome 原生 User Data 和 playwright profile。

### 完整 Python 同步代码

```
from playwright.sync_api import sync_playwright

# 自定义profile目录（新建空文件夹，不要用本机Chrome的Default！）
PROFILE_FOLDER = r"./my_chrome_profile"
TARGET_URL = "https://xxx.com"

with sync_playwright() as p:
    context = p.chromium.launch_persistent_context(
        user_data_dir=PROFILE_FOLDER,
        headless=False,
        # channel="chrome", # 不要开启！使用playwright自带Chromium，稳定性更好
        args=[
            "--no-first-run",
            "--no-default-browser-check",
        ]
    )
    page = context.pages[0]
    page.goto(TARGET_URL, wait_until="networkidle", timeout=60000)

    # 第一次运行：浏览器打开，手动登录网站，登录完成按回车
    # 第二次及以后运行：自动加载缓存，直接免登录
    input("登录完成后按回车关闭浏览器 >>")

    context.close()
```

### 使用步骤

1. 第一次运行代码 → 浏览器弹出，**手动登录你的网站** 
2. 登录成功后，按回车关闭浏览器（正常关闭，不要强制杀进程）
3. **再次运行代码**：浏览器打开，加载 `./my_chrome_profile` 里面全部缓存，直接免登录访问目标网站

注意：是在第一次执行脚本时手动登录，之后第二次执行脚本就可以直接免登录访问了。
> 
> 重点：
> 
> 
> - 不要复用电脑 Chrome 的 Default 文件夹，Chrome 会加锁，读取不到登录态
> - 同一个 profile 文件夹**不能同时多个浏览器打开**，会文件锁
> - 如果登录失效：删除 `my_chrome_profile` 文件夹，重新登录一次


代码示例
```python
from playwright.sync_api import sync_playwright
import getpass

print(f'当前用户名：{getpass.getuser()}')
# USER_DIR_PATH = f'C:\\Users\\{getpass.getuser()}\\AppData\\Local\\Google\\Chrome\\User Data\\Default'
# 复制上述路径到本地，创建一个空文件夹（即E:\\Work\\chrome_profile目录下，把刚才复制的内容粘贴过来），作为新的 profile 文件夹

USER_DIR_PATH = f'E:\\Work\\chrome_profile'

with sync_playwright() as p:
    context = p.chromium.launch_persistent_context(user_data_dir=USER_DIR_PATH,
                                                   headless=False, slow_mo=1000,
                                                   # channel="chrome"
                                                   args=["--no-first-run",
                                                         "--no-default-browser-check",
                                                         "--start-maximized"]
                                                   )
    page = context.pages[0]
    page.goto('http://47.116.12.183/index.html', wait_until="networkidle",
              timeout=60000)
    print(page.title())
    print("cookie数量：", len(context.cookies()))
    input("登录完成后按回车键继续...")

    context.close()
```

