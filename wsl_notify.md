# win设备使用WSL终端任务结束后通知方案

下面是一套完整方案：在 **Windows 电脑 + WSL 终端** 中运行命令，命令结束后弹出 **Windows 右下角通知**。适合在 WSL 里配置环境变量、激活 conda/venv，然后运行 Python 训练、测试、脚本等任务。

一、安装 Windows 通知模块

在 **Windows PowerShell** 里执行，不是在 WSL 里：

```powershell
Install-Module -Name BurntToast -Scope CurrentUser
```

然后在 WSL 里测试 Windows PowerShell 能否调用：

```bash
/mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe -NoProfile -Command "Write-Host hello"
```

如果能输出 `hello`，继续下一步。

二、在 WSL 的 `.bashrc` 里添加 `nt` 函数

编辑 WSL 的 Bash 配置：

```bash
vim ~/.bashrc
```

在文件末尾加入：

```bash
nt() {
  local status=$?
  local cmd="${*:-task}"
  local ps="/mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe"

  local cmd_b64
  cmd_b64=$(printf '%s' "$cmd" | base64 -w 0)

  "$ps" -NoProfile -ExecutionPolicy Bypass -Command "
Import-Module BurntToast

\$cmdText = [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String('$cmd_b64'))
\$statusCode = '$status'

New-BurntToastNotification -Text 'Task Done.', \"cmd: \$cmdText\", \"code: \$statusCode\"
" >/dev/null 2>&1

  return "$status"
}
```

保存后加载配置：

```bash
source ~/.bashrc
```

三、基本用法

你的命令正常写，后面加：

```bash
; nt 任务说明
```

例如：

```bash
python time.py; nt python time.py
```

或者：

```bash
python train.py --config configs/a.yaml; nt train demo
```

通知会显示类似：

```text
task done.
cmd: train demo
code: 0
```

其中：

```text
code: 0
```

表示命令成功结束。非 0 通常表示命令失败或异常退出。

四、适合你的日常工作流

例如你经常先配置变量、激活环境，再运行 Python：

```bash
export CUDA_VISIBLE_DEVICES=0
conda activate myenv

python train.py --config configs/a.yaml; nt train demo
```

这个方案不会接管 Python 命令的执行。`python train.py ...` 仍然在当前 WSL shell、当前 conda 环境、当前环境变量下正常运行。`nt` 只是在它结束之后读取退出码并发送通知。

五、测试

创建一个简单测试文件：

```bash
cat > time.py <<'EOF'
import time
time.sleep(2)
EOF
```

运行：

```bash
python time.py; nt python time.py
```

两秒后应该出现 Windows 右下角通知。

测试失败退出码：

```bash
bash -c 'exit 7'; nt fail test
echo $?
```

通知里应显示：

```text
code: 7
```

如果只想成功后通知，用：

```bash
bash -c 'exit 7' && nt fail test
```


