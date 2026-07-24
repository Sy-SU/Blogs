# tmux 的安装与基础使用

## 1. 安装和使用 tmux

AutoDL 实例通常通过 SSH 或 VS Code Remote-SSH 连接。网络中断、终端关闭或本地电脑休眠时，直接运行在前台的训练任务可能被终止。`tmux` 可以在服务器上创建独立的终端会话，使程序在连接断开后继续运行。

需要注意，`tmux` 只能防止 SSH 连接断开导致任务退出。实例关机、重启或被平台释放后，tmux 中的任务仍会停止。

### 1.1 安装 tmux

AutoDL 镜像通常基于 Ubuntu，可以使用以下命令安装：

```bash
apt update
apt install -y tmux
```

安装完成后检查版本：

```bash
tmux -V
```

正常情况下会输出类似内容：

```text
tmux 3.2a
```

### 1.2 创建 tmux 会话

创建一个名为 `train` 的会话：

```bash
tmux new -s train
```

执行后会直接进入该会话。之后可以在其中启动训练任务：

```bash
python train.py
```

建议根据任务类型为会话命名：

```bash
tmux new -s train
tmux new -s download
tmux new -s preprocess
tmux new -s tensorboard
```

会话名称最好使用英文、数字、连字符或下划线，避免使用空格。

### 1.3 临时退出会话

需要退出 tmux，但不停止其中运行的程序时，依次按下：

```text
Ctrl+B
D
```

操作方式是：

1. 按住 `Ctrl`，再按 `B`；
2. 松开按键；
3. 再按一次 `D`。

退出后，终端通常会显示：

```text
[detached from train]
```

此时训练程序仍然在服务器上运行，可以关闭 SSH 或 VS Code。

不要使用 `Ctrl+C` 退出训练会话，因为 `Ctrl+C` 通常会终止当前程序。

### 1.4 查看现有会话

查看所有 tmux 会话：

```bash
tmux ls
```

输出示例：

```text
train: 1 windows
download: 1 windows
```

如果没有正在运行的会话，可能会显示：

```text
no server running
```

### 1.5 重新进入会话

重新进入名为 `train` 的会话：

```bash
tmux attach -t train
```

也可以使用简写：

```bash
tmux a -t train
```

如果当前只有一个 tmux 会话，可以直接运行：

```bash
tmux attach
```

### 1.6 结束会话

在 tmux 会话内部，可以先结束正在运行的程序，然后执行：

```bash
exit
```

当会话中的所有终端都退出后，该 tmux 会话会自动关闭。

也可以在会话外部强制删除指定会话：

```bash
tmux kill-session -t train
```

删除所有 tmux 会话：

```bash
tmux kill-server
```

执行这些命令前，应先确认其中没有仍需运行的训练或下载任务。

### 1.7 常用快捷键

tmux 的大多数快捷键都需要先按前缀键：

```text
Ctrl+B
```

然后松开，再按第二个按键。

| 操作            | 快捷键              |
| ------------- | ---------------- |
| 退出当前会话但保持任务运行 | `Ctrl+B`，然后按 `D` |
| 创建新窗口         | `Ctrl+B`，然后按 `C` |
| 切换到下一个窗口      | `Ctrl+B`，然后按 `N` |
| 切换到上一个窗口      | `Ctrl+B`，然后按 `P` |
| 查看窗口列表        | `Ctrl+B`，然后按 `W` |
| 重命名当前窗口       | `Ctrl+B`，然后按 `,` |
| 进入滚动模式        | `Ctrl+B`，然后按 `[` |
| 关闭当前窗口        | 输入 `exit`        |

在滚动模式中，可以使用方向键、`Page Up` 和 `Page Down` 查看历史输出。退出滚动模式时按：

```text
Q
```

### 1.8 tmux 中的窗口

一个 tmux 会话可以包含多个窗口，可以将其理解为多个独立终端。

创建新窗口：

```text
Ctrl+B
C
```

窗口通常会显示在 tmux 底部状态栏中，例如：

```text
0:bash  1:train  2:tensorboard
```

切换到指定编号的窗口：

```text
Ctrl+B
0
```

或：

```text
Ctrl+B
1
```

这种方式适合在同一个会话中同时运行：

* 模型训练；
* GPU 状态监控；
* TensorBoard；
* 日志查看。

### 1.9 一个常见的使用流程

创建训练会话：

```bash
tmux new -s train
```

进入项目目录：

```bash
cd /root/autodl-tmp/projects/my-project
```

启动训练并保存日志：

```bash
python train.py 2>&1 | tee train.log
```

临时离开会话：

```text
Ctrl+B
D
```

之后重新连接服务器，并查看会话：

```bash
tmux ls
```

重新进入：

```bash
tmux attach -t train
```

训练完成后退出会话：

```bash
exit
```

### 1.10 使用时的注意事项

第一，正式训练应在 tmux 中启动，而不是先在普通终端启动，再尝试放入 tmux。已经运行的普通前台进程不能直接移动到新的 tmux 会话中。

第二，tmux 会话保存在当前实例中。实例关机或重启后，tmux 会话和其中运行的进程都会消失。

第三，tmux 不负责保存训练日志。训练时仍应使用 `tee`、日志文件、TensorBoard 或其他实验记录工具。

第四，每个重要任务最好使用独立会话，避免多个任务混在同一个终端中。例如：

```bash
tmux new -s train_exp01
tmux new -s train_exp02
tmux new -s download_dataset
```

第五，删除会话前先通过以下命令确认任务是否仍在运行：

```bash
tmux ls
nvidia-smi
ps aux | grep python
```
