## 原因

Docker 防火墙使用的是底层的 iptables，firewalld 默认不生效。

如果想要使用 firewalld，需要做以下调整：

## 步骤

重新构建 `DOCKER-USER` chain (即使 DOCKER-USER 已存在，也需要删除后重建)

```sh
firewall-cmd --permanent --direct --remove-chain ipv4 filter DOCKER-USER
firewall-cmd --permanent --direct --remove-rules ipv4 filter DOCKER-USER
firewall-cmd --permanent --direct --add-chain ipv4 filter DOCKER-USER
```

添加 iptables 规则到 DOCKER-USER chain

```sh
firewall-cmd --permanent --direct --add-rule ipv4 filter DOCKER-USER 0 -i docker0 -j ACCEPT -m comment --comment "allows incoming from docker"

firewall-cmd --permanent --direct --add-rule ipv4 filter DOCKER-USER 0 -i docker0 -o eth0 -j ACCEPT -m comment --comment "allows docker to eth0"

firewall-cmd --permanent --direct --add-rule ipv4 filter DOCKER-USER 0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT -m comment --comment "allows docker containers to connect to the outside world"

firewall-cmd --permanent --direct --add-rule ipv4 filter DOCKER-USER 0 -j RETURN -s 172.17.0.0/16 -m comment --comment "allow internal docker communication"
```

添加自定义的规则

```sh
# 允许指定 ip 流量通过, 将 1.1.1.1 替换成你需要通过的 ip
firewall-cmd --permanent --direct --add-rule ipv4 filter DOCKER-USER 0 -s 1.1.1.1/32 -j ACCEPT

# 允许指定 ip 访问指定的端口（示例）
firewall-cmd --permanent --direct --add-rule ipv4 filter DOCKER-USER 0 -p tcp -m multiport --dports 80,443 -s 1.1.1.1/32 -j ACCEPT # 你也可以指定端口，這裡僅為示例

# 拒绝其他流量
firewall-cmd --permanent --direct --add-rule ipv4 filter DOCKER-USER 10 -j REJECT --reject-with icmp-host-unreachable -m comment --comment "reject all other traffic"
```

**注意：**

- `REJECT` 规则要在最后执行
- 同一条规则不要写多个 IP 地址
- 如果在 Docker 运行时重启 firewalld，那么 firewalld 将删除 DOCKER-USER

不要忘记

```sh
firewall-cmd --reload
```

验证：

通过 `iptables -L` 查看 DOCKER-USER 的配置，或者通过 `cat /etc/firewalld/direct.xml` 查看 direct 的配置

