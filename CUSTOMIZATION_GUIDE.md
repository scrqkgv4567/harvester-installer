# CUSTOMIZATION_GUIDE

## 目标与优先级
按侵入度从低到高给出可执行方案，覆盖安装前/中/后/集群阶段及离线制品。

### 方案 1：利用官方配置入口（低侵入）
1) **适用场景**：添加内核模块、写文件、追加 systemd 单元、调整 RKE2/Harvester chart 参数；满足“在系统初始化前注入服务”和“安装后首启自动部署/修改配置”。  
2) **实现方式**：  
   - 通过 kernel cmdline 或 `configUrl` 的 `harvester.*` 字段填写 `os.writeFiles`、`os.modules`、`os.afterInstallChrootCommands`、`harvester.*` chart 值、`webhooks` 等，安装程序自动渲染到 yip stages 和 RKE2 配置。[F:pkg/config/config.go†L90-L205][F:pkg/config/cos.go†L131-L239][F:pkg/config/cos.go†L320-L460]  
   - 在 `AfterInstallChrootCommands` 写入自定义 systemd unit（如宿主机 agent），文件落在已安装系统后立即执行。  
3) **最小示例**（供 `configUrl` 或 `/oem/userdata.yaml`）：  
```
schemeVersion: 1
os:
  modules: ["mydrv"]
  writeFiles:
    - path: /usr/local/bin/my-agent.sh
      permissions: "0755"
      owner: root:root
      content: |
        #!/bin/bash
        echo started > /var/log/my-agent.log
  afterInstallChrootCommands:
    - echo "[Unit]\nDescription=My Agent\n[Service]\nExecStart=/usr/local/bin/my-agent.sh\n[Install]\nWantedBy=multi-user.target" > /etc/systemd/system/my-agent.service
    - systemctl enable my-agent.service
harvester:
  storageClass:
    replicaCount: 2
install:
  automatic: true
  device: /dev/sda
```
4) **构建与验证**：PXE/ISO 启动时传入 `harvester.install.config_url=<http(s)>`；安装后 `systemctl status my-agent` 验证。  
5) **回滚**：删除 `/etc/systemd/system/my-agent.service`，重跑安装或在下一次 config 调整。  
6) **离线要点**：将脚本嵌入 `writeFiles`，无需额外镜像；若依赖镜像，可放入 bundle（见方案 4）。  
7) **升级兼容性**：高，接口属官方 schema；关注字段变更（查看 `pkg/config/config.go` 差异）。

### 方案 2：Installer 阶段 hook（中等侵入，安装前/中）
1) **适用场景**：在 installer 进入 UI 前执行硬件探测、驱动加载；在执行 elemental 前后插入操作。  
2) **实现方式**：  
   - 修改或追加 `system/oem/*.yaml` yip 阶段，在 `initramfs`/`boot` 添加自研脚本（如加载 out-of-tree 驱动）。[F:package/harvester-os/files/system/oem/90_network.yaml†L1-L8][F:package/harvester-os/files/system/oem/99_modules.yaml†L1-L6]  
   - 在 `/usr/sbin/harv-install` 内增加 hook（如 `clear_disk_label` 后执行自定义清理、`do_preload` 前注入镜像校验）。[F:package/harvester-os/files/usr/sbin/harv-install†L15-L520]  
3) **最小示例**：新建 `package/harvester-os/files/system/oem/05_custom.yaml`：  
```
name: "Custom pre-init"
stages:
  initramfs:
    - name: "load vendor driver"
      commands:
        - modprobe my_vendor_driver
    - name: "prepare logs"
      commands:
        - mkdir -p /run/custom && touch /run/custom/pre-init.ok
```
4) **构建与验证**：重新执行 `make` 生成 ISO；启动后在安装环境查看 `/run/custom/pre-init.ok`。  
5) **回滚**：移除自定义 yip 文件或在 git revert；重新打包 ISO。  
6) **离线要点**：驱动二进制可通过 `writeFiles` 或额外 rpm 放入 ISO；确保 yip 权限 0644。  
7) **升级兼容性**：中，需在升级时 rebase yip 变化。

### 方案 3：安装流程扩展（中高侵入，安装中/后首启）
1) **适用场景**：调整安装 UI 流程、添加额外交互字段、定制磁盘布局、在落盘后立即运行宿主机服务。  
2) **实现方式**：  
   - 在 `pkg/console/install_panels.go` 添加新面板/校验逻辑，并在 `addInstallPanel` 合并到配置；可写入新的 config 字段并在模板中消费。[F:pkg/console/install_panels.go†L2566-L2717]  
   - 修改 `pkg/console/util.go` 的 `doInstall` 链路插入额外 env 或 pre/post shell（例如在调用 `/usr/sbin/harv-install` 前后调用自研脚本）。[F:pkg/console/util.go†L518-L629]  
   - 在 `pkg/config/cos.go` 的 `after-install-chroot` 阶段渲染中加入自定义模板，实现“首启前植入 systemd”。[F:pkg/config/cos.go†L227-L239]  
