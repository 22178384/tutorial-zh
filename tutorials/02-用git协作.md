# 02 用 git 协作

git 的命令多到吓人，但日常工作真的只用得上几个。这篇讲我怎么用，不是讲 git 的原理。
想懂原理去读 Pro Git，免费的。

## 0. 一次性配置

```bash
git config --global user.name "你的名字"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
git config --global pull.rebase true      # pull 时用 rebase，历史更干净
```

`pull.rebase true` 这个我强烈建议设上。不设的话每次 pull 都可能产生一个没意义的
merge commit，历史会变成一团麻。

## 1. 每天的基本流程

```bash
git switch main
git pull                        # 先同步
git switch -c feature/add-login # 开分支
# ... 改代码 ...
git add -p                      # 交互式选择要提交的改动（推荐）
git commit -m "feat: 加登录接口"
git push -u origin feature/add-login
```

然后去 GitHub/GitLab 上开 Pull Request（PR），等 review。

**`git add -p` 是我最推荐的一个习惯。** 它会一块一块问你要不要加。这样你能把
"加登录"和"顺手改了缩进"拆成两个 commit，review 的人会感谢你。

**`-u`** 只有第一次 push 需要，作用是把本地分支和远程分支关联起来，之后直接
`git push` 就行。

## 2. commit message 怎么写

我们组用 Conventional Commits，格式：

```
<类型>: <简短描述>

可选的详细说明，讲清楚为什么这么改
```

类型就这几个：

- `feat` 新功能
- `fix` 修 bug
- `docs` 只改文档
- `refactor` 重构，不改行为
- `test` 加测试
- `chore` 杂活，比如升级依赖

好例子：

```
fix: 修复分页在最后一页会多请求一次的问题

当 total 刚好是 per_page 的整数倍时，hasNext 仍然返回 true，
导致前端多发一次请求拿到空数组。改成用 offset >= total 判断。
```

**为什么要在正文里写"为什么"**：半年后你或者别人看这行代码，`git blame` 一下
能找到原因。只写"修复分页 bug"是不够的，你得说清楚原来错在哪。

## 3. 冲突怎么办

冲突不可怕，就是 git 不知道你想要哪个版本。

```bash
git switch main
git pull
git switch feature/add-login
git rebase main          # 把 main 的最新改动接到你的分支下面
# 如果有冲突：
```

打开冲突文件，会看到：

```
<<<<<<< HEAD
这是 main 上的版本
=======
这是你分支上的版本
>>>>>>> feature/add-login
```

你要做的是**把这三行标记删掉，留下最终想要的内容**。可能两边都要保留，可能只用
一边，自己判断。改完：

```bash
git add 冲突文件
git rebase --continue
```

**坑一**：rebase 到一半发现搞乱了，想放弃：

```bash
git rebase --abort      # 回到 rebase 之前
```

这个命令救过我好几次，记住它。

**坑二**：**已经在远程被别人拉过的分支不要 rebase**。rebase 会重写历史，别人
再拉就会一团乱。只在**只有你自己用的分支**上 rebase。

## 4. 改错了想撤销

这是新手最慌的场景。分情况：

```bash
# 改乱了，还没 git add，想全部丢弃
git restore 文件名

# 已经 git add 了，想取消暂存（改动还在）
git restore --staged 文件名

# 最后一个 commit 的 message 写错了
git commit --amend          # 会打开编辑器改 message
# 注意：已经 push 过的 commit 不要 amend，除非只有你在用这个分支

# 想撤销最后一个 commit 但保留改动
git reset --soft HEAD~1
```

**危险区**：`git reset --hard` 会真的丢掉改动，找不回来（其实 reflog 能找，但
别指望）。用之前想清楚。

## 5. 看历史

```bash
git log --oneline --graph --all     # 一行一条，带分支图
git log -p 文件名                   # 看某个文件的每次改动内容
git show abc1234                    # 看某个 commit 改了什么
git blame 文件名                    # 每行是谁在哪个 commit 改的
git diff main..feature/x            # 两个分支的差异
```

`git blame` 配合 `git show` 是排查历史的神器。找到可疑的那行，`git blame` 看
是哪个 commit，再 `git show` 看当时为什么这么改。

## 6. 一些保命操作

```bash
git stash              # 临时存起当前改动，去处理别的事
git stash pop          # 拿回来
git stash list         # 看存了哪些

git reflog             # 所有 HEAD 移动记录，救场用
```

`git reflog` 是最后的救命稻草。哪怕你 `reset --hard` 了，只要 commit 过，
reflog 里都能找到那个 commit 的哈希，然后 `git reset --hard <哈希>` 回去。

## 7. 关于 .gitignore

已经提交的文件，后来加到 `.gitignore` 里是没用的，git 还在跟踪它。要先移除跟踪：

```bash
git rm --cached 文件
git commit -m "chore: 停止跟踪 .env"
```

`.env` 这种文件如果误提交了，**光删掉不够**，历史里还在，得改历史（`git filter-repo`）
并且**马上把里面的密钥全部换掉**。这个我踩过，密钥泄露到公开仓库，虽然几分钟内
就改了，但已经被爬虫扫到了。

下一篇：[用 docker 容器化](03-用docker容器化.md)。
