# VyOS ARM64 AMI 构建指南

从 GitHub Actions 构建的 ISO 创建可在 AWS EC2 Graviton 上运行的 AMI。

## 前置条件

- 一台 EC2 arm64 实例作为构建工作台（Debian/Ubuntu，需要有额外 EBS 卷）
- AWS CLI 已配置
- 构建工作台上安装: `parted dosfstools e2fsprogs grub-common`
- 最新 ISO 来自 https://github.com/MakerHe/vyos-arm64-build/releases

## 流程概览

```
下载 ISO → 分区 EBS → 安装 VyOS → 配置 GRUB/串口/网络/SSM → 创建 Snapshot → 注册 AMI
```

## 详细步骤

### 1. 准备工作盘

在构建工作台实例上 attach 一个 4GB EBS volume（如 `/dev/nvme2n1`）。

```bash
sudo apt-get install -y parted dosfstools e2fsprogs grub-common
```

### 2. 下载 ISO

```bash
cd /tmp
# 获取最新 release URL
RELEASE_URL=$(curl -s https://api.github.com/repos/MakerHe/vyos-arm64-build/releases/latest \
  | jq -r '.assets[] | select(.name | endswith(".iso")) | .browser_download_url')
wget -q "$RELEASE_URL" -O vyos-new.iso
```

### 3. 分区和格式化

```bash
DISK=/dev/nvme2n1  # 改为你的目标盘

sudo wipefs -a $DISK
sudo parted -s $DISK mklabel gpt
sudo parted -s $DISK mkpart EFI fat32 1MiB 257MiB
sudo parted -s $DISK set 1 esp on
sudo parted -s $DISK mkpart persistence ext4 257MiB 100%
sudo partprobe $DISK
sleep 2

sudo mkfs.vfat -F 32 -n EFI ${DISK}p1
sudo mkfs.ext4 -L persistence -F ${DISK}p2
```

### 4. 挂载

```bash
sudo mkdir -p /mnt/iso /mnt/efi /mnt/persist
sudo mount -o loop /tmp/vyos-new.iso /mnt/iso
sudo mount ${DISK}p1 /mnt/efi
sudo mount ${DISK}p2 /mnt/persist
```

### 5. 安装 EFI 引导

```bash
PERSIST_UUID=$(sudo blkid -s UUID -o value ${DISK}p2)

sudo mkdir -p /mnt/efi/EFI/BOOT
sudo cp /mnt/iso/EFI/boot/bootaa64.efi /mnt/efi/EFI/BOOT/BOOTAA64.EFI
sudo cp /mnt/iso/EFI/boot/grubaa64.efi /mnt/efi/EFI/BOOT/grubaa64.efi

# GRUB stub - 通过 UUID 查找 persistence 分区
sudo tee /mnt/efi/EFI/BOOT/grub.cfg << EOF
search.fs_uuid ${PERSIST_UUID} root
set prefix=(\$root)'/boot/grub'
configfile \$prefix/grub.cfg
EOF
```

### 6. 安装 VyOS 文件系统

```bash
sudo mkdir -p /mnt/persist/boot/base
sudo mkdir -p /mnt/persist/boot/grub/grub.cfg.d/vyos-versions

# 复制核心文件
sudo cp /mnt/iso/live/filesystem.squashfs /mnt/persist/boot/base/base.squashfs
sudo cp /mnt/iso/live/vmlinuz /mnt/persist/boot/base/vmlinuz
sudo cp /mnt/iso/live/initrd.img /mnt/persist/boot/base/initrd.img

# persistence 配置
echo "/ union" | sudo tee /mnt/persist/persistence.conf
```

### 7. 配置 GRUB

```bash
# 主 grub.cfg
sudo tee /mnt/persist/boot/grub/grub.cfg << 'G'
load_env
insmod regexp

for cfgfile in ${prefix}/grub.cfg.d/*-autoload.cfg
do
    source ${cfgfile}
done
G

# grubenv - 关键：设置串口 console
sudo grub-editenv /mnt/persist/boot/grub/grubenv create
sudo grub-editenv /mnt/persist/boot/grub/grubenv set console_type=ttyS
sudo grub-editenv /mnt/persist/boot/grub/grubenv set console_num=0
sudo grub-editenv /mnt/persist/boot/grub/grubenv set console_speed=115200

# GRUB 配置片段
sudo tee /mnt/persist/boot/grub/grub.cfg.d/10-vyos-modules-autoload.cfg << 'G'
insmod all_video
insmod part_gpt
insmod ext2
insmod fat
G

sudo tee /mnt/persist/boot/grub/grub.cfg.d/20-vyos-defaults-autoload.cfg << 'G'
set default="vyos-base"
set timeout="5"
set console_type="ttyS"
set console_num="0"
set console_speed="115200"
set bootmode="normal"
G

sudo tee /mnt/persist/boot/grub/grub.cfg.d/25-vyos-common-autoload.cfg << 'G'
set boot_toram="no"
G

sudo tee /mnt/persist/boot/grub/grub.cfg.d/40-vyos-menu-autoload.cfg << 'G'
for cfgfile in ${prefix}/grub.cfg.d/vyos-versions/*.cfg
do
    source ${cfgfile}
done
G

# 版本启动项
sudo tee /mnt/persist/boot/grub/grub.cfg.d/vyos-versions/base.cfg << 'G'
menuentry "VyOS" --id vyos-base {
    set boot_opts="boot=live rootdelay=5 noautologin net.ifnames=0 biosdevname=0 vyos-union=/boot/base"
    if [ "${console_type}" == "ttyS" ]; then
        set console_opts="console=${console_type}${console_num},${console_speed}"
    else
        set console_opts="console=${console_type}${console_num}"
    fi
    set boot_opts="${boot_opts} ${console_opts}"
    linux "/boot/base/vmlinuz" ${boot_opts}
    initrd "/boot/base/initrd.img"
}
G
```

