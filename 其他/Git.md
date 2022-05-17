#### 初始化Git仓库

```shell
git init
```

#### 添加文件到 Git 仓库

```shell
git add <file1> <file2> ...
git commit -m <message>
```

#### 版本回退

```shell
git log //查看提交历史，确定回退版本
git reflog //查看命令历史，确定回到未来哪个版本
git reset --hard commit_id //重置版本
```

#### 删除文件

| 命令                   | 描述           |
| ---------------------- | -------------- |
| `git rm`               | 删除文件       |
| `git checkout -- file` | 丢弃工作区修改 |

#### 分支管理

| 命令                   | 描述                 |
| ---------------------- | -------------------- |
| `git branch`           | 查看分支             |
| `git branch <name>`    | 创建分支             |
| `git switch <name>`    | 切换分支             |
| `git switch -c <name>` | 创建+切换分支        |
| `git merge <name>`     | 合并某分支到当前分支 |
| `git branch -d <name>` | 删除分支             |

#### 关联 github

| 命令                                                       | 描述                           |
| ---------------------------------------------------------- | ------------------------------ |
| `git remote add origin git@server-name:path/repo-name.git` | 关联远程库                     |
| `git push -u origin master`                                | 第一次推送master分支的所有内容 |
| `git push origin master`                                   | 提交修改                       |

#### 远程库管理

| 命令                                                       | 描述                           |
| ---------------------------------------------------------- | ------------------------------ |
| `git pull`                                                 | 从远程抓取分支                 |
| `git remote -v`                                            | 查看远程库信息                 |
| `git push origin branch-name`                              | 从本地推送分支                 |
| `git checkout -b branch-name origin/branch-name`           | 在本地创建和远程分支对应的分支 |
| `git branch --set-upstream branch-name origin/branch-name` | 建立本地分支和远程分支的关联   |

#### 多人协作

1. 首先，可以试图用`git push origin <branch-name>`推送自己的修改；
2. 如果推送失败，则因为远程分支比你的本地更新，需要先用`git pull`试图合并；
3. 如果合并有冲突，则解决冲突，并在本地提交；
4. 没有冲突或者解决掉冲突后，再用`git push origin <branch-name>`推送就能成功！
如果`git pull`提示 no tracking information，则说明本地分支和远程分支的链接关系没有创建，用命令`git branch --set-upstream-to <branch-name> origin/<branch-name>`。