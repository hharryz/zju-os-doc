# 浙江大学操作系统课程仓库：文档

本文档适用于 **季江明、王海帅** 老师授课的班级，助教为 **[朱宝林](https://github.com/bowling233)、[张恒斌](https://github.com/hharryz)**。

## 实验要求和分数评定

[浙江大学2025—2026学年校历](https://bksy.zju.edu.cn/2025/0704/c28435a3067627/page.htm)

实验安排及其分数占比（将鼠标悬停在进度条上可查看具体起止日期）：

- 未发布的实验安排可能调整

[gantt month-format="%b"(docs/lab_gantt.yaml)]

- Lab3-5 允许两人组队完成
    - 课程钉钉群将发布组队登记表，请及时填写
    - 助教会帮助将队友添加为仓库成员以便协作，最好能看见两人都有提交记录
    - 请尽可能同时到场验收
    - 组队的同学可以合写一份报告，在报告中说明你们是如何分工合作的，最好能和提交记录对应
- BonusA 和 Bonus B 任选其一，两个都做也只能拿最高的一个分数

实验评分标准：

| 部分 | 占比 | 基本分 | 满分 |
| ---- | ---- | ---- | ---- |
| 代码 | 40% | **提交至 ZJU Git 对应仓库**<br>通过正确性测试和查重 | 编译无 Error 和 Warning |
| 报告 | 50% | **格式：**提交 PDF 文件<br>**内容：**是人写的且人能读懂 | **格式：**推荐使用 LaTeX、Typst、MarkDown 等标记语言编写<br>**内容：**展示出分析实验任务、思考实现方法的过程 |
| 验收 | 10% | 能讲清楚代码在干什么 | （经过提示）能正确回答助教的问题 |

- 报告：提交至学在浙大
- 代码：
    - 提交至 ZJU Git 的对应仓库，请务必按照要求提交，否则可能导致无法自动评分。
    - 每一个实验截止后我们会将对应分支的 Push 权限关闭，因此，请你在完成实验后及时推送到远程仓库的对应分支。
- **补交：**
    - 分数：在截止日期后，每迟一天扣 10%，扣完为止。代码、报告、验收分别计算扣除比例。
    - 代码：补交代码请推送到补交分支，以补交分支最后一次提交为准。
    - 报告：联系助教开通学在浙大的补交通道。

其他：

- **推荐作业：**如果你认为自己的实现有独到之处，可以在报告中说明，或许会被选为推荐作业
- **实验报告建议：**
    - 不用写实验目标、实验环境、实验结果等内容，前者实验文档已经描述清楚，后者环境已经统一，结果也已在测试中体现
    - 欢迎在报告中吐槽实验文档、代码，帮助我们改进实验
    - 尽可能精简，比如不要整片贴代码，展示关键语句即可

鼓励同学们使用 AI **辅助**编程，良好的使用方式有：

- AI 辅助，我主导编写：我自己写大部分代码，只在遇到 bug 或不懂的地方才请 AI 帮忙。
- 用 AI 学习编程概念：我主要用 AI 来学习语法、概念、算法原理，而不是直接写项目。
- 和 AI 协同调试：我让 AI 帮我分析报错信息，提供解决思路，自己来修改代码。
- 代码优化和重构：我会让 AI 帮我检查已有代码的性能、可读性，并提出优化建议。
- 生成文档或注释：我让 AI 帮我为代码写注释、生成文档、补充使用说明。

## 导航

本教学班资源：

| | GitHub | ZJU Git |
| ---- | ---- | ---- |
| 在线文档 | [ZJU-OS/doc](https://zju-os.github.io/doc/) | [os/doc](http://os.pages.zjusct.io/doc) |
| 文档仓库 | [ZJU-OS/doc](https://github.com/ZJU-OS/doc) | [os/doc](https://git.zju.edu.cn/os/doc) |
| 代码仓库 | [ZJU-OS/code](https://github.com/ZJU-OS/code) | [os/code](https://git.zju.edu.cn/os/code) |
| 助教工具 | [ZJU-OS/tool](https://github.com/ZJU-OS/tool) | [os/tool](https://git.zju.edu.cn/os/tool) |

本课程其他教学班：

- 寿黎但：[zju-os-sld/os-25fall](https://git.zju.edu.cn/zju-os-sld/os-25fall)

本校课程：

- [编译原理课程实验](https://git.zju.edu.cn/compiler)
- [编译原理助教工具](https://github.com/ZJU-CP/tools)
- [首页 - 浙江大学 25 春夏计算机体系结构实验](https://zju-arch.pages.zjusct.io/arch-sp25/)
- [计算机系统 系列课程](https://git.zju.edu.cn/zju-sys)

其他：

- [（NJU）操作系统：设计与实现 (2024 春季学期)](https://jyywiki.cn/OS/2024/)
- [(THU) THU CS Lab](https://github.com/thu-cs-lab)
- [(USTC) Vlab 实验中心](https://soc.ustc.edu.cn/)
- [rCore-Tutorial-Book 文档](https://rcore-os.cn/rCore-Tutorial-Book-v3/)

## 历史

操作系统课程实验的历史情况如下：

| 学期 | 情况 |
| ---- | ---- |
| 23 秋冬 | [JuniorSNy/os24fall-stu-full](https://github.com/JuniorSNy/os24fall-stu-full) |
| 24 秋冬 | 部分教学班统一使用 [ZJU-SEC/os24fall-stu](https://github.com/ZJU-SEC/os24fall-stu) |
| 25 秋冬 | 因为各教学班变更方向不同，各自 fork 了 |

25 年秋冬学期，本教学班向 [《编译原理》课程实验](https://git.zju.edu.cn/compiler) 看齐，相比之前的版本、其他教学班主要有以下改动：

- 拆分课程文档和代码仓库
- 学生统一使用 ZJU Git 上分配的仓库进行实验
- 使用容器化的实验环境
- 使用自动化评测

## 致谢

感谢以下各位老师和助教的辛勤付出！

[申文博](https://wenboshen.org/)、[周亚金](https://yajin.org/)、徐金焱、周侠、管章辉、张文龙、刘强、孙家栋、周天昱、庄阿得、王琨、沈韬立、王星宇、朱璟森、谢洵、[潘子曰](https://pan-ziyue.github.io/)、朱若凡、季高强、郭若容、杜云潇、吴逸飞、李程浩、朱家迅、王行楷、陈淦豪、赵紫宸、[王鹤翔](https://tonycrane.cc)、许昊瑞、杨沛山、[朱宝林](https://github.com/bowling233)、[张恒斌](https://github.com/hharryz)。
