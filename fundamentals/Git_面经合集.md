# Git 面经合集

> 目标：把远程引用、本地工作区、提交历史三件事分开说，避免只背命令。

---

## 1. git fetch 和 git pull 有什么区别？

**先记结论：**

- git fetch：从远程下载最新提交和引用信息，更新本地的远程跟踪分支；**不改变当前分支、暂不改工作区文件**。
- git pull：先 fetch，再把远程更新整合到当前分支；整合方式通常是 merge，也可以配置或显式指定为 rebase，因此**可能改变工作区并产生冲突**。

假设本地在 feature 分支，远程 main 有新提交：

~~~text
执行 git fetch origin 后：

本地 main：          A---B
远程跟踪 origin/main：A---B---C---D
当前 feature：        A---B---E

只更新了 origin/main；当前 feature、工作区都不动。

执行 git pull（当前分支跟踪 origin/feature）后：
先 fetch，再把 origin/feature 的新提交合进当前 feature。
~~~

### 什么时候用哪个？

| 命令 | 适合场景 | 注意点 |
| --- | --- | --- |
| git fetch origin | 先看看远程有什么变化；准备同步上游；不想立刻改本地代码。 | 获取后用 git log、git diff 比较，再决定如何整合。 |
| git pull | 当前分支明确跟踪某远程分支，且确认要立即同步。 | 可能 merge 或 rebase，先确认工作区干净。 |
| git pull --ff-only | 只允许快进合并；一旦两边都有新提交就失败。 | 适合保护主分支，避免意外产生 merge commit。 |
| git pull --rebase | fetch 后将本地未推送提交 rebase 到远程最新提交后。 | 改写本地提交 ID；不要对共享历史随意使用。 |

推荐日常习惯：先 fetch，查看差异，明确后再 merge 或 rebase。这样冲突和历史变化都更可控。

---

## 2. merge 和 rebase 有什么区别？

两者都是“把另一条分支的更新带过来”，区别是对提交图的处理方式不同。

假设主分支已经从 B 前进到 D，而你的功能分支有 E、F：

~~~text
起点：
main:    A---B---C---D
                  \
feature:            E---F

在 feature 上执行 git merge main：
main:    A---B---C---D
                  \     \
feature:            E---F---M
                         ↑
                       merge commit，保留真实分叉历史

在 feature 上执行 git rebase main：
main:    A---B---C---D
                      \
feature:                E'---F'
                         ↑    ↑
                把 E、F 重新应用在 D 后，提交 ID 会改变
~~~

| 对比 | merge（合并） | rebase（变基） |
| --- | --- | --- |
| 核心动作 | 创建一次合并关系，把两条历史汇合。 | 把当前分支独有提交逐个“重放”到新基线上。 |
| 历史图 | 可能有 merge commit，能保留真实分叉和汇合。 | 历史线性、阅读简洁，但原提交被替换。 |
| 提交 ID | 原有提交 ID 不变。 | 被重放的提交产生新 ID。 |
| 风险 | 历史复杂一些。 | 对已被他人基于其开发的共享分支，会让协作方历史错位。 |
| 常见场景 | 合并共享分支、发布分支、团队要求保留分支关系。 | 整理自己尚未共享的功能分支，使其跟上最新 main。 |

### 冲突怎么处理？

无论 merge 还是 rebase，冲突都要人工判断业务语义，而不只是选“左边”或“右边”。

~~~text
merge：
解决冲突 -> git add <file> -> git commit
放弃本次合并 -> git merge --abort

rebase：
解决冲突 -> git add <file> -> git rebase --continue
跳过一个明确无用的提交 -> git rebase --skip
放弃并回到变基前 -> git rebase --abort
~~~

### 一条重要协作规则

> **已经推送并被别人基于其开发的公共分支，不要随意 rebase。**

因为 rebase 改写提交 ID。确实需要更新自己已推送但尚无人协作的个人功能分支时，推送可使用 git push --force-with-lease；它会先检查远程是否已被别人更新，比强制覆盖更安全。

**面试可直接答：**

> fetch 只把远程变化下载到远程跟踪分支，不动当前代码；pull 是 fetch 加整合，会把变化 merge 或 rebase 到当前分支。merge 保留真实分叉历史，通常会产生合并提交；rebase 把我的提交重放到最新基线后，历史更线性但会改写提交 ID。我会对自己的未共享功能分支用 rebase 保持整洁，对共享分支或需要保留合并关系的场景用 merge；冲突一定按业务语义解决。

