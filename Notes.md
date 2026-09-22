# Notes

## 在新设备上安装插件（neorg / luarocks 相关）

### 背景

本配置里只有 `lua/plugins/org.lua` 的 **neorg** 依赖 luarocks。它会通过 rockspec 拉入两个
tree-sitter parser：`tree-sitter-norg` 和 `tree-sitter-norg-meta`。

在 **x86_64 Linux** 上，lazy.nvim 通常能直接从 lumen-oss 的 rocks-binaries 拉到预编译包，
无需任何额外操作，本文可跳过。

在 **aarch64（arm64）** 等没有预编译包的平台上，只能走源码编译，此时会连撞三个坑，表现为
启动 nvim 时这两个"插件"反复安装失败，并最终报：

```
Error in <config>/init.lua:
Too many rounds of missing plugins
```

三个坑分别是：

1. hererocks 生成的 luarocks 配置里**没有 `CXX`**，导致 `tree-sitter-norg` 的
   `src/scanner.cc`（C++）编译命令退化成空串：`sh: 1: -c: not found`
2. lazy.nvim 用 `--tree <私有目录> --deps-mode one` 安装，构建后端
   （`luarocks-build-treesitter-parser*`）被装进一个 **luarocks 自己搜索不到** 的目录，
   报 `module 'luarocks.build.treesitter-parser-cpp' not found`
3. 上述后端会被解析到 **6.x**，而 6.x 要求本机安装 `tree-sitter` CLI，
   报 `Error: 'tree-sitter CLI' is not installed`

以下步骤同时绕开这三个问题。**修复全部落在仓库外**（`~/.luarocks/` 和 nvim 的
`lazy-rocks/hererocks` 目录），所以每台新设备都要重做一遍。

---

### 前置依赖

```bash
# Fedora / RHEL
sudo dnf install -y gcc gcc-c++ make git python3

# Debian / Ubuntu
sudo apt install -y build-essential git python3
```

`gcc-c++`（即 `g++`）和 `python3` 是必须的：前者用来编译 norg parser 的 C++ scanner，
后者被 lazy.nvim 用来构建 hererocks。

---

### 步骤 1：写入 luarocks 用户配置（补 `CXX`）

这一步可以在装 nvim 插件之前先做。

```bash
mkdir -p ~/.luarocks
cat > ~/.luarocks/config-5.1.lua <<'EOF'
-- hererocks 生成的 luarocks 配置里没有 CXX，导致任何含 C++ 源码的 rock
-- （如 tree-sitter-norg 的 scanner.cc）编译失败：sh: 1: -c: not found
variables = {
   CXX = "g++",
}
EOF
```

> 为什么放在 `~/.luarocks/config-5.1.lua` 而不是直接改
> `<lazy-rocks>/hererocks/etc/luarocks/config-5.1.lua`：
> 后者是 hererocks 自动生成的，重建 hererocks 时会被覆盖；前者是 user config，
> luarocks 每次启动都会合并读取，且不会被覆盖。

---

### 步骤 2：首次启动 nvim（让 lazy 克隆插件并构建 hererocks）

```bash
nvim --headless "+Lazy! sync" +qa
```

这一步**预期仍会看到 norg 两个 parser 失败**，属正常现象 —— 此时 hererocks 刚被构建出来，
但构建后端还没就位。目的是先把 `lazy-rocks/hererocks` 这套 luarocks 环境准备好。

---

### 步骤 3：把构建后端预装进 hererocks tree

```bash
HEREROCKS="$(nvim --headless -c 'lua io.write(vim.fn.stdpath("data"))' -c 'qa' 2>/dev/null)/lazy-rocks/hererocks"
export PATH="$HEREROCKS/bin:$PATH"

luarocks --tree "$HEREROCKS" --lua-version 5.1 install --force-fast \
  luarocks-build-treesitter-parser-cpp 2.0.6
luarocks --tree "$HEREROCKS" --lua-version 5.1 install --force-fast \
  luarocks-build-treesitter-parser 2.0.0
```

两个要点：

- **装进 hererocks tree**（而不是让 lazy 自己装进私有 tree），因为只有 hererocks tree
  在 luarocks 的 `rocks_trees` 搜索路径上，构建后端才 `require` 得到。
- **版本必须锁在 2.x**。`luarocks-build-treesitter-parser` 的 6.x 需要 `tree-sitter` CLI，
  而 2.x 直接编译仓库里已生成的 `src/parser.c`，无需额外工具链。
  预装 2.0.0 后，lazy 再去解析 `>= 1.3.0` 这个约束时会认为已满足，就不会去下 6.x 了。

> 上面是**已验证可用**的版本组合。若将来 neorg 提高了后端版本要求（rockspec 里的
> `~> 2` 变成 `~> 3` 之类），把版本号去掉改装 latest 再试。

---

### 步骤 4：重新同步并验证

```bash
nvim --headless "+Lazy! sync" +qa
```

日志中应无 `Error` / `Failed installing` / `Too many rounds of missing plugins`。

确认两个 parser 产物已生成：

```bash
find "$(nvim --headless -c 'lua io.write(vim.fn.stdpath("data"))' -c 'qa' 2>/dev/null)/lazy-rocks" \
  -name 'norg*.so' -path '*/lib/lua/*'
```

预期输出：

```
.../lazy-rocks/tree-sitter-norg/lib/lua/5.1/parser/norg.so
.../lazy-rocks/tree-sitter-norg-meta/lib/lua/5.1/parser/norg_meta.so
```

再确认 lazy 认为所有插件都已安装：

```bash
nvim --headless \
  -c 'lua local b={} for n,p in pairs(require("lazy.core.config").plugins) do if not p._.installed then b[#b+1]=n end end print("NOT INSTALLED: "..(#b>0 and table.concat(b,", ") or "none"))' \
  -c 'qa'
```

预期输出 `NOT INSTALLED: none`。

---

### 备选方案：不用 neorg

本配置中只有 neorg 需要 luarocks。如果用不上它，删掉 `lua/plugins/org.lua` 即可，
上述全部步骤都不再需要，也无需 `g++` / `python3` / hererocks。

---

### 排错速查

| 报错 | 原因 | 对策 |
| --- | --- | --- |
| `sh: 1: -c: not found` | luarocks 缺 `CXX` | 重做步骤 1，并用 `luarocks config \| grep CXX` 确认生效 |
| `module 'luarocks.build.treesitter-parser*' not found` | 构建后端不在搜索路径上 | 重做步骤 3，注意 `--tree` 必须指向 hererocks 目录 |
| `Error: 'tree-sitter CLI' is not installed` | 后端解析到了 6.x | 先 `luarocks --tree "$HEREROCKS" remove --force luarocks-build-treesitter-parser`，再按步骤 3 装 2.0.0 |
| `Too many rounds of missing plugins` | 上面任一项失败后 lazy 反复重试 | 修好根因即可自动消失 |
