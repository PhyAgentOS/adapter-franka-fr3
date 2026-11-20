# Franka FR3v2 + 原生 Franka Hand

[English](README.md) | [简体中文](README_zh.md)

本目录接入 **Franka FR3v2 七轴机械臂 + white Franka Hand**。已完成资产归档、
构型契约、原生 MuJoCo 仿真验收，并提供 Franky/Forge 真机驱动、SDK 最小测试、
fake backend 和真机 workflow 骨架。当前仍未连接真机，所有真机验收项保持待现场确认。

> [!WARNING]
> 本项目未来可向真机发送动作，但不是功能安全系统。当前配置中的 Robot IP、
> System Image、Robot Server 版本和 home/reset 位姿均须现场复核。真机动作前
> 必须使用 PREEMPT_RT、保留 Desk/外部急停，并从低速小步开始。

## 当前资产

| 内容 | 路径 | 状态 |
|---|---|---|
| 官方原始生成 URDF/SRDF、参数、LICENSE/NOTICE | `assets/source/franka_description/` | 原样归档 |
| 仓内自包含 URDF/SRDF | `assets/urdf/` | 仅改 mesh URI |
| FR3v2 + white hand mesh | `assets/meshes/` | 官方文件原样归档 |
| 派生 MJCF | `assets/mjcf/scene.xml` | 可独立加载，7 arm + 总开口 gripper |
| GLB | `assets/glb/` | 转换脚本已准备；等待外部 `robot_asset_pipeline` |
| 来源清单与哈希 | `assets/SOURCE_MANIFEST.json` | 可机器校验 |

权威来源为：

- 仓库：<https://github.com/frankarobotics/franka_description.git>
- tag：`2.9.0`
- commit：`7aeeddc449edf8d62b594f9e36a81da53e7796f9`
- 同步日期：见 `assets/SOURCE_MANIFEST.json`
- 上游 `package.xml` 声明 Apache-2.0；完整上游 `LICENSE` 还保留 BSD-3-Clause
  文本。本仓不删改任何上游 LICENSE/NOTICE，详见
  [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)。

## 构型摘要

- 逻辑顺序：`fr3v2_joint1`～`fr3v2_joint7`、`gripper`。
- arm position/velocity：`rad` / `rad/s`；effort：`N·m`。
- `gripper`：两指总开口，单位 `m`，范围 `0～0.08`。
- URDF 两个 finger joint 各为 `0～0.04 m`，第二指 mimic 第一指。
- 数值关节限位的唯一来源：
  `assets/source/franka_description/robots/fr3v2/joint_limits.yaml`。
- wheel 内的 `data/fr3v2_joint_limits.yaml` 是由该 SSOT 逐字节生成的运行时镜像，
  由测试防漂移，不允许手工修改。
- 自动生成的可读限位表：[`config/generated/joint_limits.md`](config/generated/joint_limits.md)。
- 完整轴方向、零位、home/reset 和 TCP 契约见 [description.md](description.md)。

## 第一套 uv 开发环境

环境使用 Python 3.12，并将 Franky 固定为
`franky-control==2.0.0+libfranka.0.17.0`。该 wheel 来自 Franky 官方
Robot Server 9 索引：

```text
https://timschneider42.github.io/franky/whl/by-robot-server-version/9/
```

选择 Robot Server 9/libfranka 0.17.0 是依据当前已知 System Image 5.8.1
和 `franka参数.md` 中的兼容范围；**尚未替代真机握手验证**。若现场 Robot
Server 不是 9，必须修改 `pyproject.toml` 的 Franky index、重新生成
`uv.lock`，不得忽略版本不匹配异常。

Robot Server 兼容性由 Desk 版本记录和 `franky.Robot` 构造握手共同确认；
`franky.Gripper.server_version` 是独立的 Hand server 版本（当前真机为 `3`），
不能拿它与 wheel 索引中的 Robot Server `9` 比较。

安装和检查：

```bash
cd franka_fr3
uv sync --all-groups --frozen
uv run python -c "from importlib.metadata import version; print(version('franky-control'))"
uv run python scripts/validate_assets.py
PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 uv run pytest -q
```

此环境可做无硬件资产和 API 开发。执行 Franky 真机控制前还必须满足：

