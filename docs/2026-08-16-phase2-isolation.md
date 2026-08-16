# 第二批:home 隔离 + 迁移 + 改名实施计划

日期:2026-08-16
仓库:oh-my-tianshu(原 tianshu-public)
目标:与官方 dsh + dsh-tianshu-tui 插件**可同时安装、互不干扰**(用户数据完全隔离);
统一命名消除语义混淆。

## 改动 1:默认 home 独立化(核心)

`packages/util/paths/src/index.ts`:
- `DSH_HOME_DIR_NAME`:`.dsh` → `.dsh-tianshu`
- `DEFAULT_DSH_HOME_DISPLAY` 自动跟随(`~/.dsh-tianshu`)
- 优先级不变:显式配置 > `$DSH_HOME` > 默认(高级用户/CI 兼容)

同步:
- `tests/paths.spec.ts` 断言更新(`.dsh` → `.dsh-tianshu`)
- `README.zh.md` / `README.md`(默认目录描述)
- 其他包硬编码 `.dsh` 的引用清理(settings-local 注释等)
- 重建 `lib/`(index.js 跟仓)

## 改动 2:`migrate-home` 命令

`apps/cli` 新子命令 `migrate-home`:
- 检测:旧默认 home(`~/.dsh`)存在、新默认 home(`~/.dsh-tianshu`)不存在
- 动作:整体复制 `~/.dsh` → `~/.dsh-tianshu`(node:fs cpSync 递归,**不删旧**——保守)
- 输出:迁移清单 + 提示(旧 home 保留,确认后可自行清理)
- 幂等:新 home 已存在时提示并跳过

## 改动 3:命名统一(命令 + npm 包)

仓库名已是 `oh-my-tianshu`(remote 实证)。统一为:
- **bin 命令**:`oh-my-tianshu` → `oh-my-tianshu`(`apps/cli/package.json` bin 字段)
- **npm 包名**:`@huiliyi37/oh-my-tianshu` → `@huiliyi37/oh-my-tianshu`
- 同步:根 package.json scripts(`oh-my-tianshu` → `oh-my-tianshu`)、README/文档中的命令与包名、args.ts 的 help 文案
- 旧包 `@huiliyi37/oh-my-tianshu` 后续 npm deprecate(发布时执行)

## 改动 4:本机落地

- 执行 `migrate-home`(或手动),把当前 `~/.dsh`(已被 oh-my-tianshu 接管)迁到 `~/.dsh-tianshu`
- 恢复官方 `tui` profile:`npx -y @deepseek-ai/dsh plugin --profile tui add @huiliyi37/dsh-tianshu-tui`
- 验证:`oh-my-tianshu tui`(旧)与 `DSH_HOME=~/.dsh-tianshu oh-my-tianshu tui` 各走各的

## 顺序与验证

1. 改动 1 → paths.spec 全绿 + typecheck
2. 改动 2 → cli 测试(若有)+ 手工验证
3. 改动 3 → 全量 typecheck + 测试;**npm 发布与 git 推送前向用户确认**(发布级动作)
4. 改动 4 → 本机执行(动 `~/.dsh` 前说明)