3) **最小示例**：  
   - 在 `doInstall` 调用前添加：`execute(ctx,g,env,"/usr/local/bin/pre-flight.sh")`，并通过 `writeFiles` 将脚本打入 installer。  
   - 添加模板 `pkg/config/templates/rke2-95-custom.yaml` 并在 `initRancherdStage` 追加输出。  
4) **构建与验证**：`make build` 或 `USE_LOCAL_IMAGES=true make build`，在 Vagrant 测试 UI；再用 ISO 安装验证脚本执行日志。  
5) **回滚**：git revert 修改，或在 `config` 里关闭新字段解析。  
6) **离线要点**：确保新增脚本在 ISO 中；如需容器镜像，配合方案 4。  
7) **升级兼容性**：中高，需随着上游 UI 与模板改动维护补丁。

### 方案 4：离线制品定制（高侵入，覆盖镜像/Chart/二进制）
1) **适用场景**：将自研服务镜像、Helm chart、二进制随 ISO 一并分发；适配无外网。  
2) **实现方式**：  
   - 将镜像 tar.zst 和对应列表写入 `package/harvester-os/iso/bundle/harvester/images` 与 `images-lists`，在 `metadata.yaml` 登记；`harv-install` 会自动导入这些列表里的镜像。[F:package/harvester-os/files/usr/sbin/harv-install†L178-L318]  
   - 若需要 chart 仓库，扩展 `scripts/package-harvester-os` 在构建阶段将自研 chart/镜像加入 bundle，并调整 `harvester-release.yaml` 记录版本。[F:scripts/package-harvester-os†L33-L111]  
3) **最小示例目录**：  
```
package/harvester-os/iso/bundle/harvester/images/myapp-1.0.0.tar.zst
package/harvester-os/iso/bundle/harvester/images-lists/myapp-images.txt
```
`myapp-images.txt` 内容：`docker.io/acme/myapp:1.0.0`。  
4) **构建与验证**：更新后执行 `make`; 安装完成在目标机 `ctr -n k8s.io images ls | grep myapp`。  
5) **回滚**：删除自定义镜像/列表，重新打包 ISO。  
6) **离线要点**：确保所有依赖镜像都列在 images-lists，否则安装时 `ctr-check-images.sh` 会失败；可在 `harv-install` 增加备用目录。  
7) **升级兼容性**：高，需逐版本对比 upstream `bundle/metadata.yaml` 和镜像列表。

### 方案 5：重写安装脚本或入口（最高侵入）
1) **适用场景**：需要完全自定义安装步骤（特殊分区、替换 Elemental、改变重启逻辑）。  
2) **实现方式**：直接分叉/替换 `/usr/sbin/harv-install`、`system/oem/91_installer.yaml`、`setup-installer.sh`；或在 `doInstall` 里调用自研安装器并跳过 `harv-install`。  
3) **最小示例**：在 `harv-install` 增加自定义分区逻辑并绕过 elemental：  
```
custom_partition() { parted /dev/sda mklabel gpt ...; }
...
custom_partition
# skip elemental
```
4) **构建与验证**：全量重打 ISO；在物理机沙箱安装验证。  
5) **回滚**：恢复 upstream 文件或改回调用 elemental。  
6) **离线要点**：自研安装器需自带所有依赖包/二进制。  
7) **升级兼容性**：极低，每次升级都需手工 rebase。

## 实施路线图
- **Day 0（读代码/跑通 build）**  
  - 阅读 Top10 文件并用 `USE_LOCAL_IMAGES=true make build` 生成本地二进制/ISO 验证构建链。[F:Makefile†L1-L17]  
  - 在 Vagrant 或虚机中以 `DEBUG=true TTY=/dev/tty /vagrant/harvester-installer` 运行 UI（参考 README）。  
- **Day 1（最小化定制 demo）**  
  - 选择方案 1：通过 `configUrl` 注入 `writeFiles` + `afterInstallChrootCommands` 创建自研 systemd 服务，验证随安装自动启动。  
  - 若需前置驱动，添加 `system/oem/05_custom.yaml` 并重打 ISO。  
- **Day 2（离线打包/回归验证）**  
  - 按方案 4 将自研镜像/Chart 加入 bundle，运行 `make` 产出 ISO。  
  - 使用物理机或 KVM 全流程安装，检查 `ctr images ls` 及自研服务启动。  
- **Day 3（上线前 checklist）**  
  - 兼容性：与上游 1.6.1 的 `harv-install`/模板 diff，确保补丁最小化。  
  - 日志：开启 `DEBUG` 验证日志路径可用；在自研脚本中 `set -euxo pipefail` 并输出到 `/var/log`.  
  - 回滚：预留原版 ISO/镜像列表，记录自研改动清单；在 `config` 增加开关以便下发关闭。  
  - 灾备：确认 data disk 未被强制擦除（检查 `WipeAllDisks`/`WipeDisksList` 处理）。[F:pkg/console/util.go†L562-L576]
