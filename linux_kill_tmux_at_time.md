# linux定时关闭tmux任务

使用 `at` 可以定时执行一次性任务，例如在明天 6 点强制关闭当前用户下的所有 tmux session。

## 1. 确认 at 服务可用

```bash
sudo systemctl status atd
````

如果没有启动：

```bash
sudo systemctl enable --now atd
```

## 2. 定时关闭所有 tmux

```bash
echo "/usr/bin/tmux kill-server" | at 06:00 tomorrow
```

`tmux kill-server` 会关闭当前用户下的所有 tmux session。

## 3. 查看定时任务

```bash
atq
```

输出示例：

```text
5       Mon May 24 06:00:00 2026 a ubuntu
```

其中 `5` 是任务编号。

## 4. 查看任务内容

```bash
at -c 5
```

## 5. 取消定时任务

```bash
atrm 5
```

## 6. 时间说明

`at 06:00 tomorrow` 使用的是服务器本地系统时间，不是本机浏览器或 ChatGPT 的时间。

查看服务器时间和时区：

```bash
date
timedatectl
```

