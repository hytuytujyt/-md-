# Caddy 反向代理配置指南（Hub 用）

## 一、整体结构

```
浏览器 → https://域名:8443 → Caddy → 127.0.0.1:28080 → Hub
```

| 端口 | 用途 |
|---|---|
| 80 | Caddy 申请和续签证书用，必须开放 |
| 8443 | 访问 Hub 的 HTTPS 入口，可换成其他未占用端口 |
| 443 | 节点在用，不要动 |

## 二、开始前检查

```bash
ss -tlnp | grep 28080                      # Hub 在运行，应看到 127.0.0.1:28080
sudo ss -tlnp | grep -E ':(80|443|8443)\s' # 看端口占用
```

- 80 和 8443 没有输出，才能用。
- 域名已经准备好，例如 `status.hytuytujyt.top`。

## 三、安装 Caddy

```bash
which caddy || echo "caddy 未安装"
```

输出 `caddy 未安装` 时，依次执行：

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install -y caddy
```

## 四、写配置文件

整段复制到终端执行：

```bash
sudo tee /etc/caddy/Caddyfile > /dev/null <<'EOF'
{
        https_port 8443
}

status.hytuytujyt.top:8443 {
        reverse_proxy 127.0.0.1:28080
}
EOF
```

> 不要把配置内容单独粘贴到终端，否则每一行都会被当成命令执行，报 `command not found`。上面的命令会把内容写入 `/etc/caddy/Caddyfile`。

检查写入结果：

```bash
sudo cat -n /etc/caddy/Caddyfile
```

## 五、验证并启动

```bash
sudo caddy validate --config /etc/caddy/Caddyfile
```

输出 `Valid configuration` 后继续：

```bash
sudo systemctl enable --now caddy
sudo systemctl restart caddy
sudo systemctl status caddy --no-pager
```

看到 `active (running)` 即正常。

## 六、放行端口

本机没装 ufw 时（`sudo ufw` 报 `command not found`），只需在**云服务商网页的安全组**里放行：

| 协议 | 端口 | 授权IP |
|---|---|---|
| TCP | 80 | 0.0.0.0/0 |
| TCP | 8443 | 0.0.0.0/0 |

保存后约 1 分钟生效，不需要重启。

## 七、配置 DNS（Cloudflare）

查公网 IP：

```bash
curl -4 ifconfig.me; echo
```

在 Cloudflare 的域名 DNS 页面添加或修改 A 记录：

- 名称：`status`
- 内容：上一步查到的 IP
- 代理状态：**仅 DNS**（灰云）

等 1 到 5 分钟，检查解析：

```bash
dig +short status.hytuytujyt.top
```

## 八、验证

**服务器本机：**

```bash
curl -I -k --max-time 10 --resolve status.hytuytujyt.top:8443:127.0.0.1 https://status.hytuytujyt.top:8443
```

返回 `HTTP/2 200` 即正常。

**Windows 外网：**

```powershell
Test-NetConnection status.hytuytujyt.top -Port 8443
curl.exe -I https://status.hytuytujyt.top:8443
```

- `TcpTestSucceeded : True` 表示端口通。（`PingSucceeded` 是 ICMP，忽略它。）
- 返回 `HTTP/1.1 200 OK` 表示成功。

## 九、常见问题

| 现象 | 解决办法 |
|---|---|
| `address already in use`，提到 `:443` | 用上面的 `https_port 8443` 配置 |
| 8443 外网 `False`，80 为 `True` | 安全组没放行 8443 |
| 本机 200，外网超时 | 安全组没放行，或 DNS 指错 |
| 证书申请失败 | 检查 DNS 和安全组的 80 端口 |
| `caddy` 显示 `failed` | `sudo journalctl -u caddy --no-pager \| tail -n 30` |

## 十、注意事项

1. 配置用 `tee` 写入，不要直接粘贴到终端。
2. 不动 443，不停节点程序。
3. 改配置后先 `validate`，再 `restart`。
4. 80 端口要一直开着，否则证书无法续签。
5. 访问地址要带端口：`https://域名:8443`。
6. Hub 还在用时，DNS 的 A 记录不要删。
7. 安全组改动立即生效，Caddy 配置改动要 `restart`。

## 十一、卸载

```bash
sudo systemctl disable --now caddy
sudo apt remove --purge caddy
sudo apt autoremove
```

然后到 Cloudflare 手动删除对应的 DNS 记录。
