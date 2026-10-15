# SQLite 数据库分卷与迁移

Git 当前版本保存三个数据库的二进制分卷，每卷最多 **48 MiB（50,331,648 字节）**。
分卷不是独立 SQLite 文件，使用前必须还原。无需安装额外 Python 包，支持 Python 3.11、3.13。

```text
data/
  articles.db.parts/manifest.json, part-00000, ...
  nat.db.parts/manifest.json, part-00000, ...
  preprint.db.parts/manifest.json, part-00000, ...
  UPDATED_AT.txt
scripts/db_volumes.py
```

还原后仍使用 `data/articles.db`、`data/nat.db`、`data/preprint.db`。
所有表、索引、文章 ID、`sqlite_sequence`、抓取状态和补全缓存均保留。
完整数据库及 WAL、SHM、journal 不提交 Git；不使用压缩、Git LFS 或逻辑分库。
分卷解决单文件限制；已有 Git 历史及其体积不变。

## 使用与更新

在仓库根目录执行：

```bash
# 新 checkout：还原全部三个库。缺卷或损坏时返回非零退出码。
python3 scripts/db_volumes.py restore

# 单库还原；--database 可重复指定。
python3 scripts/db_volumes.py restore --database preprint.db

# 仅验证分卷、完整文件 SHA-256 和 SQLite quick_check，不覆盖工作库。
python3 scripts/db_volumes.py verify

# 抓取仓库：先还原，再照常 ingest，结束后打包。
bash fetch_run_once.sh
python3 scripts/db_volumes.py pack
```

`fetch_run_once.sh` 会在缺少工作库时先还原；已有本地工作库会保留。
直接调用任意 `app.ingest*` 入口时，仍需先执行还原，避免 SQLite 自动创建空库。
`pack --database preprint.db` 可只更新 preprint 分卷。打包通过 SQLite backup API
生成一致性快照，包含 WAL 中已提交的数据，不需要删除 WAL 或手动拷贝数据库。
请串行运行本地抓取、打包、Git 更新；两个 Actions workflow 已共享同一并发组。

还原默认拒绝覆盖内容不同的工作库。拉取远程分卷更新后，如果确定要丢弃本地工作库：

1. 先关闭所有使用这些库的进程和 SQLite 连接。
2. 本地有需保留的修改时，先打包或备份。不要直接删除仍含数据的 WAL。
3. 如有遗留 sidecar，先通过 SQLite 正常关闭连接、完成 checkpoint 并确认其已清除。
4. 再执行 `python3 scripts/db_volumes.py restore --force`。

`--force` 也不会绕过 sidecar 检查。工具无法可靠检测外部进程中所有空闲连接，
关闭所有连接是调用方的责任。完整文件校验一致时还原可重复执行；不同内容必须显式覆盖。

也可还原到独立目录，避免影响当前工作库：

```bash
python3 scripts/db_volumes.py restore --data-dir ./data --output-dir /tmp/literature-restored
```

每个库先写临时文件，所有校验通过后才替换目标；失败不会覆盖该库原有文件。
三个库依次处理，不提供跨库事务。需要整组切换的服务应还原到全新的暂存目录，
全部成功后再发布。分卷缺失、截断、SHA-256 不匹配、非法清单、未知格式版本、
目录内存在多余文件或 SQLite 检查失败都将阻止还原和发布。

## 清单格式 v1

每个 `manifest.json` 包含：

| 字段 | 含义 |
| --- | --- |
| `format_version` | 当前为整数 `1` |
| `database` | 原数据库文件名，限三个约定名称 |
| `size_bytes`、`sha256` | 完整数据库快照的字节数和 SHA-256 |
| `chunk_size_bytes` | 默认 `50331648`；每卷大小上限 |
| `parts` | 按顺序排列的 `{name, size_bytes, sha256}` |

卷名从 `part-00000` 开始连续编号。最后一卷可较小，整卷边界不会额外生成空卷。
工具按清单顺序拼接并验证，不依赖目录枚举顺序；拒绝路径穿越和符号链接。
清单不包含生成时间，同一快照重复打包内容相同。成功更新会移除不再需要的旧卷。

## Actions 与首次切换

抓取 workflow 依次还原三个库、运行抓取、打包、校验并提交私有分卷，再发布公开分卷。
某来源抓取失败仍保存其他成功数据，发布后才报告失败；还原或打包失败直接停止发布。
作者补全 workflow 仅还原、补全、打包 `preprint.db`。

公开发布只同步清单列出的分卷、`UPDATED_AT.txt`、本工具、本文档及必要忽略规则，
不会复制抓取目录里的压缩包或备份。首次切换会停止跟踪公开仓库原来的三个完整数据库。
即使首次由作者补全 workflow 发布，也会先迁移全部三个库；此后只更新 preprint，
保留另外两库。元数据中的数据库大小指还原后的大小，并列出卷数。

首次切换不等待下游适配。旧的“拉取后直接读取 `.db`”和“从 Git 提取 `.db`”操作
需要按下面的说明修改，否则不能更新数据。历史 commit 仍可按原格式读取。

## 下游摘要程序

在每次 clone/pull 后、打开 SQLite 之前增加还原步骤：

```bash
git -C literature-data-public pull --ff-only
# 必须先关闭旧库的所有连接。存在本地修改时先备份，不要盲目 force。
python3 literature-data-public/scripts/db_volumes.py restore \
  --data-dir literature-data-public/data --force
# 然后按原来的三个 .db 路径运行摘要或翻译。
```

不需要修改 SQL、`source_name` 或文章 ID 映射。建议下游使用独立的、只读消费的数据目录。

## 网站固定 commit / release 发布

网站现有更新脚本直接执行 `git show <commit>:data/articles.db` 等操作，必须改为提取分卷。
一次发布先固定一个完整 commit SHA，并从该 SHA 读取所有清单和它们列出的分卷。
不能分别从不断变化的 `main` 下载这些文件，以免混入不同版本。

建议在全新的临时 checkout 中提取固定版本，再还原到网站的 release 暂存目录：

```bash
# $literature_commit 必须是此次发布已记录的完整 SHA；$checkout_dir 是不存在的新路径。
git -C paper-info-database worktree add --detach "$checkout_dir" "$literature_commit"
python3 "$checkout_dir/scripts/db_volumes.py" restore \
  --data-dir "$checkout_dir/data" --output-dir "$release_staging_dir"
# 保留现有 schema/行数/摘要关联校验，全部成功后再切换 current release。
# 发布结束并关闭连接后，用 git worktree remove 清理这个临时 checkout。
```

还原后 release 中仍是原来的三个完整数据库。网站 API 和查询代码无需读取分卷。
如果继续使用 `git show`，必须从同一个 SHA 读取清单、按清单安全提取所有卷，再调用工具还原；
不能只修改原来三个 `.db` 的路径。本文不修改下游仓库的程序。
