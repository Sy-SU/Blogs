# conda 换清华源

## 问题
```bash
root@autodl-container-wtl0lh99as-844a5052:~/LATO.2# . ./setup.sh --all
Collecting package metadata (current_repodata.json): failed

CondaSSLError: Encountered an SSL error. Most likely a certificate verification issue.

Exception: HTTPSConnectionPool(host='repo.anaconda.com', port=443): Max retries exceeded with url: /pkgs/main/linux-64/current_repodata.json (Caused by SSLError(CertificateError("hostname 'repo.anaconda.com' doesn't match '112.121.184.230'")))

Error: 'conda create' failed (a corrupted conda package cache is a common cause; try 'conda clean --all')
```

HTTPS 访问被导向了错误的主机：

```text
repo.anaconda.com → 112.121.184.230
```

对方返回的证书不属于 `repo.anaconda.com`，所以 Conda 拒绝连接。常见原因是代理配置错误、DNS 异常，或者 `/etc/hosts` 中存在错误映射。Conda 会读取环境变量中的代理，也会读取 `.condarc` 的 `proxy_servers` 配置。


## 解决方法
```bash
cp ~/.condarc ~/.condarc.bak 2>/dev/null || true

cat > ~/.condarc <<'EOF'
channels:
  - defaults
show_channel_urls: true
default_channels:
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/r
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/msys2
custom_channels:
  conda-forge: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  pytorch: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
EOF

conda clean -i -y
```