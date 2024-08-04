## 使用vagrant创建虚拟机
参考博文：[超详细的 Vagrant 上手指南，搭建Linux虚拟机别再用VMware Workstation了！_==> vagrant: a new version of vagrant is available-CSDN博客](https://blog.csdn.net/MacWx/article/details/114525895?ydreferer=aHR0cHM6Ly93d3cuYmluZy5jb20v?ydreferer=aHR0cHM6Ly93d3cuYmluZy5jb20v)
辅助参考：[如何在 Ubuntu 20.04 配置 zRAM (myfreax.com)](https://www.myfreax.com/how-to-configure-zram-on-ubuntu-20-04/)
:::info
注意：安装Linux时需要注意版本问题，版本过早不自带zram。本人安装的是Ubuntu20.04LTS版。
:::
![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722593614160-92f2cca1-5ca2-4a67-93d1-a20a7aa75bd4.png#averageHue=%231f1f1f&clientId=u4cfe9fbf-4210-4&from=paste&height=133&id=uc86c5e4c&originHeight=199&originWidth=1254&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=29768&status=done&style=none&taskId=u6c2d9ca1-ccaf-4455-884b-134add5c3ff&title=&width=836)
## 如何使用linux下的zram。  
### 安装zram工具
一些发行版已经包含了zram工具，但你可能需要安装额外的工具来配置它。例如，对于Debian或Ubuntu，你可以使用以下命令安装`zram-tools`：
```
sudo apt-get update
sudo apt-get install zram-tools
```
![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722593461405-99b12491-7331-4101-a13e-c0007ff99d6b.png#averageHue=%23212121&clientId=u4cfe9fbf-4210-4&from=paste&height=1024&id=udd3dab60&originHeight=1536&originWidth=1288&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=260463&status=done&style=none&taskId=u6405122e-abb3-4830-8fa3-bea5ed65212&title=&width=858.6666666666666)
![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722593503037-c95ddbce-4618-4c2c-adb9-1933b873b496.png#averageHue=%231b1b1b&clientId=u4cfe9fbf-4210-4&from=paste&height=455&id=u34f1c4c7&originHeight=682&originWidth=1278&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=97387&status=done&style=none&taskId=uf40e993b-8b10-493e-b25d-83212d4228a&title=&width=852)
### 配置zram
可以通过创建和编辑配置文件来配置zram。在Debian或Ubuntu上，可以编辑`/etc/default/zramswap`文件：
```
sudo nano /etc/default/zramswap
```
在文件中，可以配置zram的参数，例如zram设备的数量和每个设备的大小。以下是一个简单的配置示例：
```
# Enable/Disable zramswap (default: true)
ENABLED=true

# Specifies the amount of RAM to use for the zram device(s)
# Percent is based on the total amount of available RAM (default: 50%)
PERCENT=50
```
![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722596376713-da0c39f4-a40c-4afc-bae6-e060b6ca1f74.png#averageHue=%23151515&clientId=u4cfe9fbf-4210-4&from=paste&height=493&id=ue15d2cae&originHeight=739&originWidth=1275&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=79975&status=done&style=none&taskId=u180775c6-85b5-48b5-9a78-3fc9a7ad28c&title=&width=850)
:::info
_**配置完成**_
:::
### 启动zram(可能会有问题)
配置完成后，可以使用以下命令启动zram：
```
sudo systemctl enable zramswap
sudo systemctl start zramswap
```
### 开启/加载 zRAM 模块
```shell
sudo modprobe zram num_devices=1
lsmod  | grep zram
```
```
zram                   32768  0
```
:::info
_**感觉没有什么反应？**_![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722758109759-98d7f75d-b8ab-4f89-8a2d-915b499b8163.png#averageHue=%23131212&clientId=u4cfe9fbf-4210-4&from=paste&height=250&id=u8b742a3f&originHeight=375&originWidth=1047&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=77280&status=done&style=none&taskId=ua1d50f93-ac6c-4809-8a6f-bc5edd443d7&title=&width=698)
**好像还是有的**
:::

### 持久化开启/加载 zRAM 模块
modprobe 加载的模块仅在当前运行时可用，要在重新启动计算机后，加载 zram 模块，则需要创建内核模块载入文件和模块的配置文件。
要载入 zRAM 模块在/etc/modules-load.d/目录创建文件zram.conf，运行命令 echo "zram" | sudo tee -a /etc/modules-load.d/zram.conf。
然后创建模块的配置文件 zram.conf 在目录 /etc/modules-load.d/，运行命令 echo "options zram num_devices=1" | sudo tee -a /etc/modprobe.d/zram.conf 。
```shell
echo "zram" | sudo tee -a /etc/modules-load.d/zram.conf

echo "options zram num_devices=1" | sudo tee -a /etc/modprobe.d/zram.conf
```
:::info
**结果：**![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722758303593-59ca638b-2d82-4de6-a7b1-d72fef280e6e.png#averageHue=%23141414&clientId=u4cfe9fbf-4210-4&from=paste&height=125&id=ue7654134&originHeight=187&originWidth=1252&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=27681&status=done&style=none&taskId=u460e8c1d-32ab-4d85-8ee4-ae3864cb287&title=&width=834.6666666666666)
:::

### 配置 zRAM
通常 zRAM 最常用的设置选项是 zRAM 的大小 disksize 和 zRAM 的压缩算法 comp_algorithm。
控制 zRAM 的大小的文件是 /sys/block/zram0/disksize ，压缩算法文件是 /sys/block/zram0/comp_algorithm 。
可以运行 cat 命令查看两个文件来确定 zRAM 文件的大小和压缩算法，例如命令 cat /sys/block/zram0/disksize 查看 zRAM 大小。
同样也可以通过两个文件设置 zRAM 文件的大小和压缩算法，例如命令 echo "512M" | sudo tee /sys/block/zram0/disksize 修改 zRAM 的大小。
```shell
echo "512M" | sudo tee /sys/block/zram0/disksize
echo "lzo" | sudo tee /sys/block/zram0/comp_algorithm
```
:::info
**结果：**
![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722758477718-92b19cbb-8008-44ca-9783-8954cff2eed6.png#averageHue=%23171717&clientId=u4cfe9fbf-4210-4&from=paste&height=194&id=u05e93e58&originHeight=291&originWidth=1144&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=46008&status=done&style=none&taskId=u8491b431-e307-4f64-8843-19e59cfdf24&title=&width=762.6666666666666)
:::

### 持久化 zRAM 配置
由于 /sys 目录是基于内存的文件系统，因此重启系统后，你设置的 zRAM 参数将不再存在。要持久化 zRAM 的参数，请使用 udev 进行设置。
udev 是一个用户空间系统，简单来说就是 Linux 设备管理器。它使操作系统管理员能够为设备事件运行用户指定的程序或者脚本，也可以在添加设备时指定设备参数。
要设置 zRAM 的大小，可以在 udev 规则文件的 ATTR 指定 zRAM 大小 disksize，ATTR{disksize}="512M" 表示设置 zRAM 的大小是 512M。
要设置 zRAM 的其它属性，可以在 udev 规则文件指定多个 ATTR 属性设置设备参数。例如设置 zRAM 压缩算法 ，可以添加属性 ATTR{comp_algorithm}="lzo"。
```shell
# Persistent

echo 'KERNEL=="zram0", ATTR{disksize}="512M",TAG+="systemd"' | sudo tee  /etc/udev/rules.d/99-zram.rules

echo 'KERNEL=="zram0", ATTR{disksize}="512M", ATTR{comp_algorithm}="lzo", TAG+="systemd"' | sudo tee  /etc/udev/rules.d/99-zram.rules
```
![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722771886443-e1432952-8325-495d-9849-3b30ea55c004.png#averageHue=%231f1f1f&clientId=u36736a14-5309-4&from=paste&height=227&id=ue368ca41&originHeight=340&originWidth=1231&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=75937&status=done&style=none&taskId=udd08ba9c-8085-4ead-831a-6966713eb62&title=&width=820.6666666666666)
### 激活 zRAM
要在当前运行时启用 zRAM 可以将 zRAM 当作交换空间一样的方式启用。首先需要将 zRAM 文件进行格式化，运行命令 sudo mkswap /dev/zram0 。
当格式化完成后，为了让系统识别 zRAM 文件，因此还需要启用 zRAM 文件。可以运行命令 sudo swapon /dev/zram0 启用 zRAM 文件。
如果你系统已经存在交换空间，那么总的交换空间的大小是zRAM的大小加交换空间的大小。
```
sudo mkswap /dev/zram0
sudo swapon /dev/zram0
```
![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722772911850-60cc3856-dd61-4b30-a31d-1f531c6ddfa0.png#averageHue=%23212121&clientId=u36736a14-5309-4&from=paste&height=75&id=u710b3004&originHeight=112&originWidth=961&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=26950&status=done&style=none&taskId=u66845224-38a0-48f7-a1dd-3b1de3d9461&title=&width=640.6666666666666)
### 自动挂载 zRAM
现在已经在内核启用 zRAM 模块，接下来创建 [SystemD 单元文件](https://www.myfreax.com/how-to-customize-linux-systemd-unit-files/)，在系统启动时自动激活 zRAM 文件并作为交换空间挂载。
使用你喜欢的文本编辑器创建文件 /etc/systemd/system/zram.service 。在本教程中，我们使用vim 创建并[编辑文件](https://www.myfreax.com/the-basis-of-the-linux-vim-command/) [SystemD 单元文件](https://www.myfreax.com/how-to-customize-linux-systemd-unit-files/)。
编辑完成后，[保存文件并退出 vim](https://www.myfreax.com/how-to-save-file-in-vim-quit-editor/) 编辑器，然后运行命令 sudo systemctl enable zram 启用 zram 服务，最后重启计算机。
```shell
sudo vim /etc/systemd/system/zram.service
sudo systemctl enable zram
```
```
[Unit]
Description=Swap with zram
After=multi-user.target

[Service]
Type=oneshot
RemainAfterExit=true
ExecStartPre=/sbin/mkswap /dev/zram0
ExecStart=/sbin/swapon /dev/zram0
ExecStop=/sbin/swapoff /dev/zram0

[Install]
WantedBy=multi-user.target
```
![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722772875984-b93cd571-d542-4b60-bbbe-66c22a805f74.png#averageHue=%23191919&clientId=u36736a14-5309-4&from=paste&height=99&id=u34e16557&originHeight=148&originWidth=1255&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=26014&status=done&style=none&taskId=ue681698d-fb8c-4b9c-85e3-7b422235a5b&title=&width=836.6666666666666)
验证zram是否正常工作：
```
swapon -s
```
:::info
![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722773071744-89a3c718-0037-46de-8779-9c1a7f4fe011.png#averageHue=%23131313&clientId=u36736a14-5309-4&from=paste&height=84&id=u4b98922b&originHeight=126&originWidth=1156&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=18993&status=done&style=none&taskId=uba5ae0db-abb4-4f8e-a406-643dde219fc&title=&width=770.6666666666666)
:::
### 关闭zram
如果需要关闭zram设备，可以使用以下命令：
```
sudo swapoff /dev/zram0
sudo rmmod zram
```
:::info
![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722773188208-aebca702-808e-4778-b301-c531ebf418eb.png#averageHue=%23121212&clientId=u36736a14-5309-4&from=paste&height=76&id=u0099df3a&originHeight=114&originWidth=724&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=14506&status=done&style=none&taskId=u5a154708-43bb-4fcb-91e2-396a70cb6b3&title=&width=482.6666666666667)
![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722773323850-4672ee92-0e18-4010-8b8b-dcc011a9a23e.png#averageHue=%23131313&clientId=u36736a14-5309-4&from=paste&height=114&id=u3616d9f5&originHeight=171&originWidth=1165&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=23434&status=done&style=none&taskId=ud5dc7a5a-5ca4-42d6-9cb5-5c6494f5ea2&title=&width=776.6666666666666)
:::

### 脚本配置zram
你还可以编写一个脚本来自动配置zram。例如，创建一个名为`setup_zram.sh`的脚本：
```
#!/bin/bash

# Load zram module
modprobe zram

# Set the size of the zram device
echo 2G > /sys/block/zram0/disksize

# Create swap on the zram device
mkswap /dev/zram0
swapon /dev/zram0

echo "zram setup complete"
```
![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722773541609-bb6e67b7-2df3-4c59-9349-4c1b7bfd110d.png#averageHue=%23111111&clientId=u36736a14-5309-4&from=paste&height=131&id=ue28c7c08&originHeight=196&originWidth=858&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=29997&status=done&style=none&taskId=uc3286581-a10f-48b3-be66-916f1909b8c&title=&width=572)
给脚本可执行权限并运行它：
```
chmod +x setup_zram.sh
sudo ./setup_zram.sh
```
这样就可以快速配置和使用zram了。这些步骤可以根据具体需要进行调整和优化。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722773738381-27612fab-2cf5-4e52-9661-273da54d93e7.png#averageHue=%23161616&clientId=u36736a14-5309-4&from=paste&height=301&id=u42c611f9&originHeight=451&originWidth=1231&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=84733&status=done&style=none&taskId=ue6f8e167-3f8d-4a94-9e3e-b1a28a71ccf&title=&width=820.6666666666666)
##  假设我已经配置好一个zram设备了，我怎么能在c程序中使用这个设备呢
### 使用zram作为块设备
假设你的zram设备是`/dev/zram0`，你可以在C程序中使用标准的文件I/O函数来读写该设备。
#### 安装gcc
```shell
sudo apt-get update
sudo apt-get install gcc
```
#### 打开zram设备
```shell
vim open_zram.c
```
```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

#define DEVICE_PATH "/dev/zram0"

int main() {
    int fd = open(DEVICE_PATH, O_RDWR);
    if (fd == -1) {
        perror("Failed to open zram device");
        return 1;
    }

    // Write data to zram
    const char *data = "Hello, zram!";
    if (write(fd, data, strlen(data)) == -1) {
        perror("Failed to write to zram device");
        close(fd);
        return 1;
    }

    // Seek to the beginning of the device
    if (lseek(fd, 0, SEEK_SET) == -1) {
        perror("Failed to seek zram device");
        close(fd);
        return 1;
    }

    // Read data from zram
    char buffer[1024];
    ssize_t bytesRead = read(fd, buffer, sizeof(buffer) - 1);
    if (bytesRead == -1) {
        perror("Failed to read from zram device");
        close(fd);
        return 1;
    }

    buffer[bytesRead] = '\0';
    printf("Read from zram: %s\n", buffer);

    close(fd);
    return 0;
}
```
```shell
gcc -o open_zram open_zram.c
sudo ./open_zram
```
![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722777833133-d2e31e05-b486-4b9f-a060-38c9a9317b06.png#averageHue=%23151515&clientId=u36736a14-5309-4&from=paste&height=155&id=ube283f5a&originHeight=232&originWidth=919&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=45070&status=done&style=none&taskId=u0c04d7a0-d0cb-47c9-a089-86cd4ef358a&title=&width=612.6666666666666)
:::info
**不加sudo会出现权限不够的问题**
:::

### 使用zram作为交换设备
如果你使用zram作为交换设备，通常不需要直接在C程序中进行特定的操作，因为内核会自动管理交换内存。不过，你可以通过`sysctl`接口或读取`/proc`文件系统来获取一些状态信息。
#### 获取zram交换设备的使用情况
```shell
vim get_zram.c
```
```c
#include <stdio.h>
#include <stdlib.h>

int main() {
    FILE *fp = fopen("/proc/swaps", "r");
    if (fp == NULL) {
        perror("Failed to open /proc/swaps");
        return 1;
    }

    char line[256];
    while (fgets(line, sizeof(line), fp)) {
        printf("%s", line);
    }

    fclose(fp);
    return 0;
}
```
```shell
gcc -o get_zram get_zram.c
sudo ./get_zram
```
![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722778027136-89a5d678-6f4d-41ab-b6dc-572ed6fb7bc0.png#averageHue=%23141414&clientId=u36736a14-5309-4&from=paste&height=137&id=u20be01cd&originHeight=205&originWidth=1249&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=44912&status=done&style=none&taskId=uaeed8b8f-ebcc-4de9-8487-8ef04508b45&title=&width=832.6666666666666)
#### 更详细的zram信息
你还可以从`/sys/block/zram0`目录下获取更详细的信息。例如，读取压缩的统计数据：
```shell
vim get_zram_stats.c
```
```c
#include <stdio.h>
#include <stdlib.h>

#define ZRAM_STAT_PATH "/sys/block/zram0/mm_stat"

int main() {
    FILE *fp = fopen(ZRAM_STAT_PATH, "r");
    if (fp == NULL) {
        perror("Failed to open zram stats");
        return 1;
    }

    char line[256];
    while (fgets(line, sizeof(line), fp)) {
        printf("%s", line);
    }

    fclose(fp);
    return 0;
}
```
```shell
gcc -o get_zram_stats get_zram_stats.c
sudo ./get_zram_stats
```
![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722778663618-af7c1eb5-d9eb-4199-a07d-2457d429858b.png#averageHue=%23151515&clientId=u36736a14-5309-4&from=paste&height=137&id=uc19e1b74&originHeight=205&originWidth=1254&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=46664&status=done&style=none&taskId=ud3767869-42d2-43d4-a69b-4f6ca83ff64&title=&width=836)
##  可以直接像文件一样读写这个块设备吗？  
是的，zram块设备可以像普通文件一样进行读写操作。下面是一个完整的示例程序，展示了如何在C程序中将数据写入zram设备并读取数据。
### 示例程序：在zram设备上读写数据
假设zram设备是`/dev/zram0`，下面的程序将演示如何打开设备、写入数据、读取数据并关闭设备。
```shell
vim read_zram.c
```
```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

#define DEVICE_PATH "/dev/zram0"

int main() {
    // 打开zram设备
    int fd = open(DEVICE_PATH, O_RDWR);
    if (fd == -1) {
        perror("无法打开zram设备");
        return 1;
    }

    // 写入数据到zram
    const char *data = "Hello, zram!";
    ssize_t bytes_written = write(fd, data, strlen(data));
    if (bytes_written == -1) {
        perror("写入zram设备失败");
        close(fd);
        return 1;
    }

    printf("成功写入%ld字节到zram设备\n", bytes_written);

    // 将文件偏移量移动到设备开头
    if (lseek(fd, 0, SEEK_SET) == -1) {
        perror("无法移动文件偏移量");
        close(fd);
        return 1;
    }

    // 从zram设备读取数据
    char buffer[1024];
    ssize_t bytes_read = read(fd, buffer, sizeof(buffer) - 1);
    if (bytes_read == -1) {
        perror("读取zram设备失败");
        close(fd);
        return 1;
    }

    // 确保读取的内容是以空字符结尾的字符串
    buffer[bytes_read] = '\0';
    printf("从zram设备读取的数据: %s\n", buffer);

    // 关闭zram设备
    close(fd);
    return 0;
}
```
```shell
gcc -o read_zram read_zram.c
sudo ./read_zram
```
![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722778935736-73196617-c745-4275-b0b3-006abd3dc485.png#averageHue=%23141414&clientId=u36736a14-5309-4&from=paste&height=117&id=ud08915ea&originHeight=175&originWidth=1275&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=39665&status=done&style=none&taskId=uf2aa2156-954b-41a8-bd89-20995b845f1&title=&width=850)
:::info
**和之前的打开zram设备的代码几乎一样**
:::

### 解释代码

- `open(DEVICE_PATH, O_RDWR)`: 以读写模式打开zram设备。
- `write(fd, data, strlen(data))`: 将数据写入zram设备。
- `lseek(fd, 0, SEEK_SET)`: 将文件偏移量设置到设备的起始位置，以便之后读取数据。
- `read(fd, buffer, sizeof(buffer) - 1)`: 从zram设备读取数据。
- `close(fd)`: 关闭zram设备。

这个程序展示了如何在C程序中使用zram设备，就像操作普通文件一样。这种方式允许你利用zram的压缩特性，在内存有限的系统上提高性能。
##  我怎么知道这个我写进去的数据被压缩了？  
要验证写入到zram设备的数据是否被压缩，可以通过检查zram的统计信息。这些信息通常可以在`/sys/block/zram0`目录下找到。具体来说，可以查看`/sys/block/zram0/compr_data_size`和`/sys/block/zram0/orig_data_size`文件来比较原始数据大小和压缩后的数据大小。
以下是一个示例程序，用于在写入数据后读取这些统计信息，以验证数据是否被压缩：

### 验证设置
检查 ZRAM 设备是否已经作为交换空间使用：
```
sudo swapon --show
```
### 测试 ZRAM 性能
你可以通过运行一些内存密集型的应用程序来测试 ZRAM 的效果。例如，使用 `stress` 工具来模拟高内存负载：
```
sudo apt-get install stress
stress --vm 2 --vm-bytes 256M --timeout 60s
```
![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722780294347-11305996-b42c-4625-87e3-d93e0d3a2e2a.png#averageHue=%23191919&clientId=u36736a14-5309-4&from=paste&height=416&id=u56b57b43&originHeight=624&originWidth=1251&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=91638&status=done&style=none&taskId=uf8e339d6-bf2c-46db-b8f0-dc4f6d0a683&title=&width=834)
![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722780339524-de1854a5-aa2c-404a-8034-b5dcd6441bc1.png#averageHue=%231e1e1e&clientId=u36736a14-5309-4&from=paste&height=57&id=ue7d1abf8&originHeight=85&originWidth=1270&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=20181&status=done&style=none&taskId=u85117216-89c3-4a99-83da-6a89bb8a4ba&title=&width=846.6666666666666)
这将运行一个虚拟内存压力测试，持续 60 秒，使用大约 256M 的虚拟内存。观察系统在测试期间的表现，检查是否有明显的延迟或性能下降。
### 示例程序：验证zram数据压缩
```shell
vim zram_compression_check.c
```
```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

#define DEVICE_PATH "/dev/zram0"
#define COMPR_DATA_SIZE_PATH "/sys/block/zram0/compr_data_size"
#define ORIG_DATA_SIZE_PATH "/sys/block/zram0/orig_data_size"

void read_zram_stat(const char *path, unsigned long long *value) {
    FILE *file = fopen(path, "r");
    if (!file) {
        perror("无法打开zram统计信息文件");
        exit(1);
    }
    if (fscanf(file, "%llu", value) != 1) {
        perror("读取zram统计信息失败");
        fclose(file);
        exit(1);
    }
    fclose(file);
}

int main() {
    // 打开zram设备
    int fd = open(DEVICE_PATH, O_RDWR);
    if (fd == -1) {
        perror("无法打开zram设备");
        return 1;
    }

    // 写入数据到zram
    const char *data = "Hello, zram! This is a test data to check compression.";
    ssize_t bytes_written = write(fd, data, strlen(data));
    if (bytes_written == -1) {
        perror("写入zram设备失败");
        close(fd);
        return 1;
    }
    printf("成功写入%ld字节到zram设备\n", bytes_written);

    // 读取压缩前后的数据大小
    unsigned long long compr_data_size = 0;
    unsigned long long orig_data_size = 0;

    read_zram_stat(COMPR_DATA_SIZE_PATH, &compr_data_size);
    read_zram_stat(ORIG_DATA_SIZE_PATH, &orig_data_size);

    printf("原始数据大小: %llu 字节\n", orig_data_size);
    printf("压缩后数据大小: %llu 字节\n", compr_data_size);

    // 关闭zram设备
    close(fd);

    return 0;
}
```
```shell
gcc -o zram_compression_check zram_compression_check.c
sudo ./zram_compression_check
```
:::danger
**出现问题：并没有/sys/block/zram0/compr_data_size和/sys/block/zram0/orig_data_size文件显示压缩数据和元数据的压缩比。***
:::

### 编译和运行程序

1. 将上述代码保存为`zram_compression_check.c`。
2. 使用以下命令编译程序：
```
gcc zram_compression_check.c -o zram_compression_check
```

3. 运行编译后的可执行文件：
```
sudo ./zram_compression_check
```
### 解释代码

- `open(DEVICE_PATH, O_RDWR)`: 以读写模式打开zram设备。
- `write(fd, data, strlen(data))`: 将数据写入zram设备。
- `read_zram_stat(COMPR_DATA_SIZE_PATH, &compr_data_size)`: 读取压缩后数据的大小。
- `read_zram_stat(ORIG_DATA_SIZE_PATH, &orig_data_size)`: 读取原始数据的大小。
- `printf`: 打印原始数据大小和压缩后数据大小。

通过比较原始数据大小和压缩后数据大小，你可以确定写入的数据是否被压缩。如果`compr_data_size`小于`orig_data_size`，则数据被成功压缩了。
##  我希望你写一段sh脚本，配置一个1G大小的zram设备，并输出该设备的目录。  
下面是一个配置1GB大小zram设备的shell脚本，并输出该设备的目录。将以下内容保存为`setup_zram.sh`脚本文件：
```
#!/bin/bash

# 加载zram模块
sudo modprobe zram

# 设置zram设备的大小为1G
echo 1G | sudo tee /sys/block/zram0/disksize

# 创建swap在zram设备上
sudo mkswap /dev/zram0

# 启用swap
sudo swapon /dev/zram0

# 输出zram设备的目录
echo "zram设备目录: /sys/block/zram0"
```
### 使用说明

1. 将上述脚本保存为`setup_zram.sh`。
2. 给脚本赋予可执行权限：
```
chmod +x setup_zram.sh
```

3. 运行脚本：
```
sudo ./setup_zram.sh
```
### 脚本解释

- `sudo modprobe zram`: 加载zram内核模块。
- `echo 1G | sudo tee /sys/block/zram0/disksize`: 设置zram设备的大小为1GB。
- `sudo mkswap /dev/zram0`: 在zram设备上创建swap空间。
- `sudo swapon /dev/zram0`: 启用zram设备的swap空间。
- `echo "zram设备目录: /sys/block/zram0"`: 输出zram设备的目录。

运行此脚本后，zram设备将被配置为1GB，并且将输出其目录信息。
##  写到zram设备里的数据保存在什么地方？  
写到zram设备里的数据保存在内存中，并且是以压缩的形式存储的。zram设备实际上是一个压缩的内存块设备，利用内存的压缩技术来提高有效内存的使用率。具体来说，数据在写入zram设备时，会被压缩并存储在内存中，而不是在磁盘上。
### 数据存储过程

1. **数据写入zram设备**：当你向zram设备写入数据时，数据会先进入内存。
2. **数据压缩**：写入的数据会被zram驱动程序压缩，使用的是内存压缩算法（如LZO或LZ4）。
3. **压缩数据存储在内存中**：压缩后的数据被存储在系统内存中，而不是存储在物理磁盘上。

由于zram的这种工作机制，它可以在有限的物理内存中存储更多的数据，提高系统的内存利用效率。
### 查看zram设备状态
可以通过查看`/sys/block/zram0`目录中的一些文件来了解zram设备的状态和存储信息：

- `/sys/block/zram0/disksize`：zram设备的大小（设置的大小）。
- `/sys/block/zram0/compr_data_size`：当前存储的压缩数据大小。
- `/sys/block/zram0/orig_data_size`：原始未压缩数据的大小。
- `/sys/block/zram0/mem_used_total`：zram设备实际使用的物理内存量。
### 示例：查看zram设备状态的脚本
下面是一个示例脚本，用于显示zram设备的状态信息：
```
#!/bin/bash

# zram设备目录
ZRAM_DIR="/sys/block/zram0"

# 检查zram设备是否存在
if [ ! -d "$ZRAM_DIR" ]; then
    echo "zram设备不存在，请先配置zram设备。"
    exit 1
fi

# 显示zram设备的状态信息
echo "zram设备状态信息："
echo "设备大小: $(cat $ZRAM_DIR/disksize)"
echo "压缩数据大小: $(cat $ZRAM_DIR/compr_data_size)"
echo "原始数据大小: $(cat $ZRAM_DIR/orig_data_size)"
echo "实际使用的物理内存: $(cat $ZRAM_DIR/mem_used_total)"
```
### 使用说明

1. 将上述脚本保存为`check_zram_status.sh`。
2. 给脚本赋予可执行权限：
```
chmod +x check_zram_status.sh
```

3. 运行脚本：
```
./check_zram_status.sh
```
该脚本将输出zram设备的状态信息，包括设备大小、压缩数据大小、原始数据大小和实际使用的物理内存量。通过这些信息，你可以了解zram设备的实际使用情况和压缩效果。
##  我希望你写一个c程序，可以接受外部给的zram地址，并读写之前脚本创建的zram设备，并输出相关信息证明数据被压缩了。最后写一段脚本调用该c程序。  
下面是一个C程序，它可以接受外部传入的zram设备地址，写入数据，读取数据，并输出相关信息来证明数据被压缩了。之后再编写一个脚本调用该C程序。
### C程序
将下面的代码保存为`zram_test.c`：
```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

void read_zram_stat(const char *path, unsigned long long *value) {
    FILE *file = fopen(path, "r");
    if (!file) {
        perror("无法打开zram统计信息文件");
        exit(1);
    }
    if (fscanf(file, "%llu", value) != 1) {
        perror("读取zram统计信息失败");
        fclose(file);
        exit(1);
    }
    fclose(file);
}

int main(int argc, char *argv[]) {
    if (argc != 2) {
        fprintf(stderr, "用法: %s <zram设备路径>\n", argv[0]);
        return 1;
    }

    const char *device_path = argv[1];
    const char *compr_data_size_path = "/sys/block/zram0/compr_data_size";
    const char *orig_data_size_path = "/sys/block/zram0/orig_data_size";

    // 打开zram设备
    int fd = open(device_path, O_RDWR);
    if (fd == -1) {
        perror("无法打开zram设备");
        return 1;
    }

    // 写入数据到zram
    const char *data = "Hello, zram! This is a test data to check compression.";
    ssize_t bytes_written = write(fd, data, strlen(data));
    if (bytes_written == -1) {
        perror("写入zram设备失败");
        close(fd);
        return 1;
    }
    printf("成功写入%ld字节到zram设备\n", bytes_written);

    // 读取压缩前后的数据大小
    unsigned long long compr_data_size = 0;
    unsigned long long orig_data_size = 0;

    read_zram_stat(compr_data_size_path, &compr_data_size);
    read_zram_stat(orig_data_size_path, &orig_data_size);

    printf("原始数据大小: %llu 字节\n", orig_data_size);
    printf("压缩后数据大小: %llu 字节\n", compr_data_size);

    // 关闭zram设备
    close(fd);

    return 0;
}
```
### 编译和运行C程序

1. 编译C程序：
```
gcc zram_test.c -o zram_test
```
### Shell脚本
将下面的脚本保存为`run_zram_test.sh`：
```
#!/bin/bash

# 配置1GB的zram设备
sudo modprobe zram
echo 1G | sudo tee /sys/block/zram0/disksize
sudo mkswap /dev/zram0
sudo swapon /dev/zram0

# 运行zram测试程序
./zram_test /dev/zram0

# 显示zram设备的目录
echo "zram设备目录: /sys/block/zram0"
```
### 使用说明

1. 确保C程序已编译，并在同一目录下有可执行文件`zram_test`。
2. 给脚本赋予可执行权限：
```
chmod +x run_zram_test.sh
```

3. 运行脚本：
```
sudo ./run_zram_test.sh
```
### 脚本解释

- 脚本首先配置1GB大小的zram设备。
- 运行编译好的C程序`zram_test`，传入zram设备路径`/dev/zram0`。
- C程序会写入数据到zram设备，并读取压缩前后的数据大小，以验证数据被压缩。
- 最后，脚本输出zram设备的目录。