1. `uname -a` 显示 PREEMPT_RT；
2. 运行用户具有 `rtprio`/`memlock` 权限；
3. Desk 已核对 Robot Server 版本并启用 FCI；
4. 第一阶段仅连接读状态，不发送动作。

## 资产复现

从已锁定的相邻官方仓库重新同步：

```bash
uv run python scripts/sync_franka_description_assets.py \
  --source ../franka_description
uv run python scripts/generate_contract_docs.py
uv run python scripts/generate_mjcf.py
uv run python scripts/validate_assets.py
```

同步脚本会拒绝错误 commit、非官方 origin 或有本地修改的源仓库。运行后
`SOURCE_MANIFEST.json` 会记录同步时间及所有归档文件的 SHA-256。

后续有 `robot_asset_pipeline` 时生成 GLB：

```bash
ROBOT_ASSET_PIPELINE_ROOT=/absolute/path/to/robot_asset_pipeline \
  bash scripts/build_glb_robot_asset_pipeline.sh
```

GLB 是 MJCF 的派生产物，不作为运动学或限位的 SSOT。

## 仿真验收

原生 MuJoCo sweep 不依赖 Forge Runtime：

```bash
cd franka_fr3
uv run --frozen python scripts/validate_assets.py
uv run --frozen python scripts/run_mujoco_sweep.py
uv run --frozen python -m unittest discover -s tests -v
```

它逐轴执行 `near-min → home → near-max → home`，再执行 Franka Hand
`0.08 → 0.04 → 0 → 0.08 m`，同时覆盖 NaN/Inf、超限、错误顺序、未知关节、
重复 action、快速交替、长稳态和 reset。仿真 workflow 与外部依赖说明见
[`examples/03_workflows/sim_basic_motion/README.md`](examples/03_workflows/sim_basic_motion/README.md)。
检查结果见
[`JOINT_SWEEP_CHECKLIST.md`](examples/03_workflows/simulation/JOINT_SWEEP_CHECKLIST.md)。

## Forge / Dora 驱动

`FrankaFR3Driver` 实现 Forge `BaseRobotDriver`。Dora 节点接受 `tick` 和 sparse
`action`，发布标准 `state`；Franka Hand 通过
`JointCommand(name=["gripper"], position=[开口偏移米值])` 使用同一 action 接口。
默认情况下，`JointCommand.position` 是相对于 connect/recover 时 fresh state 的
偏移，而不是绝对关节位姿。例如启动时 J1 为 `-0.4 rad`，发送 J1 `+0.02` 的绝对
SDK 目标为 `-0.38 rad`；周期性重发仍是 `-0.38 rad`，不会不断累加。

锁定的 Franky 2.0 `JointMotion` 是轨迹级 API。驱动只允许一个 arm motion 和一个
Hand future 同时活动；运动期间的新目标覆盖单个 pending slot，前一运动结束后才提交
最新目标。默认 worker 周期为 `20 ms`、最短 SDK 提交间隔为 `50 ms`，Dora tick
不会无界创建 Franky motion。

所有 action 均 fail-closed：只接受纯 position payload，并在相对偏移换算为 SDK
绝对目标后验证 finite、名称、官方位置限位内侧 `0.05 rad`、相邻目标最大
`0.05 rad` 和 Hand 实测范围。越界不会被
静默裁剪；驱动会停止 arm/Hand、锁存错误，并由调用方显式 recovery 或 reconnect。
velocity/acceleration/jerk 分别通过 Franky `RelativeDynamicsFactor` 限制，默认均为
`0.05`。完整参数见 [`config/robot.example.yaml`](config/robot.example.yaml)。

无硬件 Dora/driver 验收：

```bash
PYTHONPATH=src python examples/02_dora_tests/smoke_fake_driver.py
PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 uv run pytest -q \
  tests/test_backend_driver.py tests/test_franky_backend.py tests/test_dora_node.py
```

## 当前边界

- `--backend fake` 是默认且唯一的 CI 后端；3.2 已由 fake/mock 验收，Dora 真机
  连续运行、断线和现场动作仍属于 3.3 L4。
- Franky 驱动和 CLI：见 `examples/01_sdk_tests/README.md`；Dora 节点入口为
  `src/robots_franka_fr3/node.py`，真机 workflow 见
  `examples/03_workflows/real_basic_motion/`。
