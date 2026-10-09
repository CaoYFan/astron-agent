# Casdoor 密码存储与升级

Docker Compose 认证部署使用 Argon2id 存储本地密码。Casdoor 2.67 会先创建内置组织，
再读取初始化模板，因此仅修改模板无法更新内置组织或已有数据库。**新安装和升级都必须执行迁移。**

Casdoor 本身仅在 Compose 内部网络监听。对外端口由 `casdoor-gateway` 提供；它仅在
只读密码存储检查通过后启动。检查失败不会修改账户，也不会开放对外认证端口。
内部网络仍须保持可信；启动检查不是持续监控。

## 新安装

在 `docker/astronAgent` 配置 `.env` 后执行：

```bash
# 私有初始化，不启动对外网关
docker compose -f docker-compose-with-auth.yaml up -d casdoor
# 等待初始化完成后预览；只显示计数，不输出密码或哈希
docker compose -f docker-compose-with-auth.yaml run --rm casdoor-passwords
# 应用迁移前必须停止所有 Casdoor 实例
docker compose -f docker-compose-with-auth.yaml stop casdoor
docker compose -f docker-compose-with-auth.yaml run --rm casdoor-passwords --apply
# 启动应用及通过检查的认证网关
docker compose -f docker-compose-with-auth.yaml up -d
```

数据库尚未初始化时，工具会退出且不修改数据；等待初始化完成后重新预览。
首次迁移也会转换 Casdoor 自动创建的管理员账户。开放访问前须修改示例管理员密码、
删除不用的示例账户；哈希存储不能使公开示例密码变成秘密。

RPA 联合部署和独立认证部署分别将上述每条命令的文件名换为
`docker-compose-with-auth-rpa.yaml` 和 `docker-compose-auth.yml`。

## 已有部署

1. 备份 **Casdoor MySQL 数据库**并验证可恢复性，安排认证维护窗口。
   历史备份可能含明文密码，应按敏感数据保管。
2. 停止公开认证和应用访问，停止共享此数据库的所有 Casdoor 实例，包括其他项目中的副本；
   保持 MySQL 运行。旧会话可能仍然有效。
3. 先运行预览并处理拒绝迁移的状态。数据库密码不同时，在 `.env` 中设置已有 `casdoor`
   数据库用户的 `CASDOOR_MYSQL_PASSWORD`；修改环境变量不会自动修改持久化数据库中的密码。
4. 执行 `casdoor-passwords --apply`，再运行不带 `--apply` 的预览，确认账户和策略变更计数均为零。
5. 重启并重新执行检查：

   ```bash
   docker compose -f docker-compose-with-auth.yaml up -d casdoor
   docker compose -f docker-compose-with-auth.yaml up -d --force-recreate casdoor-password-check casdoor-gateway
   ```

6. 用测试账户验证正确及错误密码后再恢复访问。保留管理员恢复途径；必要时恢复备份。
   Argon2id 哈希不能反向恢复为明文。

迁移保留账户 ID 和原登录密码，为每个明文密码生成独立随机盐，保留已有合法 bcrypt/
Argon2id 哈希，并在修改组织策略前固定继承的 bcrypt 元数据。Casdoor 可在成功登录后将
bcrypt 更新为 Argon2id。重复迁移不会再次哈希已有哈希；全部密码和组织策略修改在单个事务中提交。

工具会拒绝不支持或格式错误的哈希、元数据不明的疑似哈希、存在本地密码的 LDAP 账户，
以及组织级主密码/默认密码设置。请通过 Casdoor 管理界面核实并处理后重试，不要将旧摘要
误当明文，也不要通过删除账户绕过检查。无本地密码的外部身份账户保持无密码状态。
工具只针对 Casdoor **v2.67.0** 默认表名的 MySQL 数据库；自定义表前缀、其他数据库和
外部身份系统须单独验证迁移。

保留 `initDataNewOnly = true`。改为 false 可能在重启时从示例模板重建组织和账户。
模板用户的 `passwordType: plain` 表示输入格式，Casdoor 的 `AddUser` 会按组织的
Argon2id 策略哈希后再存储。

自动对外端口检查适用于 Docker Compose。Helm 仍标记为开发中，开放其 Casdoor 服务前，
必须完成等效的私有初始化、迁移和检查。本次代码修改不会部署或修改生产数据库。

测试命令、上游实现链接和完整说明见 [英文文档](../CASDOOR_PASSWORD_STORAGE.md)。
CI 使用真实 Casdoor 2.67 与 MySQL 临时容器验证新用户存储、已有账户迁移、登录、事务回滚、
重复迁移、重启及对外网关阻断行为。
