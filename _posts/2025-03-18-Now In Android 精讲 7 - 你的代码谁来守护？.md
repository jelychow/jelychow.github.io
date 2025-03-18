---
theme: fancy
excerpt: "excerpt: "Learn how the Now In Android project ensures code quality through CI workflows, dependency management, and automated testing.""

---
## 前言

相信我们经常写代码的同学都会有这样的感觉，团队协作、代码提交、检查变得越来越重要。在某些团队里面，code review 甚至会花出相当大比例的时间与精力来保证代码质量。本文来学习借鉴一下 [now in android](https://github.com/android/nowinandroid) 项目是如何保证稳定性。

## 切入点

在大型项目中，质量保障机制往往深度集成于CI流程。通过分析`.github/workflows/Build.yaml` 配置文件，我们可以系统性拆解nowinandroid项目的质量保障策略。该CI流程包含多个关键任务节点，形成覆盖代码规范、依赖管理、自动化测试等多维度的质量防护网。

### GitHub CI 文件介绍

GitHub 的 CI 文件一般会配置在 `.github` 文件夹里面。今天我们要学习的在 [.github/workflows/Build.yaml](https://github.com/android/nowinandroid/blob/main/.github/workflows/Build.yaml) 这个文件里面。下面我们就来逐步分析里面关于保证代码质量的相关模块吧。

## 项目插件目录检查

```yaml
- name: Check build-logic
  run: ./gradlew :build-logic:convention:check
```

这个任务比较简单，是使用 Android Lint 来校验 `:build-logic` 模块代码质量。由于 `build-logic` 里面会配置一些 convention plugin，包含variant定义、依赖版本控制等关键配置，需优先确保其规范性

## 格式化风格检查

```yaml
- name: Check spotless
  run: ./gradlew spotlessCheck --init-script gradle/init.gradle.kts --no-configuration-cache
```

[init.gradle](https://github.com/android/nowinandroid/blob/main/gradle/init.gradle.kts) 文件详见这里，这个脚本值得我们学习借鉴，建议大家都试一下。首先找个 `src` 目录下面建一个 `test.kt` 文件，然后执行上面的脚本，看看是不是报错了，然后按照它报错的地方进行修改。`./gradlew spotlessCheck --init-script gradle/init.gradle.kts --no-configuration-cache` 这个脚本的用法确实是我第一次见到这么实用的，这脚本可以看成是两步：

1.  配置 Spotless 初始化脚本
2.  执行 `spotlessCheck`

有关 Spotless 的介绍可以看[这里](https://github.com/diffplug/spotless)。首先呢，Spotless check 类似于 Lint 的 check，它会根据配置文件里面的参数来进行 check，如果遇到了规则之外的代码，它会 report 一个错误出来，并且终止当前 task。Spotless 还有一个 apply 任务，一般用于在代码提交之前，用于按照设定的规则对文件进行修改。目前 Spotless 里面配置的 Kotlin 的规则是 ktlint，如果是以纯 Kotlin 开发的项目，建议直接使用 ktlint 插件，使用起来会更简单。对于 now in android 里面这个 `spotlessCheck` 主要作用是校验 Kotlin 格式，以及 xml，kt，kts 的 copyright 的校验。

## 依赖检查

```yaml
- name: Check Dependency Guard
  id: dependencyguard_verify
  continue-on-error: true
  run: ./gradlew dependencyGuard
```

在我们开发过程中依赖库版本变化经常会让我们头疼不已，一不小心一个下午就偷偷溜走。**dependencyGuard** 任务会检查当前依赖库有没有发生改变，如果发生改变，它会终止当前任务。好奇的宝宝可以尝试在项目里面修改一下某个 library 的版本号，然后执行一下 `./gradlew dependencyGuard` 看看会发生什么。当然细心的同学肯定会问，如果我想更新依赖怎么办呢？我们可以通过 `./gradlew dependencyGuardBaseline` 来更新依赖文件。如果想了解更多关于 dependencyguard 的知识请移步至[这里](https://github.com/dropbox/dependency-guard)。

## 重新生成依赖文件，如果上个任务失败，并且这个任务是 PR 触发的

```yaml
- name: Generate new Dependency Guard baselines if verification failed and it's a PR
  id: dependencyguard_baseline
  if: steps.dependencyguard_verify.outcome == 'failure' && github.event_name == 'pull_request'
  run: |
    ./gradlew dependencyGuardBaseline
      
- name: Push new Dependency Guard baselines if available
  uses: stefanzweifel/git-auto-commit-action@v5
  if: steps.dependencyguard_baseline.outcome == 'success'
  with:
    file_pattern: '**/dependencies/*.txt'
    disable_globbing: true
    commit_message: "🤖 Updates baselines for Dependency Guard"
```

有了上一节的知识，这个任务不必多说了。

## 验证截图

```yaml
- name: Run all local screenshot tests (Roborazzi)
  id: screenshotsverify
  continue-on-error: true
  run: ./gradlew verifyRoborazziDemoDebug
```

对于这个任务，我个人认为其偏向于自动化测试方面的。当然，Android 开发如果想了解更多我觉得这是一个不错的知识点。它的介绍可以看[这里](https://github.com/takahirom/roborazzi)。对于某些不经常变更又非常重要的页面，做 screenshot test 是一件极其有意义的事情，它直观地反映出修改，以免在发布之前出现行为不一致。

## 运行本地测试样例

```yaml
- name: Run local tests
  run: ./gradlew testDemoDebug :lint:test
```

这步主要是为了执行一些自定义的 lint 任务，在项目这个 test 主要是调用 `DesignSystemDetector` 这个文件来检查项目里面有没有使用 design system 里面的组件，如果使用了系统自带的有关组件，它会 report 一个 issue 来提醒使用方使用正确的组件。当然，Android Lint 的功能非常强大，远远不止校验组件这一个功能，在实际开发过程中我们可以按照自己需求灵活使用。

## Build 所有变种组合然后上传到指定文件夹

```yaml
- name: Build all build type and flavor permutations
  run: ./gradlew :app:assemble

- name: Upload build outputs (APKs)
  uses: actions/upload-artifact@v4
  with:
    name: APKs
    path: '**/build/outputs/apk/**/*.apk'
```

这个任务堪称整个 CI 的核心，俗称 `丐中丐`，即便我们编写一个最简单的 CI 任务，我们也需要运行这个任务，保证我们上传的代码能够正常地通过编译，同时还要生成产物方便后续 QA 下载测试。

## 上传产物

```yaml
- name: Upload JVM local results (XML)
  if: ${{ !cancelled() }}
  uses: actions/upload-artifact@v4
  with:
    name: local-test-results
    path: '**/build/test-results/test*UnitTest/**.xml'

- name: Upload screenshot results (PNG)
  if: ${{ !cancelled() }}
  uses: actions/upload-artifact@v4
  with:
    name: screenshot-test-results
    path: '**/build/outputs/roborazzi/*_compare.png'
```

上传本地测试任务和截图到 outputs

## 执行 Lint 任务

```yaml
- name: Check lint
  run: ./gradlew :app:lintProdRelease :app-nia-catalog:lintRelease :lint:lint

- name: Upload lint reports (HTML)
  if: ${{ !cancelled() }}
  uses: actions/upload-artifact@v4
  with:
    name: lint-reports
    path: '**/build/reports/lint-results-*.html'

- name: Upload lint reports (SARIF)
  if: ${{ !cancelled() && hashFiles('**/*.sarif') != '' }}
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: './'
```

执行 `lint` 任务并且上传到产物，和 codeql，其中 codeql 是一个静态分析的工具，主要用于检测代码中的安全漏洞和潜在缺陷。

## 检查测试覆盖率

`androidTest` 这个 job 主要作用是使用 `jacoco` 来检查各种 test 的覆盖率，以此来保证尽可能多的场景被测试到。

## 其他措施

### Code Style 配置

Android Studio 项目可以配置 code style，默认目录是在 `.idea` 目录下面，在这里我们可以看到 now in android 项目，也配置自己的 code style。

### Commit Hook

在 `tools` 目录下面有两个 sh 文件，`setup.sh`，`pre-push`，它们的主要作用就是用来 hook commit，在本项目里面这个 hook 的主要目的是：

*   校验 branch 名称
*   检查 WIP or stash
*   执行 `spotless check` 任务，确保 push 的文件格式正确

在团队协作的时候，每个团队可能都会有一些要求，比如提交的 branch、commit 命名需要满足特定的规则，这时候 git hook 就能通过设置规则，使得代码的提交更为规范，尽可能地让代码风格保持一致。

### 巧用 Android Studio 配置

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/9dcee73466264a179befa9a54467e17d~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgQ2FwdGFpblo=:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMzU0NDQ4MTIxNzM4MzkxMSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1742861605&x-orig-sign=%2FpqkeZfQozGrxgjO9oLxqTekgWU%3D)
AS 本身也带有 commit checks，建议需要的小伙伴可以按需选择。

## 总结

## 结语

nowinandroid项目展示了Google Android团队在代码质量管理方面的最佳实践。其CI体系通过分层校验、自动化防护、智能基线管理等手段，构建了完整质量保障生态。建议开发团队结合自身项目特点，选择性引入相关实践，逐步建立适应自身需求的质量保障体系。值得注意的是，任何自动化检查都应服务于开发效率与代码质量的平衡，避免过度校验导致的开发流程僵化。