- `172.16.0.2`、System Image `5.8.1`、FR3v2 Type Label 和实际 Hand
  状态均标为待现场确认。
- MJCF 当前用官方 collision STL 和 finger primitive 兼作可视化，保留的
  DAE 高精视觉资产供后续 GLB/可视化管线使用。
- `robot_asset_pipeline` 不在本仓库或 `forge_runtime/` 中；GLB 目视验收仍需通过
  `ROBOT_ASSET_PIPELINE_ROOT` 显式指定外部工具目录。源码型 Forge Runtime 已位于
  本仓库根目录 `forge_runtime/`，Dora 端到端 workflow 已完成启动、消息交换和停止验证。
- 不包含 Robotiq 2F85、相机或 RealSense 资产。

## 许可证

本仓自有代码、配置和文档采用
[Apache License 2.0](LICENSE)。机器人描述资产源自
`franka_description`（Apache-2.0，Franka Robotics GmbH，BSD 例外条款见
[NOTICE](NOTICE)），运行时依赖 `franky-control` 为 MIT；完整归属见
[NOTICE](NOTICE) 和 [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)。
Apache-2.0 不授予 Franka 商标的使用权。

## 分版本发布

**发布规则（强制）：** 合并版本/锁文件修改 → 在已审查提交上创建 annotated 版本标签 → 从该标签精确构建 → 校验产物 → 发布。禁止使用移动分支、未打标签的提交构建 Release，禁止覆盖已发布标签。

| 维护线 | 源码 | 节点/包版本 | 运行时 |
| --- | --- | --- | --- |
| Forge 1.x | `stable/1.x` | `1.x`，标签 `v1.X.Y` | Forge 1.x / Dora 0.4.x |
| Forge 2.x | `main` | `2.x`，标签 `v2.X.Y` | Forge 2.x / Dora 1.x |

**旧版版本号统一：** `stable/1.x` 原为 0.x 的节点以 **1.0.0** 发布，保留 Forge 1.x / Dora 0.4.x 依赖与运行行为。已有 1.x 版本保留。节点/包主版本与 Forge 兼容线一致，不再接受 0.x 发布标签。

两条线均从 `main` 运行 **Build versioned release**。先将版本和锁文件修改通过 PR 合并至对应源码分支，再在已审查提交上创建未占用的 **annotated** 标签，标签必须匹配包元数据。禁止移动已有标签或复用已发布版本。工作流使用自身已审查的发布工具与精确标签源码，旧标签无需包含新发布脚本。

```bash
# 替换为已存在、已审查、未发布且与源码元数据一致的标签。
TAG=v1.0.0
gh workflow run release.yml --ref main -f release_line=1.x -f tag="$TAG"
TAG=v2.X.Y
gh workflow run release.yml --ref main -f release_line=2.x -f tag="$TAG"
```

流水线校验标签类型、源码提交、分支归属、源码版本及锁定的 Forge/Dora 兼容性；构建源码归档及项目支持的 wheel/sdist，检查 Python 分发元数据与 SHA-256，并创建或附加至 **GitHub Release 草稿**。不覆盖已发布 Release 或产物；发布前检查全部构建结果与验收依据。该流程不发布节点包至 PyPI。

设置 `build_binaries=true` 复用已有 Ubuntu 20.04 二进制工作流，保留原架构、CLI 和归档校验。也可直接运行 `build-ubuntu20.yml`，传入 `ref`、`version`、`release_line`；`ref` 必须为 annotated 标签，不能为移动分支。仅在所有选定任务通过后附加二进制产物。

源码归档保留标签中的文件；外部 SDK/submodule、模型下载与现场配置仍是独立前置条件。依赖或 SDK 构建失败不能视为发布验收通过。构建成功不代表单节点或串联 `dora run`、真机、真实模型、长稳态或回滚验收通过；每条验收链必须包含相同兼容线的 `adapter-forge-gateway`。Forge 2.x 属于 **breaking change**，整个部署的 Forge 库、节点版本与 Dora 必须对齐；1.x 保留在维护分支。

## 依赖维护

本维护分支使用 Forge Msgs 1.3.1 与 Pillow 13，保持 Forge 1.x、Dora 0.4.x 兼容；依赖漏洞审计继续启用。
