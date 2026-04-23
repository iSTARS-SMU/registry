# engine-registry/ — Engine 注册清单

**核心仓对 engine 的唯一引用**。只放 yaml 清单,**不放任何代码**。

> **Bundled fake-engine fixtures only.** 真正的部署 registry 在
> [`iSTARS-SMU/registry`](https://github.com/iSTARS-SMU/registry) ——
> 学生提交的 engine yaml PR 进那个仓,**不进这个目录**。这里只保留
> `fake-recon` / `fake-report` / `fake-llm-planner` 等 thin-slice demo +
> contract-test 用的 fixture,确保 `git clone` 后跑测试零额外步骤就能过。
>
> 部署时切换到 org registry:
> ```
> git clone git@github.com:iSTARS-SMU/registry external/registry
> TRUSTCHAIN_REGISTRY_DIR=./external/registry bash trustchain/scripts/compose.sh up
> ```
>
> 同步本目录的 fake yaml 到 org registry(如果改了的话):
> `bash trustchain/scripts/sync-mirrors.sh registry`

## 结构
每个 active stage 一个子目录(见 [spec §3.2](../../doc/spec.md)):
```
engine-registry/
├── recon/
│   └── targetinfo-agent.yaml
├── weakness-gather/
│   └── exa-cve-search.yaml
├── attack-plan/
│   └── llm-planner.yaml
├── exploit/
│   └── autoeg.yaml
└── report/
    ├── docx-pro.yaml
    ├── sarif.yaml
    └── json.yaml
```

0.1 reserved stage `verify/` 不建目录(等 0.2)。

## 单个 yaml 的字段

完整字段表见 [spec §4.1](../../doc/spec.md)。一个清单的最小样子:

```yaml
id: targetinfo-agent
version: 1.0.3
stage: recon
entry: http://recon-targetinfo-svc:9600
source_repo: https://github.com/student-a/recon-targetinfo
image: ghcr.io/trustchain/recon-targetinfo:1.0.3

capabilities:
  destructive: false
  network_egress: true
  uses_llm: true
  writes_artifacts: true
risk_level: low

timeout_default: 600
resource_profile: {memory_mb: 512, cpu: 0.5}

required_tools: [nmap, feroxbuster]
optional_tools: [nuclei]

secret_requirements:
  - name: openai
    required: true

artifact_types: [screenshot, raw_output]

input_schema:  {$ref: "trustchain-contracts://recon/input"}
output_schema: {$ref: "trustchain-contracts://recon/ReconOutput"}
config_schema:
  type: object
  properties:
    depth: {type: integer, minimum: 1, maximum: 5, default: 2}
  required: [depth]
```

## 怎么加新引擎

学生视角(见 [engine-author-guide.md §7](../../doc/engine-author-guide.md)):
1. Engine 代码在自己私有仓跑过测
2. 构建 image push 到镜像仓
3. 向 `trustchain-core` 提 PR,**只加一个文件**:`engine-registry/<stage>/<name>.yaml`
4. 核心 CI 跑契约测试,过了合 PR

## Registry 服务怎么用这些 yaml
- 启动时扫全目录
- 对每个 yaml 调对应 engine 的 `GET /schema` 校验,不一致标 `mismatch`
- Tool 依赖不 healthy 标 `unavailable`
- 其它 → `available`,进入 UI 下拉
