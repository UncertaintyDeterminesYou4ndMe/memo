# gacp —— 个人 Git 提交推送快捷工具

## 背景

日常开发中频繁重复三条命令，且每次只有提交信息不同：

```bash
git add .
git commit -m "提交信息"
git push origin staging
```

目标：封装为一条命令，同时加入确认机制和撤回能力，避免误操作。

---

## 设计思路

| 需求 | 方案 |
|------|------|
| 简化提交流程 | `gacp "message"` 一条命令完成 add + commit + push |
| 防止误操作 | push 前展示变更文件列表，需手动确认 |
| 支持撤回 | `gundo` 自动判断策略，一键撤回上次推送 |
| 未来多人协作 | `-c` flag 启用 pull --rebase，按需开启 |

---

## 安装位置

放在 Oh My Zsh 的 custom 目录，不会被框架升级覆盖，且自动加载：

```
~/.oh-my-zsh/custom/git-shortcuts.zsh
```

新终端自动生效，当前终端手动 source 一次：

```bash
source ~/.oh-my-zsh/custom/git-shortcuts.zsh
```

---

## 完整代码

```zsh
# ─── 颜色定义 ────────────────────────────────────────────────
_GIT_GREEN='\033[0;32m'
_GIT_BLUE='\033[0;34m'
_GIT_YELLOW='\033[1;33m'
_GIT_RED='\033[0;31m'
_GIT_GRAY='\033[0;90m'
_GIT_NC='\033[0m'

# 被视为"共享分支"的分支名 → gundo 自动选 revert 策略
_SHARED_BRANCHES=("main" "master" "staging" "develop" "dev" "production")

function _git_is_shared_branch() {
  local branch="$1"
  for shared in "${_SHARED_BRANCHES[@]}"; do
    [[ "$branch" == "$shared" || "$branch" == release/* || "$branch" == hotfix/* ]] && return 0
  done
  return 1
}

# ─── gacp: Add + Commit + Push ───────────────────────────────
# 用法:
#   gacp "提交信息"          个人模式（默认）
#   gacp -c "提交信息"       协作模式：push 前先 pull --rebase 同步他人变更
function gacp() {
  local collab=0

  if [[ "$1" == "-c" || "$1" == "--collab" ]]; then
    collab=1
    shift
  fi

  if [[ -z "$1" ]]; then
    echo "用法: gacp [-c] \"提交信息\""
    echo "  -c / --collab   协作模式，push 前先 pull --rebase"
    return 1
  fi

  local branch
  branch=$(git symbolic-ref --short HEAD 2>/dev/null)
  if [[ -z "$branch" ]]; then
    echo "错误: 当前目录不在 git 仓库中"
    return 1
  fi

  local mode_label=""
  [[ $collab -eq 1 ]] && mode_label=" ${_GIT_GRAY}[协作模式]${_GIT_NC}"

  echo "${_GIT_YELLOW}分支:${_GIT_NC}    $branch$mode_label"
  echo "${_GIT_YELLOW}提交信息:${_GIT_NC} $1"
  echo "${_GIT_YELLOW}变更文件:${_GIT_NC}"
  git status --short
  echo ""
  echo -n "确认提交并推送? (y/N) "
  read -r _confirm
  [[ "$_confirm" != "y" && "$_confirm" != "Y" ]] && echo "${_GIT_GRAY}已取消${_GIT_NC}" && return 0

  local steps=3
  [[ $collab -eq 1 ]] && steps=4
  local step=1

  echo "${_GIT_BLUE}[$step/$steps] git add .${_GIT_NC}"; (( step++ ))
  git add . || return 1

  echo "${_GIT_BLUE}[$step/$steps] git commit -m \"$1\"${_GIT_NC}"; (( step++ ))
  git commit -m "$1" || return 1

  if [[ $collab -eq 1 ]]; then
    echo "${_GIT_BLUE}[$step/$steps] git pull --rebase origin ${branch}${_GIT_NC}"; (( step++ ))
    if ! git pull --rebase origin "$branch" 2>/dev/null; then
      git rebase --abort 2>/dev/null
      echo "${_GIT_RED}Rebase 冲突，已中止。请手动解决:${_GIT_NC}"
      echo "  git pull --rebase origin $branch"
      echo "  git push origin $branch"
      return 1
    fi
  fi

  echo "$( git rev-parse HEAD) $branch" > ~/.git_last_push

  echo "${_GIT_BLUE}[$step/$steps] git push origin ${branch}${_GIT_NC}"
  git push origin "$branch" && echo "${_GIT_GREEN}完成! 已推送到 origin/${branch}${_GIT_NC}"
}

# ─── gundo: 智能撤回上次 gacp ────────────────────────────────
# 自动判断策略：共享分支 → revert，个人分支 → reset --soft + force-with-lease
# 用法: gundo
function gundo() {
  if [[ ! -f ~/.git_last_push ]]; then
    echo "没有找到上次 gacp 的记录，无法撤回"
    return 1
  fi

  local last_hash last_branch
  read -r last_hash last_branch < ~/.git_last_push

  if ! git cat-file -e "${last_hash}^{commit}" 2>/dev/null; then
    echo "${_GIT_RED}错误: commit $last_hash 不存在，记录已过期${_GIT_NC}"
    rm ~/.git_last_push
    return 1
  fi

  echo "${_GIT_YELLOW}准备撤回:${_GIT_NC}"
  git log --oneline -1 "$last_hash"
  echo "${_GIT_YELLOW}分支:${_GIT_NC} $last_branch"
  echo ""

  local strategy reason
  if _git_is_shared_branch "$last_branch"; then
    strategy="revert"
    reason="$last_branch 是共享分支 → 追加反向 commit，保留历史"
  else
    strategy="reset"
    reason="$last_branch 是个人分支 → 抹除 commit，force-with-lease 推送"
  fi

  echo "${_GIT_BLUE}策略: $strategy${_GIT_NC}  ${_GIT_GRAY}($reason)${_GIT_NC}"
  echo -n "确认撤回? (y/N) "
  read -r _confirm
  [[ "$_confirm" != "y" && "$_confirm" != "Y" ]] && echo "${_GIT_GRAY}已取消${_GIT_NC}" && return 0

  if [[ "$strategy" == "revert" ]]; then
    git revert HEAD --no-edit || return 1
    git push origin "$last_branch" || return 1
  else
    git reset --soft HEAD~1 || return 1
    git push --force-with-lease origin "$last_branch" || return 1
  fi

  rm ~/.git_last_push
  echo "${_GIT_GREEN}撤回完成${_GIT_NC}"
}
```

