# Atlas

一个 **Knowrary 知识库（vault）**：只有数据，没有程序。这里放的是我自己的学习笔记与认知图谱——
计算机系统、编译、AI / Agent 这几条线，一个知识点一个 md，节点之间用带类型的关系连起来。

程序在 **[Knowrary](https://github.com/asdbex1078/Knowrary)**：本地跑一个服务，把这些 md
渲染成可拖动的结构图，还能出题、记复习、把文章拆成节点。

## 怎么用

```bash
git clone https://github.com/asdbex1078/Atlas.git
git clone https://github.com/asdbex1078/Knowrary.git && cd Knowrary
python3 -m venv .venv && .venv/bin/pip install -r server/requirements.txt
./server/dev.sh
# 浏览器里打开 http://127.0.0.1:8765/ ，「设置 → 知识库」选中刚才 clone 的 Atlas 目录
```

也可以用 Obsidian 直接打开本目录——它本来就是一个 vault。

## 这是参考库，不是模板

**我会继续往里写**。你要是直接在这个目录里学习、摆位置、记复习，下次 `git pull` 会和我的改动撞上。
两个更省心的做法：

- **只读着看**，看到有用的知识点，用 Knowrary 的「复制到我的库」抄进你自己的库；
- 或者 fork 一份，从此各走各的。

在 Knowrary 里新建一个属于你自己的空库只要一步：「设置 → 知识库 → 选择目录 → 选定并初始化」。

## 目录

| 路径 | 放什么 |
| --- | --- |
| `nodes/` | 知识节点，按领域分子目录（`01-理论基础` … `AI`、`Agent`，`_stubs` 是待补的空壳） |
| `fields/` | 领域总览，每个顶层领域一个文件 |
| `harness/` | 还没拆成节点的学习笔记原文 |
| `assets/` | 节点里引用的图片 |
| `relation-types.json` | 关系类型表：5 个族，具体类型可以自己加 |
| `.knowrary/` | 机器数据：图的摆位、项目清单、教练侧写、历次导入的方案 |

节点怎么写、关系怎么连，见 Knowrary 仓库里的 `doc/规范文档/Markdown文档规范.md`。

**这里不会有密钥**：LLM 配置写在 `.knowrary/llm.local.json`，被 `.gitignore` 挡着；
答题记录、复习流水、对话留档同理——它们是过程，不是知识。
