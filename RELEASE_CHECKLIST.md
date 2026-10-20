# 发布前清单（实测于 2026-10-20）

这份文件记的是「推 tag 之前必须知道的事实」，每一条都带查证方式。结论先写在最前面：
**三包都已备到「只差推 tag」，而缺口本身不补。**

## 一、三包在 PyPI 与 GitHub 上的实际状态

| 包 | PyPI 最新 | 该版本上传 | 仓库 tag 数 | 本地待发 |
|---|---|---|---|---|
| `torikago` | 1.20.3 | 2026-10-06 13:03 UTC | 42 | 1.20.4（已 bump） |
| `nanodesu` | 1.7.4 | 2026-10-06 17:28 UTC | 19 | 1.7.5（已 bump） |
| `ichiwatashi` | 无（PyPI 返回 404） | — | 0 | 0.1.0 |

**「tag 停在旧版本」这个印象是本地没抓 tag 造成的。** 工作树里原先只有 17 个与 6 个 tag，
而 `git ls-remote --tags origin` 看到 42 个与 19 个；`git fetch --tags` 之后两侧一致。
远端有、本地无的 tag 会让「这个版本发过没有」这个问题在两个终端里得到相反答案。

## 二、tag 与 PyPI 的真正缺口

- `torikago` 打了 tag 但从未上 PyPI：1.9.0、1.10.0、1.11.0、1.12.0、1.13.0、1.14.0、1.14.1、1.14.2（8 个）
- `torikago` 既没 tag 也没发布：1.14.3、1.14.4、1.15.0、1.15.1、1.15.2（5 个）
- `torikago` 在 PyPI 上而仓库里没有 tag：1.3.1
- `torikago` 的 `v0.3.1` 这个 tag，里面的 `pyproject.toml` 写的是 0.3.0
- `nanodesu` 打了 tag 但从未上 PyPI：1.0.0、1.0.1、1.1.0、1.2.0、1.2.1（5 个；发布从 1.3.0 起走 CI）

**这些不补。** PyPI 不接受覆盖，回填等于让四天前的旧代码以一个新版本号永久出现在索引里，
而没有人会去读那个版本的日期。缺口写在这里，比补上更诚实，也是后来人查得到的形式。

## 三、已发布内容与 tag 的一致性核对

做法：从 PyPI 取 sdist，先比它自己的 sha256（证明这份下载没坏），再用 `git archive` 把同名 tag
导出成树，两边逐文件比 sha256。脚本在同目录外的地方，只是工具，不进仓库。

| 包与版本 | 相同文件 | 不同文件 | 结论 |
|---|---|---|---|
| `nanodesu` 1.7.4 | 16 | 0 | PyPI 上那份就是 tag `v1.7.4`（提交 `2f0479b`） |
| `torikago` 1.20.3 | 33 | 0 | PyPI 上那份就是 tag `v1.20.3`（提交 `2ec38c7`） |

差异只有 sdist 自动生成的 `*.egg-info/` 与 `setup.cfg`；反方向缺的是文档、发布记录、`tools/`、
`.github/`，属于打包选择（`MANIFEST.in` 没带它们），不是缺陷。

**于是 §三 那条待查的疑问有了答案**：1.7.4 的发布与本地一致；问题出在 tag 之后又有提交沿用了
同一个版本号 —— `nanodesu` 是 `45bd735`（窗口），`torikago` 是 `c332414`（YARA 测量）与 `5808579`
（窗口）。所以升版本不是可选动作，是发布这件事的前提。

## 四、这一轮发现并修掉的三个缺陷

1. **`--window` 在 `pyz` / `neutralize` / `build` 上被静默接受。** 拒绝检查写在那三个命令的提前
   `return` 之后，所以它从未对它们执行过 —— 这正是那段注释声称已经修好的形状。已把检查移到最前，
   并加测试逐条比对三个命令；实测 `neutralize --window` 现在 `exit 2`，句子是本工具自己的。
2. **独立 exe 里没有面板数据。** 工作流的 PyInstaller 命令行没有 `--add-data`，所以 `--window`
   在这份 exe 上只会退化成打印报告 —— 而 `--help` 照样通过，看不出问题。已加两个 `--add-data`
   与一条 `pyi-archive_viewer` 断言；本地按同一命令行构建实测：`--help` 退出码 0，两个 JSON 都在包内。
3. **发布工作流的三处松口。** `torikago` 的 wheel 检查只点名两个文件（现在含 `panel_window.py`
   与两个数据文件，按后缀匹配，不钉版本号）；`nanodesu` 的清单步骤只打印不断言（现在断言）；
   `nanodesu` 发布前不跑测试（现在跑）。

另外确认了一条此前只是推断的机制：onefile 构建里 `dirname(abspath(__file__))` 就是解包根目录，
`--add-data "panel.json;."` 落点正好是 `embedded()` 先查的位置。

## 五、三份产物与实测证据

| 包 | 产物 | 干净 venv 安装 | 装好后能否自己找到数据 | 渲染 |
|---|---|---|---|---|
| `torikago` 1.20.4 | whl 133.2 KiB · tar.gz 229.6 KiB | Python 3.14.7 | `<venv>/share/torikago` | 1432x770 |
| `nanodesu` 1.7.5 | whl 59.6 KiB · tar.gz 100.3 KiB | Python 3.14.7 | `<venv>/share/nanodesu` | 1432x770 |
| `ichiwatashi` 0.1.0 | whl 116.1 KiB · tar.gz 191.7 KiB | Python 3.14.7 | `panel --export` 后 `panel --verify`：3840 个像素全一致；`font --check`：88 字形，窗口用到的全套字符齐备 | — |

- 测试：`torikago` **491 条**（20 条跳过）全过；`nanodesu` **210 条**全过。
- 家族一致性：`ichiwatashi/tools/check_embeds.py` 报三处嵌入同一 panel（校验和 `974460bf932d728f`），
  三处渲染出的文本行与像素哈希一致（`ef5e97774e08ac7f` / `ee047994ad219d4c`）。
- 「装好后自己找到数据」这一条是特意从**不是仓库的目录**跑起来的：`share/` 那条分支只有
  `pip install` 的人才会走，在克隆里永远走不到。

## 六、发布这一步要做什么（人来做）

`torikago` 与 `nanodesu` 用 Trusted Publishing：推 tag 就触发，仓库里没有任何 token。

1. `torikago`：`git tag v1.20.4 && git push origin v1.20.4`
2. `nanodesu`：`git tag v1.7.5 && git push origin v1.7.5` —— 这一条同时产出 PyPI 包与带 `--window`
   的独立 exe，exe 的窗口数据由 `--add-data` 带进包内，并由 CI 断言
3. `ichiwatashi`：先在 PyPI 建一个 pending publisher（仓库 `Nesarf/ichiwatashi`、工作流
   `release.yml`、environment `pypi`），再 `git tag v0.1.0 && git push origin v0.1.0`
4. 三条共同的前提：工作树干净、`git status` 与远端同步、测试在推 tag 前刚跑过

## 七、明确不做

- **不回填 1.9.0–1.15.2 与 1.0.0–1.2.1。** 理由在 §二。
- **不给 `v0.3.1` 改名。** 已发布的 tag 改名会让所有指向它的引用失效；记录比修复更安全。
- **不在 `--add-data` 之外再塞一份数据。** 解包路径就是资源路径，复制一份只会多一个会飘的副本。
