# H5_cursor_cowork
可以，用“本地 RPA 脚本 + 你的 Cowork 后端”把这一步全自动掉，不用你手动粘贴。

我总结成一句话：  
**H5 发指令 → 后端蒸馏并保存 skill → 本地 RPA 脚本轮询后端 → 自动操作 Cursor 聊天框 + 上传按钮 + 一键同意。**

### 1. 整体桥接方案（不再手动粘贴）

在你机器上跑一个常驻脚本（RPA Agent），做三件事：

1. 定时请求你的后端：  
   `GET /api/next_task_for_cursor`  
   拿到：
   - `clipboard_text`（要粘进聊天框的内容）
   - `skill_file_path`（本地已保存的 JSON/YAML 或其他附件）
2. 自动把 `clipboard_text` 粘到 **Cursor 当前对话的输入框**  
3. 自动点击：
   - “上传按钮”（打开文件选择框，选中 `skill_file_path`）  
   - “发送 / 回车”  
   - “一键全部同意”（Apply all，对应快捷键或按钮）

---

### 2. 用什么做 RPA 比较现实（Windows）

你现在是 Win11，可以选这些开源/免费方案：

- **AutoHotkey**（推荐，适合你这种“锁定一个应用+点击固定按钮”）
- **pywinauto + pyautogui**（Python 版 RPA，方便和你现有 Python 后端衔接）
- 较重的如 TagUI / SikuliX 就先不建议了

比较稳妥的组合：  
**后端还是 Python → RPA 层用 AutoHotkey 或 pywinauto**。

---

### 3. RPA 脚本核心逻辑（伪代码）

#### 3.1 Python 守护进程（轮询后端）

```python
import time
import requests
import json
import subprocess

BACKEND = "http://127.0.0.1:8000"

def fetch_next_task():
    r = requests.get(f"{BACKEND}/api/next_task_for_cursor")
    if r.status_code != 200:
        return None
    data = r.json()
    if not data.get("has_task"):
        return None
    return data

def save_clipboard_text(text: str):
    import pyperclip
    pyperclip.copy(text)

def trigger_rpa(clipboard_text: str, skill_path: str):
    # 这里有两种做法：
    # 1. 直接在 Python 里用 pywinauto/pyautogui 操作 Cursor
    # 2. 调用一个 AutoHotkey 脚本，把参数传过去
    save_clipboard_text(clipboard_text)
    subprocess.Popen(["autohotkey.exe", "cursor_rpa.ahk", skill_path])

def main_loop():
    while True:
        task = fetch_next_task()
        if task:
            clipboard_text = task["clipboard_text"]
            skill_path = task["skill_file_path"]
            trigger_rpa(clipboard_text, skill_path)
        time.sleep(3)

if __name__ == "__main__":
    main_loop()
```

#### 3.2 AutoHotkey 伪脚本（锁定 Cursor 聊天框 + 上传按钮）

```ahk
; cursor_rpa.ahk
; 参数1：skill 文件路径
skillPath := %1%

; 1. 激活 Cursor 窗口（标题里一般会有 "Cursor"）
IfWinExist, ahk_exe Cursor.exe
{
    WinActivate
    WinWaitActive
}
else
{
    MsgBox, 没找到 Cursor 窗口
    ExitApp
}

; 2. 将剪贴板中的文本粘贴到聊天输入框
; 这里有两种方法：
; - 直接 Send ^v （前提是光标已经在输入框）
; - 或使用坐标点击输入框，再粘贴

; 假设你事先用 Window Spy 记录好了输入框坐标 (相对窗口)
; 例：x=300, y=800（这里是示意，需要你实际调整）
offsetX := 300
offsetY := 800

WinGetPos, winX, winY, winW, winH, A
clickX := winX + offsetX
clickY := winY + offsetY

Click, %clickX%, %clickY%
Sleep, 100
Send, ^v
Sleep, 200

; 3. 点击“上传按钮”（同样需要事先量好坐标）
uploadOffsetX := 650
uploadOffsetY := 800
uploadX := winX + uploadOffsetX
uploadY := winY + uploadOffsetY

Click, %uploadX%, %uploadY%
Sleep, 500

; 4. 在系统“打开文件”对话框中输入 skillPath 并回车
; 一般可以直接 Send skillPath 回车
Send, %skillPath%
Sleep, 300
Send, {Enter}
Sleep, 1000

; 5. 发送信息（如果 Cursor 发送键是回车）
Send, {Enter}
Sleep, 500

; 6. 一键全部同意（Apply All）
; 如果有快捷键，直接发快捷键，比如：Alt+Shift+A（举例，需你自己查）
; Send, !+a

; 或者用坐标点击“Apply all”按钮：
applyOffsetX := 1000
applyOffsetY := 150
applyX := winX + applyOffsetX
applyY := winY + applyOffsetY

Click, %applyX%, %applyY%

ExitApp
```

> 坐标你需要自己用 AutoHotkey 附带的 Window Spy 测一次，改成你机器上的数值。  
> 以后 Cursor UI 不大改，这套脚本就能一直用。

---

### 4. 后端增加一个“给 RPA 用的接口”

再补一个简单接口，让 RPA 拉任务：

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class RPATask(BaseModel):
    has_task: bool
    clipboard_text: str | None = None
    skill_file_path: str | None = None

@app.get("/api/next_task_for_cursor", response_model=RPATask)
def next_task_for_cursor():
    # TODO: 从队列/数据库中取一条待发给 Cursor 的任务
    task = load_one_pending_task()
    if not task:
        return RPATask(has_task=False)
    mark_task_as_dispatched(task.id)
    return RPATask(
        has_task=True,
        clipboard_text=task.clipboard_text,
        skill_file_path=task.skill_file_path,
    )
```

---

### 5. 现实限制和你需要注意的点

- **这是“桌面自动化”，不是官方 API**：  
  - UI 一改，坐标要重新测；  
  - 但优点是：**你完全控制本地电脑，不依赖任何外部接口**。
- **安全性**：  
  - RPA 脚本相当于在你机器上“有键盘鼠标权限”，建议只在自己开发机用，不要随便给别人。
- **稳定性**：  
  - 尽量用 **窗口相对坐标 / 控件名 / 快捷键**，少用绝对坐标。  
  - 如果后面有精力，可以用 `pywinauto` 走 Windows UIAutomation，比纯坐标更稳。

---

如果你愿意，下一步我可以：

- 帮你把 **Python 守护进程 + FastAPI 接口** 写成一套可直接跑的最小代码  
- 再帮你把 AutoHotkey 脚本精简成“只需要你改坐标和快捷键”的版本（你照着一步步调试即可）。
