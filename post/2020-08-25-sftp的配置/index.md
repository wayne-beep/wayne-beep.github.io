
## SFTP配置过程
1. 连接用户根据用户需求添加系统用户配置密码

   ```bash
   $ useradd sftp_test
   $ passwd sftp_test
   ```

2. 将用户加入sftp-group组（已预先添加）

   ```shell
   $ usermod -a -G  sftp-group sftp_test
   ```

3. 创建chroot目录，该目录为sftp用户登录的目录，权限为root，限制用户在此目录操作权限，并且创建用户的操作目录

   ```shell
   $ mkdir /data/sftp
   $ mkdir /data/sftp/test
   ```

4. 授权操作目录

   ```shell
   $ chown sftp_test:sftp-group /data/sftp/test
   ```

5. 配置ssh配置文件

   ```shell
   Protocol 2
   Port 22
   AddressFamily any
   ListenAddress 0.0.0.0
   AllowGroups root
   AllowGroups deploy sftp-group
   AuthorizedKeysCommand /usr/bin/sss_ssh_authorizedkeys
   AuthorizedKeysCommandUser root
   ChallengeResponseAuthentication yes
   LogLevel INFO
   PasswordAuthentication yes
   PermitEmptyPasswords no
   PermitRootLogin yes
   PubkeyAuthentication yes
   Subsystem sftp-user /usr/lib/openssh/sftp-server
   SyslogFacility AUTHPRIV
   UsePAM yes
   X11Forwarding yes
   Subsystem sftp /usr/lib/openssh/sftp-server
   Match Group sftp-group
   ChrootDirectory /data/sftp
   AllowTcpForwarding no
   PermitTunnel no
   ForceCommand internal-sftp
   ```

   其中新增的配置如下：

   ```shell
   AllowGroups deploy sftp-group
   Subsystem sftp /usr/lib/openssh/sftp-servers
   Match Group sftp-group
   ChrootDirectory /data/sftp
   AllowTcpForwarding no
   PermitTunnel no
   ForceCommand internal-sftp
   ```

6. 重启sshd服务

   ```shell
   $ systemctl restart sshd
   ```

   

## SFTP新增用户

1. 新增用户并将用户加入sftp-group组

   ```bash
   $ useradd tom -s /sbin/nologin -m -d /data/sftp/tom
   $ passwd tom # 密码随机生成即可
   $ usermod -a -G sftp-group tom
   ```

2. 切换用户，确保家目录路径正确

   ```bash
    $ su - tom -s /bin/bash
    $ pwd
   /data/sftp/tom
   ```

3. 授权家目录为root

   ```bash
   $ chown -R root:root /data/sftp/tom
   ```

4. 在家目录创建上传目录并授权用户

   ```bash
   mkdir tom/tomUpload
   chown -R tom:tom  tom/tomUpload
   ```

5. 查看/etc/ssh/sshd_config

   ```bash
   # 配置文件确保ChrootDirectory关联到自己的家目录
   ChrootDirectory /data/sftp/%u
   # 以及匹配组为sftp-group
   Match Group sftp-group
   ```

6. 通过prod和test来区分生产或测试环境

   ```bash
   mkdir prod
   mkdir test
   ```

7. 测试连接

   ```bash
   $ sftp -oPort=22 tom@ip`
   ```

