# 关于 replay

## swe-agent 中的 replay

run_replay 读取一个 traj，得到原始的 yaml 配置和 actions 列表，并将 model 设置为 ReplayModel，然后执行正常的 run_single
正常情况下 run_single 调用的是 LiteLLMModel 的 query ，在我们的情况下就是向 bailian API 发起请求，
然而在 run_replay 状态下调用 ReplayModel 的 query，直接返回 traj 中的 action，不发起真正的 API 请求。

注释中提到两种适用场景：

1. 创建 demo（演示的时候就可以不调用 LLM 迅速且确定地重现执行轨迹）
1. 调试工具行为（工具行为可能不符合预期）

这与我们期待的**调试大语言模型行为**目前并不匹配，因为它不支持改变大语言模型的输出轨迹。

## 我们的改动

邓书刚改造的 flow 破坏了原有的结构。

run_single 下面没有 agent 了，取而代之的是 flow。
要想复原原来的行为需要找到 flow 下面的每个 agent 设置它们的 model 为 ReplayModel。

主要是 replay 不符合我们的需求，即使改造为原有的行为意义也不大。

## Hack

我在原来的 SWE-agent 基础上做了一些 hack（当然是 vibe coding）。

主要是创建 HybridModel，如果在执行 run_replay 时指定 --and_continue 则使用之（而不是 ReplayModel），使得在 replay 已有步骤之后继续调用 LLM 完成任务（注意此时的 traj 的 histroy 一定要删除 submit action，因为 submit 是退出的标志）。

我做的测试：

首先 `run` single 生成 traj 文件。

```bash
uv run sweagent run --env.repo.github_url=https://github.com/SWE-agent/test-repo   --problem_statement.github_url=https://github.com/SWE-agent/test-repo/issues/1
```

然后在删除最后的 submit action traj 上以 `--and_continue` flag 运行 `run-replay`。

```bash
uv run sweagent run-replay --traj_path trajectories/allen/no_config__dashscope--qwen3-max__t-0.00__p-1.00__c-0.00___SWE-agent__test-repo-i1/remove_submission/SWE-agent__test-repo-i1.traj --and_continue
```

还测试了仅保留前几个 action 的 traj 上运行的效果，看上去也能正常工作。

```bash
uv run sweagent run-replay --traj_path trajectories/allen/no_config__dashscope--qwen3-max__t-0.00__p-1.00__c-0.00___SWE-agent__test-repo-i1/remove_half_way/SWE-agent__test-repo-i1.traj --and_continue
```

具体代码变更参见 `patch.diff`，对应的 SWE-agent 的 base commit 为 77f0a5ea8849640c87a5d2498a9a089740108710。