### 8. 预配置 VyOS (config.boot + SSM)

```bash
# config.boot - DHCP + SSH + serial console
sudo mkdir -p /mnt/persist/boot/base/rw/opt/vyatta/etc/config
sudo tee /mnt/persist/boot/base/rw/opt/vyatta/etc/config/config.boot << 'CONFIG'
interfaces {
    ethernet eth0 {
        address dhcp
    }
    loopback lo {
    }
}
service {
    ssh {
        port 22
    }
}
system {
    host-name vyos
    login {
        user vyos {
            authentication {
                encrypted-password "$6$rounds=656000$SJsG87tq0bbBWrqZ$BRzd3F6tTbAjGZrH.Pja3qxeABa2l47hSFUWAdqyaeZxMPo7vsrzlfmkngu9HklJMUdeBqrVxbeGfyZ0DFDg30"
                public-keys macbook {
                    key "AAAAC3NzaC1lZDI1NTE5AAAAIK+vNgQZqxPCuijsLL/+wi0/85n7Bzgk1CWRD62OMi4C"
                    type ssh-ed25519
                }
            }
        }
    }
    console {
        device ttyS0 {
            speed 115200
        }
    }
    update-check {
        auto-check
        url "https://raw.githubusercontent.com/MakerHe/vyos-arm64-build/refs/heads/master/version.json"
    }
}
CONFIG

# 启用 SSM Agent 开机自启
sudo mkdir -p /mnt/persist/boot/base/rw/etc/systemd/system/multi-user.target.wants
sudo ln -sf /lib/systemd/system/amazon-ssm-agent.service \
  /mnt/persist/boot/base/rw/etc/systemd/system/multi-user.target.wants/amazon-ssm-agent.service
```

### 9. 卸载

```bash
sudo sync
sudo umount /mnt/iso /mnt/efi /mnt/persist
```

### 10. 创建 AMI

```bash
REGION=us-east-1
VOL_ID=vol-xxxxx  # 你的 EBS volume ID

# 创建 snapshot
SNAP_ID=$(aws ec2 create-snapshot \
  --volume-id $VOL_ID \
  --description "VyOS arm64 $(date +%Y.%m.%d)" \
  --region $REGION \
  --query SnapshotId --output text)

aws ec2 wait snapshot-completed --snapshot-ids $SNAP_ID --region $REGION

# 注册 AMI
AMI_ID=$(aws ec2 register-image \
  --architecture arm64 \
  --block-device-mappings "[{\"DeviceName\":\"/dev/xvda\",\"Ebs\":{\"DeleteOnTermination\":true,\"SnapshotId\":\"$SNAP_ID\",\"VolumeSize\":4,\"VolumeType\":\"gp3\"}}]" \
  --boot-mode uefi \
  --description "VyOS arm64 with AWS packages" \
  --ena-support \
  --name "vyos-arm64-$(date +%Y.%m.%d)" \
  --root-device-name /dev/xvda \
  --virtualization-type hvm \
  --region $REGION \
  --query ImageId --output text)

echo "AMI: $AMI_ID"
```

## AMI 特性

| 特性 | 说明 |
|------|------|
| 架构 | arm64 (Graviton) |
| 启动模式 | UEFI |
| Serial Console | ttyS0 @ 115200 |
| 网络 | eth0 DHCP, ENA |
| SSH | port 22, 公钥认证 |
| SSM Agent | 开机自启 |
| CloudWatch Agent | 已安装，需配置后启动 |
| AWS GWLB | aws-gwlbtun + vyos-1x-aws 已安装 |
| 默认用户 | vyos (密码: vyos) |

## 注意事项

- `grubenv` 中 `console_type=ttyS` 是关键，否则 EC2 serial console 无输出
- config.boot 必须符合 VyOS 语法，否则启动时配置解析失败导致无网络
- SSM Agent 需要实例有 IAM Role 或启用 Default Host Management 才能工作
- CloudWatch Agent 需要手动创建配置文件 (`/opt/aws/amazon-cloudwatch-agent/etc/`)
- ISO 是 generic 构建，NVMe 驱动已内置在内核中 (`CONFIG_BLK_DEV_NVME=y`)

## 当前 AMI

| AMI ID | Region | 日期 |
|--------|--------|------|
| `ami-018fcd03ead38ab58` | us-east-1 | 2026-05-29 |
