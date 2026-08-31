# Fcitx5 拼音词库配置教程

本教程基于 Fcitx5 拼音输入法，教你如何安装与启用这三份词库。
词库为 Fcitx5 拼音模块可直接加载的二进制 `.dict` 格式（由 `libime` 生成），**无需转换**，拷贝到位重启即可生效。

## 词库说明

| 文件 | 说明 |
| --- | --- |
| `custom-pinyin.dict` | 自用扩展拼音词库（含常用成语、人名、地名、词组等） |
| `zhwiki.dict` | 基于维基百科条目的超大词库，覆盖大量专有名词、术语、人名地名 |
| `web-slang.dict` | 网络流行语 / 键盘缩写词库 |

## 安装方法

Fcitx5 会自动加载词典目录下的所有 `.dict` 文件，按以下任一方式放置：

### 方式一：仅当前用户生效（推荐）

```bash
mkdir -p ~/.local/share/fcitx5/pinyin/dictionaries/
cp custom-pinyin.dict web-slang.dict zhwiki.dict ~/.local/share/fcitx5/pinyin/dictionaries/
```

### 方式二：全局生效（需要 root）

```bash
sudo mkdir -p /usr/share/fcitx5/pinyin/dictionaries/
sudo cp custom-pinyin.dict web-slang.dict zhwiki.dict /usr/share/fcitx5/pinyin/dictionaries/
```

### 重启输入法

```bash
fcitx5 -r
```

> 若词库已在运行中新增，`fcitx5 -r` 即可热重载；个别环境需注销重新登录。

## 常用配置（`~/.config/fcitx5/conf/pinyin.conf`）

可开关云拼音、候选词翻页、模糊音等（配置修改后同样重启生效）：

```ini
# 启用云拼音
CloudPinyinEnabled=True
# 启用预测
Prediction=False
# 页大小
PageSize=9
# 启用 Unicode 拓展区 B 字符
ExtBEnabled=True
```

按需在 `[Fuzzy]` 里开启模糊音，例如常见错误分组：

```ini
[Fuzzy]
# ue -> ve
VE_UE=True
# c <-> ch
C_CH=False
# z <-> zh
Z_ZH=False
# u <-> v
V_U=False
```

## 验证

安装后重新打开任意输入界面，输入 `zhonghua` ，候选应出现维基词库中的「中华」扩展词条；输入网络用语（如 `yyds`、`awsl`）应能直接上屏。

## 进阶：转储为文本（可选）

如需审查或编辑词库，可用 `libime` 工具转出为可读文本（本教程默认不转换）：

```bash
libime_pinyindict -d custom-pinyin.dict custom-pinyin.txt
# 文本格式：汉字 拼音 频次
# 修改后可重新转回二进制：
libime_pinyindict custom-pinyin.txt custom-pinyin.dict
```

> 注意：重新生成 `.dict` 前请备份原文件。
