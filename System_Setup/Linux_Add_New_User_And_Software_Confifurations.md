# Linux Configurations (Ubuntu)

## I. User Manipulation

### 1. Add User

```bash
# add user, -m is used to generate user home because some instance may doesn't generate user home automatically.
sudo useradd -m [username]

# set password
sudo passwd [username]
```

### 2. Delete User

```bash
sudo rm -rf /home/[username]

sudo userdel [username]
```

### 3. Set root  authority and Cancel password for sudo

```bash
# set root authority and cancel password for sudo, add following into the User privilege specification of /etc/sudoers
[username] ALL=(ALL) NOPASSWD:ALL
```

If  write some error into ```/etc/sudoers``` which disables sudo, use the methods in this article [Ubuntu 误修改sudoers 导致 无法使用sudo的解决办法](https://blog.csdn.net/DKBDKBDKB/article/details/117925944?spm=1001.2101.3001.6661.1&utm_medium=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1-117925944-blog-80674542.pc_relevant_multi_platform_whitelistv4&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1-117925944-blog-80674542.pc_relevant_multi_platform_whitelistv4&utm_relevant_index=1)

## III. Software Configurations

### 1. SSH

```bash
# configure /etc/ssh/sshd_config
PasswordAuthentication yes                        #启用密码验证
 
PubkeyAuthentication yes                          #启用密钥对验证
 
AuthorizedKeysFile     .ssh/authorized_keys       #指定公钥库文件（ls -a可查看）

# restart sshd
systemctl restart sshd

# generate key pair
ssh-keygen

# append client's public key into authorized_keys
vim authorized_keys
```

### 2. zsh

```bash
# install zsh
sh -c "$(wget https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh -O -)"
```

