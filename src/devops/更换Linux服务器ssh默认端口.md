## 环境

centos 7

## 步骤

备份 ssh 配置文件

```sh
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
```

修改默认端口

```
vi etc/ssh/sshd_config
```

修改`#Port 22` 为 `Port 2233` ，则 ssh 端口分改为 2233

重启 sshd 服务

```sh
systemctl restart sshd
```

**注意：**

防火墙需要放行新的端口

```sh
firewall-cmd --premanent --add-port=80/tcp
firewall-cmd --reload
```

