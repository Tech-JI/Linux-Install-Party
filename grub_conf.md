# GRUB2 启动菜单配置指南 (Fedora UEFI 系统)

**文档版本：** 1.0
**最后更新：** 2025-05-14 (基于对话)
**适用系统：** Fedora Linux (UEFI 引导模式)

## 目录

1.  [引言与重要提示](#1-引言与重要提示)
2.  [GRUB 核心文件与命令 (Fedora 特定)](#2-grub-核心文件与命令-fedora-特定)
3.  [备份与恢复准备](#3-备份与恢复准备)
4.  [管理启动项](#4-管理启动项)
    *   [4.1. 禁用/启用 `os-prober` (自动检测其他系统)](#41-禁用启用-os-prober-自动检测其他系统)
    *   [4.2. 使用 `/etc/grub.d/40_custom` 添加自定义启动项 (例如 Windows)](#42-使用-etcgrubd40_custom-添加自定义启动项-例如-windows)
        *   [4.2.1. 获取原始 `menuentry`](#421-获取原始-menuentry)
        *   [4.2.2. 编辑 `40_custom` 并修改名称](#422-编辑-40_custom-并修改名称)
        *   [4.2.3. 使用 `GRUB_OS_PROBER_SKIP_LIST` 避免重复条目](#423-使用-grub_os_prober_skip_list-避免重复条目)
    *   [4.3. 管理 Fedora 内核启动项](#43-管理-fedora-内核启动项)
        *   [4.3.1. 理解 BootLoaderSpec (BLS) 和 `/boot/loader/entries/`](#431-理解-bootloaderspec-bls-和-bootloaderentries)
        *   [4.3.2. 移除旧内核版本](#432-移除旧内核版本)
        *   [4.3.3. 修改 Fedora 启动项显示名称](#433-修改-fedora-启动项显示名称)
5.  [美化 GRUB 界面](#5-美化-grub-界面)
    *   [5.1. 设置 GRUB 分辨率 (`GRUB_GFXMODE`)](#51-设置-grub-分辨率-grub_gfxmode)
    *   [5.2. 设置背景图片 (`GRUB_BACKGROUND`)](#52-设置背景图片-grub_background)
    *   [5.3. 设置 GRUB 字体 (`GRUB_FONT`)](#53-设置-grub-字体-grub_font)
    *   [5.4. 使用 GRUB 主题 (`GRUB_THEME`)](#54-使用-grub-主题-grub_theme)
    *   [5.5. 基本颜色设置 (`GRUB_COLOR_NORMAL`, `GRUB_COLOR_HIGHLIGHT`)](#55-基本颜色设置-grub_color_normal-grub_color_highlight)
6.  [控制 GRUB 菜单显示行为](#6-控制-grub-菜单显示行为)
    *   [6.1. 设置菜单超时 (`GRUB_TIMEOUT`)](#61-设置菜单超时-grub_timeout)
    *   [6.2. 设置菜单显示样式 (`GRUB_TIMEOUT_STYLE`)](#62-设置菜单显示样式-grub_timeout_style)
    *   [6.3. 解决菜单自动隐藏问题 (禁用 `12_menu_auto_hide`)](#63-解决菜单自动隐藏问题-禁用-12_menu_auto_hide)
7.  [应用更改与故障排除](#7-应用更改与故障排除)
    *   [7.1. 更新 GRUB 配置命令](#71-更新-grub-配置命令)
    *   [7.2. 常见问题与解决方案](#72-常见问题与解决方案)

---

## 1. 引言与重要提示

本文档旨在帮助用户在 Fedora UEFI 系统上自定义 GRUB2 启动加载器的菜单项和外观。GRUB 是 Linux 系统中常用的引导加载程序，允许用户选择要启动的操作系统或内核。

**⚠️ 重要提示：**
*   **数据备份：** 在对 GRUB 配置进行任何修改之前，务必备份所有重要数据。
*   **Live 环境准备：** 熟悉如何使用 Fedora Live USB/DVD 启动并在需要时修复 GRUB。错误配置可能导致系统无法启动。
*   **小心操作：** 严格按照步骤操作，理解每一步的含义。

---

## 2. GRUB 核心文件与命令 (Fedora 特定)

*   **主要配置文件 (用户修改)：** `/etc/default/grub`
    *   定义 GRUB 的全局行为和默认外观设置。
*   **GRUB 脚本目录：** `/etc/grub.d/`
    *   包含多个脚本，`grub2-mkconfig` 会按顺序执行这些脚本来生成最终的 `grub.cfg`。
    *   `00_header`: 设置头部信息，包括超时、默认主题等。
    *   `10_linux`: 使用 `blscfg` 根据 `/boot/loader/entries/` 下的 BLS 文件生成 Fedora 内核启动项。
    *   `12_menu_auto_hide` (Fedora 特有): 可能根据上次启动状态自动隐藏菜单或减少超时。
    *   `30_os-prober` (或 `30_os-prober_proxy`): 负责检测其他操作系统。
    *   `31_uefi-firmware`: 添加 UEFI 固件设置的启动项。
    *   `40_custom`: 用户添加自定义启动项的地方，这里的条目不会被自动更新覆盖。
*   **生成的 GRUB 配置文件 (不要直接编辑此文件！)：**
    *   在你的 Fedora UEFI 系统中，主配置文件通常是：`/boot/grub2/grub.cfg`
    *   EFI 分区上的 `grub.cfg` (`/boot/efi/EFI/fedora/grub.cfg`) 可能只是一个指向 `/boot/grub2/grub.cfg` 的包装器或存根。
*   **更新 GRUB 配置的命令：**
    ```bash
    sudo grub2-mkconfig -o /boot/grub2/grub.cfg
    ```
    **每次修改 `/etc/default/grub` 或 `/etc/grub.d/` 目录下的文件后，都必须运行此命令。**

---

## 3. 备份与恢复准备

在开始之前，备份关键配置文件：
```bash
sudo cp /etc/default/grub /etc/default/grub.bak
sudo cp -R /etc/grub.d /etc/grub.d.bak
sudo cp /boot/grub2/grub.cfg /boot/grub2/grub.cfg.bak # 备份当前生效的GRUB配置
```
如果出现问题，你可以从备份中恢复这些文件，然后重新运行 `sudo grub2-mkconfig -o /boot/grub2/grub.cfg`。

---

## 4. 管理启动项

### 4.1. 禁用/启用 `os-prober` (自动检测其他系统)

`os-prober` 会扫描磁盘以查找其他操作系统（如 Windows）并为其创建启动项。

*   **禁用 `os-prober`** (推荐，如果你想完全手动控制非 Fedora 启动项)：
    编辑 `/etc/default/grub`：
    ```bash
    sudo nano /etc/default/grub
    ```
    添加或修改行：
    ```ini
    GRUB_DISABLE_OS_PROBER=true
    ```
*   **启用 `os-prober`**：
    在 `/etc/default/grub` 中设置：
    ```ini
    GRUB_DISABLE_OS_PROBER=false
    ```
    (或者注释掉 `GRUB_DISABLE_OS_PROBER` 这一行，默认是启用的)。
    确保 `os-prober` 包已安装：`sudo dnf install os-prober`。

### 4.2. 使用 `/etc/grub.d/40_custom` 添加自定义启动项 (例如 Windows)

这是添加或重命名非 Fedora 启动项（如 Windows）的推荐方法。

#### 4.2.1. 获取原始 `menuentry` (如果需要参考)
如果 `os-prober` 之前检测到过该系统，你可以查看 `/boot/grub2/grub.cfg` (不要修改它) 来复制其 `menuentry` 块作为参考。

#### 4.2.2. 编辑 `40_custom` 并修改名称
```bash
sudo nano /etc/grub.d/40_custom
```
确保文件结构如下，并将你的 `menuentry` 块添加到注释之后：
```grub
#!/bin/sh
exec tail -n +3 $0
# This file provides an easy way to add custom menu entries.  Simply type the
# menu entries you want to add after this comment.  Be careful not to change
# the 'exec tail' line above.

menuentry 'Windows 10' --class windows --class os $menuentry_id_option 'osprober-efi-YOUR_ESP_UUID' {
        insmod part_gpt
        insmod fat
        search --no-floppy --fs-uuid --set=root YOUR_ESP_UUID
        chainloader /efi/Microsoft/Boot/bootmgfw.efi # 标准 UEFI Windows 路径
}```
*   将 `Windows 10` 替换为你想要的名称。
*   将 `YOUR_ESP_UUID` 替换为你的 Windows EFI 系统分区 (ESP) 的文件系统 UUID (例如 `6078-2E2D`)。你可以使用 `lsblk -f` 或 `sudo blkid` 查找。
*   **重要：** 确保此文件有执行权限：`sudo chmod +x /etc/grub.d/40_custom`。

#### 4.2.3. 使用 `GRUB_OS_PROBER_SKIP_LIST` 避免重复条目
如果你启用了 `os-prober` 但又在 `40_custom` 中定义了同一个系统，为避免重复，可以在 `/etc/default/grub` 中告诉 `os-prober` 跳过该分区：
```ini
# GRUB_OS_PROBER_SKIP_LIST="YOUR_ESP_UUID@/dev/your_esp_device_name"
# 例如: GRUB_OS_PROBER_SKIP_LIST="6078-2E2D@/dev/nvme1n1p1"
```
如果已设置 `GRUB_DISABLE_OS_PROBER=true`，则不需要此项。

### 4.3. 管理 Fedora 内核启动项

Fedora 使用 BootLoaderSpec (BLS)，内核条目由 `/boot/loader/entries/` 下的 `.conf` 文件定义。

#### 4.3.1. 理解 BootLoaderSpec (BLS) 和 `/boot/loader/entries/`
该目录包含形如 `[machine-id]-[kernel-version].conf` 的文件，每个文件定义一个内核启动项。
*   `machine-id`: 系统的唯一标识符，来自 `/etc/machine-id`。
*   `kernel-version`: 内核的具体版本号。
*   `0-rescue.conf`: 救援模式启动项。

#### 4.3.2. 移除旧内核版本
1.  **配置保留数量：** 编辑 `/etc/dnf/dnf.conf`，设置 `installonly_limit` (例如 `installonly_limit=2` 保留最新的2个内核)。
2.  **自动移除：** 运行 `sudo dnf autoremove`。这将移除超出 `installonly_limit` 限制的旧内核及其对应的 `/boot/loader/entries/` 文件。
3.  **不要移除当前内核 (`uname -r`) 或所有备用内核。**

#### 4.3.3. 修改 Fedora 启动项显示名称
1.  定位到 `/boot/loader/entries/` 目录下你想要修改的内核对应的 `.conf` 文件。
2.  使用 `sudo nano` 编辑该文件。
3.  修改 `title` 行的值为你想要的名称。例如：
    ```
    title      My Fedora (Kernel 6.x.x)
    ```
4.  保存文件。**注意：** 内核更新时，新内核的 `.conf` 文件会以默认标题生成，你可能需要再次修改。

---

## 5. 美化 GRUB 界面

所有这些设置都在 `/etc/default/grub` 文件中进行。

### 5.1. 设置 GRUB 分辨率 (`GRUB_GFXMODE`)
```ini
GRUB_GFXMODE="2560x1440,auto" # 设置为你显示器支持的分辨率，可加多个备选
# GRUB_GFXPAYLOAD_LINUX=keep # （可选）让内核保持此分辨率
```
在 GRUB 菜单按 `c` 进入命令行，输入 `videoinfo` 查看支持的模式。

### 5.2. 设置背景图片 (`GRUB_BACKGROUND`)
1.  将图片 (PNG, JPG, TGA) 复制到 `/boot/grub2/` 目录 (或其他 GRUB 能访问的路径)。
    ```bash
    sudo cp /path/to/your/image.jpg /boot/grub2/mybackground.jpg
    ```
2.  在 `/etc/default/grub` 中设置：
    ```ini
    GRUB_BACKGROUND="/boot/grub2/mybackground.jpg"
    ```

### 5.3. 设置 GRUB 字体 (`GRUB_FONT`)
1.  GRUB 使用 `.pf2` 格式字体。转换 `.ttf` 或 `.otf` 字体：
    ```bash
    # 示例：将 JetBrainsMono 转换为 myfont.pf2，大小为18
    sudo grub2-mkfont -s 18 -o /boot/grub2/fonts/myfont.pf2 /path/to/JetBrainsMono.ttf
    # 确保 /boot/grub2/fonts 目录存在，如果不存在则创建：sudo mkdir -p /boot/grub2/fonts
    ```
2.  在 `/etc/default/grub` 中设置：
    ```ini
    GRUB_FONT="/boot/grub2/fonts/myfont.pf2"
    ```

### 5.4. 使用 GRUB 主题 (`GRUB_THEME`)
1.  将主题文件夹 (例如 `my-theme`) 放到 `/boot/grub2/themes/` 或 `/usr/share/grub/themes/` 目录下。
    ```bash
    sudo cp -R /path/to/your/my-theme /usr/share/grub/themes/
    ```
2.  在 `/etc/default/grub` 中指定主题的 `theme.txt` 文件：
    ```ini
    GRUB_THEME="/usr/share/grub/themes/my-theme/theme.txt"
    ```
    你的主题路径是：`GRUB_THEME="/usr/share/grub/themes/tela/theme.txt"`

### 5.5. 基本颜色设置 (`GRUB_COLOR_NORMAL`, `GRUB_COLOR_HIGHLIGHT`)
主要用于无主题的文本模式。如果使用主题，主题颜色会覆盖这些。
```ini
# GRUB_COLOR_NORMAL="light-gray/black"
# GRUB_COLOR_HIGHLIGHT="magenta/black"
```

---

## 6. 控制 GRUB 菜单显示行为

这些设置也在 `/etc/default/grub` 中。

### 6.1. 设置菜单超时 (`GRUB_TIMEOUT`)
设置菜单显示的秒数。
```ini
GRUB_TIMEOUT="15" # 菜单显示15秒。如果为0，则不显示菜单直接启动默认项。
```

### 6.2. 设置菜单显示样式 (`GRUB_TIMEOUT_STYLE`)
控制菜单如何显示。
```ini
GRUB_TIMEOUT_STYLE=menu # 总是显示菜单
# 其他可能的值: hidden (按Shift/Esc显示), countdown (显示倒计时后自动启动)
```

### 6.3. 解决菜单自动隐藏问题 (禁用 `12_menu_auto_hide`)
在 Fedora 中，`/etc/grub.d/12_menu_auto_hide` 脚本有时会根据上次启动是否成功来覆盖你在 `/etc/default/grub` 中设置的 `GRUB_TIMEOUT` 和 `GRUB_TIMEOUT_STYLE`，导致菜单被隐藏或超时变为0。
**解决方案：移除该脚本的执行权限。**
```bash
sudo chmod -x /etc/grub.d/12_menu_auto_hide
```
之后重新生成 GRUB 配置，这将阻止该脚本运行。

---

## 7. 应用更改与故障排除

### 7.1. 更新 GRUB 配置命令
**在对 `/etc/default/grub` 或 `/etc/grub.d/` 目录下的任何文件进行修改后，必须运行以下命令来使更改生效：**
```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

### 7.2. 常见问题与解决方案

*   **`grub2-mkconfig` 输出不显示自定义条目，但条目实际已添加：**
    *   **现象：** `sudo grub2-mkconfig ...` 的屏幕输出没有明确提到正在添加来自 `40_custom` 的条目。
    *   **检查：** 使用 `sudo cat /boot/grub2/grub.cfg | grep 'Your Menu Entry Name'` 查看最终配置文件中是否包含该条目。
    *   **原因：** `40_custom` 通过 `exec tail` 直接输出内容，`grub2-mkconfig` 可能不会为这种方式生成特定的状态消息。只要条目在最终的 `grub.cfg` 中，就是正常的。

*   **GRUB 菜单不显示，直接进入系统：**
    1.  **检查 `/etc/default/grub`：**
        *   确保 `GRUB_TIMEOUT` 是一个正数 (如 `10` 或 `15`)。
        *   确保 `GRUB_TIMEOUT_STYLE=menu`。
    2.  **检查 `/etc/grub.d/40_custom`：**
        *   确保 `#!/bin/sh` 在第一行。
        *   确保 `exec tail -n +3 $0` 在第二行。
        *   确保 `menuentry` 语法正确。
        *   确保文件有执行权限 (`sudo chmod +x /etc/grub.d/40_custom`)。
    3.  **禁用 `12_menu_auto_hide`：** 运行 `sudo chmod -x /etc/grub.d/12_menu_auto_hide`。
    4.  **重新生成配置：** `sudo grub2-mkconfig -o /boot/grub2/grub.cfg`。
    5.  **强制显示：** 重启时尝试按住 `Shift` 或反复按 `Esc`。
    6.  **简化测试：** 临时移除 `/etc/default/grub` 中的主题、字体等美化设置，只保留基本超时和样式配置，然后重新生成并测试。

*   **GRUB 菜单出现多余的 Windows 启动项：**
    *   **原因：** `os-prober` 可能在多个位置检测到 Windows 文件，或者你同时使用了 `os-prober` 和 `40_custom` 来定义同一个 Windows 系统。
    *   **解决方案：**
        1.  在 `/etc/default/grub` 中设置 `GRUB_DISABLE_OS_PROBER=true`，然后只依赖 `40_custom` 来定义你的 Windows 启动项。
        2.  如果必须使用 `os-prober`，使用 `GRUB_OS_PROBER_SKIP_LIST` 来精确排除你已在 `40_custom` 中定义的分区。

---

**在进行任何修改后，务必运行 `sudo grub2-mkconfig -o /boot/grub2/grub.cfg` 并重启电脑以查看效果。**
