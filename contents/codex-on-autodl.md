# 在 AutoDL 服务器上配置 VS Code Codex 插件

## 问题

通过 VS Code Remote SSH 在 AutoDL 上运行 Codex 扩展时，远程服务器可能因无法访问 OpenAI 服务而卡在认证或请求阶段。

## 前提

本地电脑已经运行代理软件，并确认 `7897` 是 HTTP 代理端口或 Mixed Port。

## 1. 配置 SSH 反向代理

编辑本地电脑的 SSH 配置文件：

```sshconfig
Host alias
    HostName connect.****.seetacloud.com
    User root
    Port ****

    # 将 AutoDL 的 127.0.0.1:17890
    # 转发到本地电脑的代理端口 127.0.0.1:7897
    RemoteForward 127.0.0.1:17890 127.0.0.1:7897

    ServerAliveInterval 60
    ServerAliveCountMax 3
    ExitOnForwardFailure yes
````

修改后，断开当前 VS Code Remote SSH 连接，再重新连接该主机。

## 2. 安装远程 Codex 扩展

在已经连接 AutoDL 的 VS Code 窗口中打开扩展页面，安装 Codex 扩展，并确认其状态为：

```text
Installed in SSH: alias
```

## 3. 配置远程 VS Code 代理

打开命令面板：

```text
Ctrl + Shift + P
```

运行：

```text
Preferences: Open Remote Settings (JSON)
```

确认打开的是：

```text
Remote [SSH: alias]
```

加入以下设置：

```json
{
    "http.proxy": "http://127.0.0.1:17890",
    "http.proxySupport": "override"
}
```

如果设置文件中已有其他内容，只合并这两个字段，不要覆盖原有配置。

## 4. 重启远程扩展宿主

打开命令面板并运行：

```text
Remote-SSH: Kill VS Code Server on Host...
```

选择 AutoDL 主机，然后重新连接。

重新打开 Codex 扩展，选择：

```text
Sign in with ChatGPT
```

在本地浏览器中完成认证即可。

````

还建议在正式登录前，在 AutoDL 终端验证隧道：

```bash
curl -x http://127.0.0.1:17890 \
  -sS -o /dev/null \
  -w "HTTP %{http_code}\n" \
  https://api.openai.com/v1/models
````

返回 `HTTP 401` 通常说明代理通路正常，只是测试请求没有携带 API Key。若出现 `Connection refused`，优先检查本地代理是否启动、端口是否确实为 `7897`，以及 VS Code 是否通过修改后的 SSH 配置重新建立了连接。
