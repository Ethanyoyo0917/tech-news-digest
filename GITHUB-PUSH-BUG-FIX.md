# GitHub 推送丢失文件问题 & 修复方案

**问题发生时间:** 2026-03-14  
**影响:** 远程仓库只剩下当天1个文件，历史文件全部丢失  
**根本原因:** cron job 使用了错误的 Git 操作逻辑

---

## 问题根因

原代码:
```bash
git clone --depth=1 "https://${GITHUB_TOKEN}@github.com/${REPO}.git" "$TMPDIR/repo"
# ↑ 每次都浅克隆（只取最新1个commit）

cp "$DIGEST_FILE" "digests/${YEAR}/${MONTH}/tech-digest-${DATE}.md"
# ↑ 只复制当天这1个文件！

git add -A
git commit -m "📰 Tech Digest ${DATE}"
git push origin main
# ↑ 用当天文件的临时目录覆盖整个远程仓库
```

**结果:** 每次运行都会创建一个全新的临时目录，只放当天文件，然后 push 覆盖远程。旧文件必然丢失。

---

## 修复方案

### 1. 移除 shallow clone
```bash
# 错误 ❌
git clone --depth=1 ...

# 正确 ✅
git clone ...
```

### 2. 复制所有文件，不只是当天文件
```bash
# 错误 ❌
cp "$DIGEST_FILE" "digests/${YEAR}/${MONTH}/..."

# 正确 ✅
cp "${SOURCE_DIR}"/daily-*.md "digests/${YEAR}/${MONTH}/"
```

### 3. 完整修复后的代码
```bash
GITHUB_TOKEN=ghp_xxx
REPO=owner/repo
DATE=2026-03-15
YEAR=2026
MONTH=03
SOURCE_DIR=/workspace/archive/tech-news-digest
TMPDIR=$(mktemp -d)

# Clone repo WITH full history (NOT --depth=1)
git clone "https://${GITHUB_TOKEN}@github.com/${REPO}.git" "$TMPDIR/repo"
cd "$TMPDIR/repo"

# Create year/month directory structure
mkdir -p "digests/${YEAR}/${MONTH}"

# Copy ALL daily digest files (not just today's)
cp "${SOURCE_DIR}"/daily-*.md "digests/${YEAR}/${MONTH}/"

# Configure git and commit
git config user.email "openclaw-bot@noreply"
git config user.name "OpenClaw Digest Bot"
git add -A
git commit -m "📰 Tech Digest ${DATE}"
git push origin main
rm -rf "$TMPDIR"
```

---

## 教训

1. **永远不要用 `--depth=1`** 用于需要保留历史的仓库
2. **批量操作时，思考是否会覆盖/删除已有数据**
3. **本地先测试 git 操作**，确认不会丢失数据再放到 cron job

---

## 相关文件

- Cron Job ID: `4d7e7af4-5111-40e1-bd3c-636403cbba4a`
- Archive 目录: `/workspace/archive/tech-news-digest/`
-e 

---
Token test: Sun Mar 15 11:58:55 CST 2026
