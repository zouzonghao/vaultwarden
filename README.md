# Vaultwarden 自动构建版

本项目是 [Vaultwarden](https://github.com/dani-garcia/vaultwarden) 的 Fork，主要用途是通过 GitHub Actions 自动跟踪上游更新，并为 **Linux x86_64 + Alpine Linux** 编译可直接运行的单文件二进制程序，自动发布到本项目的 GitHub Releases。

> 本项目不是 Vaultwarden 官方仓库，也不提供 Docker 镜像构建。上游项目的功能、许可证和使用限制仍然适用。

## 项目用途

本 Fork 适合以下场景：

- 服务器运行 Alpine Linux 64 位系统；
- 不希望使用 Docker 或 Podman；
- 需要下载一个已经编译好的 Vaultwarden 可执行文件；
- 希望上游 Vaultwarden 更新后自动获得新的 Release；
- 希望使用 musl 静态链接，减少运行时依赖。

构建目标为：

```text
x86_64-unknown-linux-musl
```

构建启用的数据库和功能特性为：

```text
sqlite,mysql,postgresql,enable_mimalloc
```

## 自动同步和发布流程

项目包含两个工作流：

- [`upstream-sync.yml`](.github/workflows/upstream-sync.yml)：定时检查上游 `dani-garcia/vaultwarden` 的 `main` 分支。
- [`release-binary.yml`](.github/workflows/release-binary.yml)：编译并发布 Alpine Linux x86_64 静态二进制。

当检测到上游有新提交时，流程如下：

1. 将上游 `main` 合并到本项目的 `main`；
2. 推送同步后的代码；
3. 创建一个带时间戳的 Tag，例如：

   ```text
   upstream-20260807-143017-a1b2c3d4
   ```

4. 根据该 Tag 启动二进制编译流程；
5. 创建 GitHub Release；
6. 上传以下文件：

   ```text
   vaultwarden-linux-amd64-musl
   vaultwarden-linux-amd64-musl.sha256
   ```

同步工作流默认每小时运行一次，也可以在 GitHub 的 **Actions → Sync upstream → Run workflow** 中手动运行。

## 下载和运行

打开本项目的 [Releases](../../releases) 页面，下载最新 Release 中的：

```text
vaultwarden-linux-amd64-musl
```

在 Alpine Linux x86_64 服务器上执行：

```sh
chmod +x vaultwarden-linux-amd64-musl
./vaultwarden-linux-amd64-musl
```

> 注意：本项目发布的是服务端二进制，不包含 Web Vault 静态文件。要启用 Web Vault，需要另外准备与上游版本匹配的 Web Vault 文件，并将其放入程序默认查找的 `web-vault/` 目录；或者设置 `WEB_VAULT_ENABLED=false`，仅提供 API 服务。缺少 Web Vault 文件时，程序会启动失败。

建议将程序放入专用目录，并创建数据目录：

```sh
mkdir -p /opt/vaultwarden/data
mv vaultwarden-linux-amd64-musl /opt/vaultwarden/vaultwarden
chmod 755 /opt/vaultwarden/vaultwarden
```

Vaultwarden 默认使用 `/data` 作为数据目录。可以通过环境变量指定配置，例如：

```sh
cd /opt/vaultwarden
DATA_FOLDER=/opt/vaultwarden/data \
ROCKET_ADDRESS=0.0.0.0 \
ROCKET_PORT=8000 \
WEB_VAULT_ENABLED=false \
./vaultwarden
```

生产环境建议使用反向代理提供 HTTPS。Vaultwarden Web Vault 需要安全上下文，不能只通过普通 HTTP 直接暴露到公网。

请根据实际需要配置以下内容：

- `DOMAIN`：对外访问地址，例如 `https://vault.example.com`；
- `DATABASE_URL`：数据库连接地址；
- `ADMIN_TOKEN`：管理后台访问令牌；
- SMTP 邮件相关配置；
- 反向代理和 HTTPS 证书。

详细配置请参考上游文档：

- [Vaultwarden Wiki](https://github.com/dani-garcia/vaultwarden/wiki)
- [配置选项](https://github.com/dani-garcia/vaultwarden/wiki/Configuration-overview)
- [HTTPS 和反向代理](https://github.com/dani-garcia/vaultwarden/wiki/Enabling-HTTPS)
- [数据库配置](https://github.com/dani-garcia/vaultwarden/wiki/Using-the-MySQL-Backend)

## 校验下载文件

Release 同时提供 SHA256 校验文件。下载两个文件后执行：

```sh
sha256sum -c vaultwarden-linux-amd64-musl.sha256
```

如果输出类似下面内容，说明文件校验通过：

```text
vaultwarden-linux-amd64-musl: OK
```

也可以检查文件类型：

```sh
file vaultwarden-linux-amd64-musl
```

它应当是 Linux x86-64 的 musl 静态链接可执行文件。

## 上游同步失败或出现冲突

自动同步使用 Git 将上游 `main` 合并到本项目的 `main`。如果本项目没有修改上游同一位置的文件，通常可以自动完成同步。

如果 GitHub Actions 中的 **Sync upstream** 工作流失败，请先打开对应运行记录，查看失败步骤。

### 常见原因一：合并冲突

如果本项目修改了上游也修改过的文件，可能出现合并冲突。尤其要注意：

- `.github/workflows/` 下的工作流文件；
- `Cargo.toml`、`Cargo.lock`；
- `src/` 下的源代码；
- 上游删除而本项目修改过的文件。

处理步骤如下：

1. 将本项目最新代码克隆到本地：

   ```sh
   git clone https://github.com/你的用户名/vaultwarden.git
   cd vaultwarden
   ```

2. 添加上游仓库并获取最新代码：

   ```sh
   git remote add upstream https://github.com/dani-garcia/vaultwarden.git
   git fetch upstream main
   ```

3. 合并上游代码：

   ```sh
   git checkout main
   git pull origin main
   git merge upstream/main
   ```

4. 查看冲突文件：

   ```sh
   git status
   ```

5. 手动编辑冲突文件，保留需要的内容。完成后执行：

   ```sh
   git add <已解决的文件>
   git commit
   git push origin main
   ```

6. 推送成功后，手动进入 **Actions → Release Alpine binary → Run workflow**，重新执行构建。

不要直接执行 `git reset --hard`，除非你确认本地未提交的修改可以丢弃。

### 常见原因二：GitHub Actions 权限不足

请检查仓库设置：

```text
Settings → Actions → General → Workflow permissions
```

需要选择：

```text
Read and write permissions
```

同时确认 Actions 没有被禁用，并允许工作流创建 Tag、触发其他工作流和创建 Release。

本项目的同步工作流需要以下权限：

```yaml
permissions:
  contents: write
  actions: write
```

发布工作流需要：

```yaml
permissions:
  contents: write
```

### 常见原因三：上游暂时不可访问

如果上游仓库、容器镜像仓库或 GitHub 服务暂时不可用，工作流可能失败。此时可以：

1. 等待服务恢复；
2. 在 Actions 页面重新运行失败的工作流；
3. 如果代码已经同步成功，只重新运行 **Release Alpine binary**，不需要再次同步。

### 常见原因四：构建失败

构建失败时，请查看 **Release Alpine binary** 的日志，重点关注：

- Rust 或依赖版本变化；
- musl 目标平台兼容性；
- 上游源码编译错误；
- GitHub Actions 容器镜像无法拉取；
- 构建资源不足。

如果只是构建过程中的临时错误，可以直接重新运行工作流。如果是源码或依赖问题，需要等待上游修复，或在本项目中解决冲突后再构建。

## 手动触发构建

在 GitHub 仓库中进入：

```text
Actions → Release Alpine binary → Run workflow
```

手动运行分支时只会生成 Actions Artifact，不会自动创建 GitHub Release。自动 Release 使用上游同步流程创建的 `upstream-*` Tag。

如需手动创建 Release，可以先创建并推送符合格式的 Tag：

```sh
git tag upstream-$(date -u +%Y%m%d-%H%M%S)-$(git rev-parse --short HEAD)
git push origin --tags
```

## 本地编译

如果不想等待 GitHub Actions，也可以使用 Rust musl 工具链本地编译。需要安装 Rust、musl 编译环境以及项目要求的系统依赖。

构建命令：

```sh
cargo build --release \
  --features sqlite,mysql,postgresql,enable_mimalloc \
  --target x86_64-unknown-linux-musl
```

生成的文件位于：

```text
target/x86_64-unknown-linux-musl/release/vaultwarden
```

## 数据备份和安全

Vaultwarden 保存密码、附件和其他敏感数据。请务必：

- 定期备份数据库和 `/data` 目录；
- 不要将 `ADMIN_TOKEN`、数据库密码等敏感信息提交到 Git；
- 使用 HTTPS 和安全的反向代理；
- 及时关注上游 Vaultwarden 的安全更新；
- 在升级前先备份数据，并保留可回滚的旧版本二进制文件。

本项目不对数据丢失承担责任。使用前请确认你已经建立可靠的备份和恢复方案。

## 项目来源和许可证

本项目基于 [Vaultwarden](https://github.com/dani-garcia/vaultwarden)，遵循上游项目的 [AGPL-3.0-only](LICENSE.txt) 许可证。

Vaultwarden 是 Bitwarden 客户端 API 的非官方兼容实现，与 Bitwarden, Inc. 没有关联。遇到功能问题时，请先确认问题是否由本 Fork 的构建流程导致；Vaultwarden 本身的问题请参考上游项目的 Issue 和 Discussions。
