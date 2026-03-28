# dronetest

用于测试 Drone CI/CD 的最简 Golang 项目。

## 本地运行

```bash
go run .
go test -v ./...
go build -o app .
```

## CI/CD

- 推送代码到 `main` 或 `master` 分支，或发起 PR 时触发流水线。
- 流水线步骤：构建并推送 Docker 镜像，按分支条件部署（无单独 CI 测试步骤）。

## 使用前

1. 在 Drone 中激活该仓库。
2. 确保仓库根目录有 `.drone.yml`（已包含）。
3. 若 `go.mod` 里 `module` 路径与真实 GitHub 仓库不一致，可改为你的仓库路径，例如：`module github.com/你的用户名/dronetest`。
