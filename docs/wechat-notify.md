# Drone 构建结果推送企业微信 - 配置与排错

本文记录在 Drone CI 中对接企业微信群机器人，在构建成功/失败后自动发通知到企业微信的完整流程与常见问题，便于以后重新部署或排查时参考。

---

## 1. 目标

- 构建结束（成功或失败）时，自动向企业微信群发送一条通知。
- 通知内容包含：状态、仓库、分支、构建号、提交人、提交信息、Drone 构建详情链接。

---

## 2. 企业微信侧：创建群机器人并获取 Webhook

### 2.1 创建群机器人

1. **PC 端**（推荐）：打开目标群聊 → 右上角「…」→ 群设置 → **群机器人** → **添加机器人** → 选择 **自定义机器人**。
2. 填写机器人名称（如：`Drone CI 通知`），可选头像。
3. 若手机端提示「无可用机器人」，需企业管理员在 **企业微信管理后台** 开启「群机器人」相关能力；或直接在 PC 端添加。

### 2.2 获取 Webhook 地址

- 添加成功后，会展示 **Webhook 地址**，形如：
  ```text
  https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
  ```
- **复制并妥善保存**，后续会用在 Drone 的请求里；不要泄露到公开仓库。

### 2.3 可选：用 Secret 存储 Webhook（推荐）

- 在 Drone 仓库 **Settings → Secrets** 中新增：
  - **Name**：`WECHAT_WEBHOOK`
  - **Value**：上述完整 Webhook URL
- 流水线里通过 `from_secret: WECHAT_WEBHOOK` 注入，避免把 key 写进 `.drone.yml`。

---

## 3. Drone 侧：添加通知步骤

### 3.1 基本思路

- 在 pipeline 的 `steps` 末尾增加一个 **notify-wechat** 步骤。
- 使用 **无论成功/失败都执行** 的 `when.status`，用 `curlimages/curl` 镜像调用企业微信 Webhook 接口发送消息。

### 3.2 企业微信机器人接口说明

- **请求方式**：POST  
- **URL**：你的 Webhook 地址（即上面拿到的完整 URL）  
- **Content-Type**：`application/json`  
- **Body 示例（文本消息）**：
  ```json
  {
    "msgtype": "text",
    "text": {
      "content": "要发送的文本内容"
    }
  }
  ```