---

## 用法

### gacp — 提交推送

```bash
# 个人项目（默认）
gacp "修复登录 bug"

# 多人协作项目，push 前自动同步他人变更
gacp -c "修复登录 bug"
```

运行后展示摘要并确认：

```
分支:    feature/login
提交信息: 修复登录 bug
变更文件:
 M src/auth.ts
 M src/login.tsx

确认提交并推送? (y/N)
```

### gundo — 撤回上次推送

```bash
gundo
```

自动判断撤回策略：

```
准备撤回: a1b2c3d 修复登录 bug
分支: feature/login

策略: reset  (feature/login 是个人分支 → 抹除 commit，force-with-lease 推送)
确认撤回? (y/N)
```

---

## 关键知识点

### 撤回的两种策略

| 策略 | 命令 | 适用场景 | 历史 |
|------|------|----------|------|
| revert | `git revert HEAD` | 共享分支（main/staging/develop） | 保留，追加反向 commit |
| reset | `git reset --soft HEAD~1` | 个人分支（feature/*） | 重写，commit 消失 |

`gundo` 根据分支名自动选择，`_SHARED_BRANCHES` 数组可按需增减。

### force-with-lease vs force

- `--force`：无条件覆盖远端，危险
- `--force-with-lease`：检测远端是否有新 commit，有则拒绝，更安全

### 协作模式的 pull --rebase

多 agent / 多人协作时，`-c` flag 会在 commit 之后、push 之前执行：

```bash
git pull --rebase origin <branch>
```

rebase 将本地 commit 接在对方 commit 之后，保持线性历史，避免产生无意义的 merge commit。

### 记录文件 ~/.git_last_push

每次 `gacp` 成功后写入：

```
<commit_hash> <branch_name>
```

`gundo` 读取此文件定位要撤回的 commit，撤回完成后自动删除，防止重复撤回。
