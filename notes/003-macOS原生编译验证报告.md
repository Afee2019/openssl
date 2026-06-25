# macOS 原生编译验证报告

## 1. 验证结论

这份 OpenSSL 仓库可以在当前这台 macOS 上原生编译通过，不需要 Docker、不需要 Rosetta，也不需要交叉编译。

我实际执行的是：

```sh
cd /Users/shawn/dev/jcmail-wksp/openssl
./Configure darwin64-arm64-cc no-tests shared
make -j8
```

结果是：

- `Configure` 成功
- `make` 成功
- 生成了 `libcrypto.4.dylib`
- 生成了 `libssl.4.dylib`
- 生成了 `providers/legacy.dylib`
- 生成了 `apps/openssl`

随后我补全测试配置并执行了完整验证：

```sh
./Configure darwin64-arm64-cc shared
make -j8 test
```

结果是：

- `make test` 成功
- `run_tests` 最终结果为 `PASS`
- `Files=371`
- `Tests=4373`
- `117 wallclock secs`

## 2. 验证环境

当前机器信息如下：

| 项目 | 值 |
|---|---|
| CPU 架构 | `arm64` |
| macOS 版本 | `26.5.1` |
| 编译器 | `Apple clang version 17.0.0` |
| Perl | `5.34.1` |

这组环境满足 OpenSSL 的 Unix / macOS 构建要求。

## 3. 最小命令

对这份仓库来说，最小原生编译路径可以写成：

```sh
./Configure darwin64-arm64-cc no-tests shared
make -j8
```

说明：

- `darwin64-arm64-cc` 是 Apple Silicon 的官方配置目标
- `no-tests` 用于减少验证阶段的额外开销
- `shared` 让构建产出动态库，符合大多数 Postfix 集成场景

如果你要的是更极简的“只确认能编过”，这就是可落地的最短路径。

## 4. 实测耗时

这次实际耗时如下：

| 阶段 | 耗时 |
|---|---:|
| `Configure` | `5.63s` |
| `make -j8` | `23.44s` |
| `Configure darwin64-arm64-cc shared` | `6.63s` |
| `make -j8 test` | `117s` |

注意这个时间是“从仓库已存在、依赖已就绪”的本机增量验证时间，不是第一次从零装依赖的全链路时间。

## 5. 构建过程观察

这次构建过程中可以看到几个关键事实：

- `Configure` 自动识别了 Darwin / arm64 目标
- 生成了 `Makefile`、`configdata.pm`、`include/openssl/*.h`
- 编译命令使用了 `-arch arm64`
- 最终链接阶段产出了 `libcrypto`、`libssl`、`apps/openssl`

这说明仓库的 macOS 原生路径是完整的，而不是只停留在配置级别。

## 6. 依赖与路径观察

构建日志里出现了：

- `/opt/homebrew/opt/openssl@3/include`
- `/opt/homebrew/opt/openssl@3/lib`

这说明当前环境的工具链搜索路径里存在 Homebrew OpenSSL 3 相关路径，构建系统也把它纳入了 include / link 阶段。

对你的项目而言，这意味着：

- 本机编译是可行的
- 但如果你想要“可复现构建”，最好显式固定依赖前缀
- 不要依赖环境里恰好存在的隐式路径

## 7. 对 Postfix 的实际意义

对 Postfix 来说，这个验证结果说明：

- OpenSSL 不需要外部虚拟机才能编
- 你可以在 macOS 上直接做开发验证
- 适合把它作为本地调试依赖，而不是仅在 Linux CI 中验证

如果你后续要让 Postfix 使用这份自编 OpenSSL，建议显式指定：

- 头文件路径
- 库路径
- `rpath` 或等价运行时路径

## 8. 部署建议

如果你想把这份构建结果用于后续工程集成，建议再做两步：

1. 跑 `make test`
2. 生成一个明确的安装前缀，例如 `--prefix=/Users/shawn/dev/jcmail-wksp/openssl/local`

这样可以把“能编译”与“能稳定被别的项目消费”分开验证。

## 9. 风险说明

当前验证只证明了：

- macOS arm64 原生配置可成功
- `make` 可成功产出主库和 CLI
- `make test` 可完整通过，测试汇总为 `PASS`

它没有覆盖：

- `make test`
- `make install`
- `static` 构建
- 自定义前缀
- FIPS / legacy 关闭策略

如果你要把它作为 Postfix 的正式依赖，后续仍然建议补完整测试和安装验证。

## 10. 最终判断

结论很直接：

- 这份 OpenSSL 仓库可以在当前 macOS 上原生编译
- 最小可行命令就是 `./Configure darwin64-arm64-cc no-tests shared && make -j8`
- 实测成功，且总耗时大约 29 秒

对你现在的 Postfix 集成工作来说，这个依赖是可用的，可以直接进入下一步的链接与集成设计。