- 官方文档：[群机器人配置说明](https://developer.work.weixin.qq.com/document/path/91770)

### 3.3 完整 notify 步骤示例

下面是一份可直接复用的配置（若使用 Secret，将 URL 改为从 `WECHAT_WEBHOOK` 读取即可）：

```yaml
  - name: notify-wechat
    image: curlimages/curl:8.7.1
    when:
      status:
        - success
        - failure
    commands:
      - |
        if [ "$DRONE_BUILD_STATUS" = "success" ]; then
          MSG="构建成功 状态:$DRONE_BUILD_STATUS 仓库:$DRONE_REPO_NAMESPACE/$DRONE_REPO_NAME 分支:$DRONE_COMMIT_BRANCH 构建号:#$DRONE_BUILD_NUMBER 提交人:$DRONE_COMMIT_AUTHOR 提交信息:$DRONE_COMMIT_MESSAGE 详情:$DRONE_BUILD_LINK"
        else
          MSG="构建失败 状态:$DRONE_BUILD_STATUS 仓库:$DRONE_REPO_NAMESPACE/$DRONE_REPO_NAME 分支:$DRONE_COMMIT_BRANCH 构建号:#$DRONE_BUILD_NUMBER 提交人:$DRONE_COMMIT_AUTHOR 提交信息:$DRONE_COMMIT_MESSAGE 详情:$DRONE_BUILD_LINK"
        fi
        cat <<EOF >/tmp/wechat_payload.json
        {
          "msgtype": "text",
          "text": {
            "content": "$MSG"
          }
        }
        EOF
        curl -sS "你的Webhook地址" \
          -H 'Content-Type: application/json' \
          -d @/tmp/wechat_payload.json
```

### 3.4 可用的 Drone 环境变量（丰富文案时参考）

| 变量 | 说明 |
|------|------|
| `DRONE_BUILD_STATUS` | 构建状态，如 success / failure |
| `DRONE_REPO_NAMESPACE` | 仓库所属命名空间（如 GitHub 用户名/组织） |
| `DRONE_REPO_NAME` | 仓库名 |
| `DRONE_COMMIT_BRANCH` | 触发构建的分支 |
| `DRONE_BUILD_NUMBER` | 构建号 |
| `DRONE_COMMIT_AUTHOR` | 提交人 |
| `DRONE_COMMIT_MESSAGE` | 提交信息 |
| `DRONE_BUILD_LINK` | 当前构建在 Drone 中的详情页链接 |

更多变量见 [Drone 官方文档](https://docs.drone.io/pipeline/environment/reference/)。

---

## 4. 常见问题与解决

### 4.1 YAML 解析报错：`did not find expected comment or line break`

- **原因**：在 YAML 的 `commands` 里，若在双引号字符串中直接换行，YAML 会认为格式非法。
- **解决**：通知内容用**单行**拼进 `MSG`，不要在多行字符串里换行；或改用 `msgtype: markdown` 时注意 JSON 和 YAML 的换行/转义。

### 4.2 企业微信返回：`errcode: 44004, empty content`

- **原因**：请求体里 `content` 实际为空。常见情况是在 `.drone.yml` 里用 `-d "{\"content\":\"${MSG}\"}"` 时，Drone 对 `$` 的转义导致 `${MSG}` 未被 shell 展开，企业微信收到空字符串。
- **解决**：不要在 `curl -d` 里直接拼复杂 JSON + 变量。改为：
  1. 用 shell 先给 `MSG` 赋值；
  2. 用 **heredoc** 把 JSON 写入临时文件（如 `/tmp/wechat_payload.json`），其中 `content` 使用 `"$MSG"`；
  3. 使用 `curl -d @/tmp/wechat_payload.json` 发送。

### 4.3 想用 Markdown 或更多格式

- 企业微信机器人支持 `text`、`markdown` 等类型。把上面示例里的 `msgtype` 改为 `markdown`，并按[文档](https://developer.work.weixin.qq.com/document/path/91770)构造 `markdown.content` 即可。
- 注意：若内容里有 `"`、`\` 等，heredoc 写文件时要注意转义，避免 JSON 非法。

### 4.4 使用 Secret 时的写法示例

- 在 Drone Secrets 里添加 `WECHAT_WEBHOOK`（值为完整 Webhook URL）。
- 在 step 里增加环境变量并用于 `curl`：

```yaml
  - name: notify-wechat
    image: curlimages/curl:8.7.1
    environment:
      WECHAT_WEBHOOK:
        from_secret: WECHAT_WEBHOOK
    when:
      status:
        - success
        - failure
    commands:
      - |
        # ... 同上 MSG 与 cat <<EOF 生成 /tmp/wechat_payload.json ...
        curl -sS "$WECHAT_WEBHOOK" -H 'Content-Type: application/json' -d @/tmp/wechat_payload.json
```

---

## 5. 小结

| 步骤 | 说明 |
|------|------|
| 企业微信 | 群设置 → 群机器人 → 添加自定义机器人 → 复制 Webhook URL |
| Drone | 在 pipeline 末尾加 notify 步骤，用 curl POST 到 Webhook |
| 内容 | 用 shell 变量拼好 `MSG`，再通过 heredoc 写 JSON 文件，最后 `curl -d @file` 发送 |
| 安全 | 建议 Webhook 存 Drone Secret，不在仓库里明文暴露 |

按上述配置后，每次构建结束都会在对应企业微信群收到一条包含状态、仓库、分支、构建号、提交人和详情链接的文本通知，便于以后复现和排错时参考。
