# Git 使用笔记

> 常用 Git 命令速查，结合 STM32 工程开发场景整理。

## 一、基本概念

Git 有四个区域，理解这四个区域就能理解大多数命令：

1. **工作区**（Working Directory）——你正在编辑的文件夹
2. **暂存区**（Staging Area / Index）——准备提交的修改
3. **本地仓库**（Local Repository）——提交历史保存在这里
4. **远程仓库**（Remote Repository）——GitHub / Gitee 等云端仓库

流程：`工作区 → 暂存区 → 本地仓库 → 远程仓库`

## 二、初始配置（新电脑只需做一次）

```bash
git config --global user.name "你的名字"
git config --global user.email "你的邮箱"
git config --list              # 查看当前配置
```

## 三、创建与克隆仓库

```bash
git init                       # 把当前目录初始化成 Git 仓库
git clone <仓库地址>            # 克隆远程仓库到本地
```

## 四、日常提交工作流

```bash
git status                     # 查看工作区状态（哪些文件改了）
git add .                      # 暂存所有修改
git add <文件名>                # 暂存指定文件
git commit -m "提交说明"        # 提交到本地仓库
git push                       # 推送到远程仓库
```

## 五、分支管理

```bash
git branch                     # 查看本地分支
git branch <分支名>             # 创建分支
git switch <分支名>             # 切换分支（推荐）
git switch -c <分支名>          # 创建并切换分支
git merge <分支名>              # 把指定分支合并到当前分支
git branch -d <分支名>          # 删除已合并的分支
```

> 旧写法 `git checkout <分支名>` 也可切换分支，但 `switch` 更清晰、更不易误操作。

## 六、查看状态与历史

```bash
git log --oneline --graph      # 图形化、单行显示提交历史
git diff                       # 查看未暂存的修改
git diff --staged              # 查看已暂存的修改
git show <提交号>               # 查看某一次提交的内容
```

## 七、撤销与回退

```bash
git restore <文件名>            # 撤销工作区修改（回到暂存区/仓库状态）
git restore --staged <文件名>   # 取消暂存（但保留工作区修改）
git reset --soft HEAD~1        # 撤销最近一次提交，保留修改
git reset --hard HEAD~1        # 撤销提交并丢弃修改（危险，慎用）
git revert <提交号>             # 反向提交（安全，会产生新提交）
```

> `restore` 是 Git 2.23+ 推荐用法，旧的 `git checkout -- <文件>` 效果类似。

## 八、远程仓库

```bash
git remote -v                  # 查看远程仓库地址
git remote add origin <地址>    # 关联远程仓库
git fetch                      # 拉取远程更新（只下载，不合并）
git pull                       # 拉取并合并远程更新
git push -u origin main        # 首次推送并建立跟踪关系
```

> `git pull` 大致等于 `git fetch` + `git merge`。

## 九、暂存工作现场（stash）

```bash
git stash                      # 暂存当前未提交的修改
git stash list                 # 查看暂存列表
git stash pop                  # 恢复最近一次暂存（并删除记录）
git stash drop                 # 丢弃最近一次暂存
```

## 十、标签（tag）

```bash
git tag                        # 查看标签
git tag <标签名>                # 打轻量标签
git tag -a <标签名> -m "说明"   # 打带说明的标签
git push --tags                # 推送所有标签
```

## 十一、忽略文件（.gitignore）

在仓库根目录新建 `.gitignore`，把不需要纳入版本管理的文件写进去：

```gitignore
# Keil / MDK 工程生成文件
*.uvguix.*
Objects/
Listings/
DebugConfig/
*.bak
*.o
*.d

# 系统文件
Thumbs.db
.DS_Store

# 编译输出
build/
```

## 十二、常见场景速查

| 场景             | 命令                                 |
| -------------- | ---------------------------------- |
| 改错了想丢弃某文件的修改   | `git restore <文件名>`                |
| 提交了但想撤销（保留改动）  | `git reset --soft HEAD~1`          |
| 提交了想彻底撤销（丢弃改动） | `git reset --hard HEAD~1`          |
| 推送到远程被拒绝       | 先 `git pull`，解决冲突后再 `git push`     |
| 合并冲突了          | 手动改文件 → `git add .` → `git commit` |
| 忘拉代码直接提交       | `git pull --rebase` 再 `git push`   |
| 暂时切分支又不想提交     | `git stash`，回来 `git stash pop`     |


