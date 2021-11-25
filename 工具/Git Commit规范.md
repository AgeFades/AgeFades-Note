
# Git Commit Message

## Message 格式

    <type>[optional scope]: <description>

    [optional body]

    [optional footer(s)]

### type

**必填**。用于表明提交的类型，必须为下列之一：

| type | 说明 |
|---|---|
| feat | 新功能/特性 |
| fix | 修复 bug |
| docs | 只修改了文档相关的内容 |
| style | 不影响代码逻辑/含义，仅对代码风格的变更，如空格、格式化、缩进等 |
| refactor | 重构、调整结构（即不是新增功能，也不是修改 bug 的代码变更） |
| perf | 性能优化相关的变更 |
| test | 测试相关的变更，如添加测试或修改/修正已有测试 |
| ci | 与 CI 配置/脚本相关的变更 |
| revert | 回滚某个更早的提交 |
| build | 构建过程/系统、外部依赖相关的变更，如 webpack、npm、xcodebuild、gulp 等 |
| chore | 其他未对代码或测试相关文件的变更，如依赖、开发工具、辅助工具/库（如文档生成）的变更 |

### scope

**选填**。用于说明 commit 影响的范围或模块，当影响的范围有多个时候，可以使用 `*`。

### description

**必填**。对提交目的的简洁描述。不超过 72 个字。

我们的 GitLab 跟 Jira 是有联动的，与 Jira 或者 GitLab 中的 Issue 相关的 commit，需要将 commit 与其进行关联，语法：

- 关联/引用 Jira 提案（JDY-1 代表提案号）
	- `feat: 实现订单超时未处理的提醒短信 JDY-1`
- 关联/引用 GitLab Issue（\#123 代表 GitLab 中的 Issue ID）
	- `feat: 实现订单超时未处理的提醒短信 #123`

### body

**选填**。body 部分是对本次 commit 的详细描述，可以分成多行。 是对 description 的补充。

### footer(s)

**选填**。footer 目前用于两种情况。

1. 不兼容的变动

    所有不兼容的变动都必须在 footer 区域进行说明，以 `BREAKING CHANGE:` 开头，后面的是对变动的描述，变动的理由和迁移注释。

2. 关闭 Jira 中的提案或者 GitLab 中的 Issue，语法：

    - 关闭 Jira 中的提案
        - `fix: Resolves JDY-1`
        - `feat: Closes JDY-1`
        - `fix: Fixes JDY-1`
    - 关闭 GitLab 中的 Issue
        - `feat: Closes #123`
        - `fix: Closes #123, #456, #789`
        - 其他支持的指令
            - Close, Closes, Closed, Closing, close, closes, closed, closing
            - Fix, Fixes, Fixed, Fixing, fix, fixes, fixed, fixing
            - Resolve, Resolves, Resolved, Resolving, resolve, resolves, resolved, resolving
            - Implement, Implements, Implemented, Implementing, implement, implements, implemented, implementing

## 示例

``` text
- docs(changelog): 添加 1.0.1 更新日志
- chore(deps): 依赖更新
- chore(deps): django 从 3.0.6 更新为 3.0.7
- feat(管理平台): 实现订单锁定机制
- feat(支付): 完成微信支付对接
- fix(注册): 修复手机号校验正则
- style(*): 统一使用 `black` 格式化代码
- refactor(sms): 重构短信发送模块
- perf(report): 优化统计报表性能
- test: 提升覆盖率
- test(登录): 添加登录相关单元测试
- ci: CI 流程优化
- build: webpack 配置调整
```

## 相关工具

- [commitizen/cz-cli: The commitizen command line utility](https://github.com/commitizen/cz-cli)
- [conventional-changelog/commitlint: 📓 Lint commit messages](https://github.com/conventional-changelog/commitlint)

## 示例仓库

- [angular](https://github.com/angular/angular/commits/master)
- [vue](https://github.com/vuejs/vue/commits/dev)

## 参考

- [约定式提交](https://www.conventionalcommits.org/zh-hans/)
- [Angular Commit Message Guidelines](https://github.com/angular/angular/blob/master/CONTRIBUTING.md#commit)
- [GitLab Jira integration | GitLab](https://docs.gitlab.com/ee/user/project/integrations/jira.html)
- [Using Smart Commits - Atlassian Documentation](https://confluence.atlassian.com/fisheye/using-smart-commits-960155400.html)
- [Managing issues | GitLab](https://docs.gitlab.com/ee/user/project/issues/managing_issues.html#closing-issues-automatically)
- [Crosslinking Issues | GitLab](https://docs.gitlab.com/ee/user/project/issues/crosslinking_issues.html)
