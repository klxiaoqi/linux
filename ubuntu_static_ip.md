# Ubuntu 18.04 LTS 静态 ＩＰ设置

## 方法1：

> 原来设置 `/etc/network/interfaces` 的方法还可以用，只是设置的dns没有用
>
> 设置 dns 可以编辑 `/etc/systemd/resolved.conf` 文件

```bash
sudo vi /etc/systemd/resolved.conf

[Resolve]
DNS=114.114.114.114

```

## 方法2：

> 编辑 `/etc/netplan/`下的yaml文件
>
> 这里文件名是 `01-network-manager-all.yaml`

```bash
sudo vi /etc/netplan/01-network-manager-all.yaml

# 注释掉 renderer:NetworkManager

network:
	version: 2
	ethernets: 
		# 网络名
		enp0s3:
			# 一个ip数组，用 ‘,’ 隔开
			addresses: [192.168.0.116/24]
			# 使用dhcp 动态获取ip: true/no
			dhcp4: no
			# ipv4 网关
			gateway4: 192.168.0.1
			# dns
			nameservers: 
				addresses: [114.114.114.114]
				search: [localdomain]
			optional: true

# 立即生效
sudo netplan apply
```

### 补充

```bash
# 查看网关
netstat -rn 
# 或
route -n


```

