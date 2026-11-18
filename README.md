# Franka FR3v2 and native Franka Hand

[English](README.md) | [简体中文](README_zh.md)

A seven-axis Franka FR3v2 driver with a native Franka Hand, fake backend, official robot-description assets and MuJoCo examples.

## Driver and hardware setup

Use Python 3.12. `franky-control==2.0.0+libfranka.0.17.0` comes from the official Robot Server 9 wheel index in `pyproject.toml`. Confirm Robot Server compatibility with Desk and the SDK handshake; the independent Hand server version is not the Robot Server version. Hardware operation requires PREEMPT_RT, real-time permissions, FCI and an available emergency stop. Start by reading state. The configured IP, System Image, initial/reset pose and physical limits still need on-site confirmation.

```bash
uv sync --all-groups --frozen
uv run python scripts/validate_assets.py
PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 uv run pytest -q
PYTHONPATH=src python examples/02_dora_tests/smoke_fake_driver.py
```

`FrankaFR3Driver` implements `BaseRobotDriver`. The Dora node accepts `tick` and sparse `action` and emits `state`. `JointCommand.position` represents an offset from fresh connect/recover state, not an accumulating increment or an absolute SDK target. Gripper position is total opening in metres, in `[0, 0.08]`; arm units are rad/rad/s. Commands are position-only, checked for finite values, valid names and configured limits. Invalid commands stop the arm/Hand and latch a fault requiring explicit recovery. Franky trajectory submissions are bounded: one arm motion, one Hand future and one pending replacement. Default worker and SDK submission intervals are 20 ms and 50 ms. See [configuration](config/robot.example.yaml) and [frame, home and TCP contracts](description.md).

## Assets and verification

Official `frankarobotics/franka_description` tag `2.9.0` is pinned in `assets/SOURCE_MANIFEST.json`. `assets/source/` retains original files and notices; `assets/urdf/`, `assets/meshes/` and `assets/mjcf/scene.xml` hold self-contained assets. Joint limits come from the archived official `joint_limits.yaml`; the packaged mirror must remain byte-identical. See [generated limits](config/generated/joint_limits.md).

```bash
uv run python scripts/sync_franka_description_assets.py --source ../franka_description
uv run python scripts/generate_contract_docs.py
uv run python scripts/generate_mjcf.py
uv run python scripts/run_mujoco_sweep.py
```

Asset synchronization rejects an incorrect commit, origin or dirty source tree. GLB generation needs the external `ROBOT_ASSET_PIPELINE_ROOT` and `scripts/build_glb_robot_asset_pipeline.sh`; GLB is not the limit or kinematic source of truth. See [SDK examples](examples/01_sdk_tests/README.md), [simulation](examples/03_workflows/sim_basic_motion/README.md) and [hardware workflow](examples/03_workflows/real_basic_motion/). Fake tests do not establish hardware, disconnect or sustained-motion acceptance. No Robotiq, camera or RealSense assets are included.

## License

Project code, configuration and documentation use [Apache-2.0](LICENSE). Preserve upstream notices and BSD exceptions; `franky-control` uses MIT. See [NOTICE](NOTICE) and [third-party notices](THIRD_PARTY_NOTICES.md). No trademark rights are granted.

## Versioned releases

**Mandatory tag-first release rule:** merge the version/lock change, create an annotated version tag on that reviewed commit, build that exact tag, validate the artifacts, then publish. Moving branch names, untagged commit IDs and replacement of published tags are not release inputs.

| Release line | Source | Node/package version | Runtime |
| --- | --- | --- | --- |
| Forge 1.x | `stable/1.x` | `1.x`, tag `v1.X.Y` | Forge 1.x / Dora 0.4.x |
| Forge 2.x | `main` | `2.x`, tag `v2.X.Y` | Forge 2.x / Dora 1.x |

**Legacy normalization:** nodes previously at 0.x on `stable/1.x` release as **1.0.0**, retaining Forge 1.x / Dora 0.4.x dependencies and runtime behavior. Existing 1.x package versions are retained. Node/package majors now follow the Forge compatibility line; a 0.x tag is rejected.

Run **Build versioned release** from `main` for either line. First merge the version/lock changes into the selected source branch, then create an unused **annotated** tag on that reviewed commit. The tag must match package metadata. Never move existing tags or reuse a published version. Dispatch uses the workflow's reviewed tooling with the exact tagged source, so legacy tags need not contain the new release helper.

```bash
# Replace with an existing, reviewed and unpublished tag matching source metadata.
TAG=v1.0.0
gh workflow run release.yml --ref main -f release_line=1.x -f tag="$TAG"
TAG=v2.X.Y
gh workflow run release.yml --ref main -f release_line=2.x -f tag="$TAG"
```

The workflow verifies tag type, source commit, branch ancestry, source version and locked Forge/Dora compatibility. It builds source archives and wheel/sdist where supported, checks Python distribution metadata and SHA-256, and creates or attaches to a **draft GitHub Release**. Published releases/assets are not overwritten; review all build results and acceptance evidence before publishing. This workflow does not publish node packages to PyPI.

Set `build_binaries=true` to call the existing Ubuntu 20.04 binary workflow; its existing architecture, CLI and archive checks remain in force. You can also run `build-ubuntu20.yml` directly with `ref`, `version` and `release_line`; `ref` must be an annotated tag, not a moving branch. Binary artifacts are attached only after all selected jobs succeed.

The source archive preserves the tagged tree; external SDKs/submodules, model downloads and site configuration remain separate prerequisites. A dependency or SDK build failure is not successful release acceptance. Build success does not establish single-node or Gateway-chain `dora run`, hardware, real-model, sustained-run or rollback acceptance. Every acceptance chain must include `adapter-forge-gateway` from the same compatibility line. Forge 2.x is a **breaking change**: align Forge libraries, node versions and Dora across the entire deployment; keep 1.x on its maintenance branch.
