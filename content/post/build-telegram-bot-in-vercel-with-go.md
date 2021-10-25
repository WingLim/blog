---
title: "使用 Go 在 Vercel 上部署 Telegram Bot"
date: 2021-10-25T14:47:30+08:00
slug: build-telegram-bot-in-vercel-with-go
description: 是机器人！
tags:
- Telegram
- Golang
- Vercel
categories:
- Tech
---

Telegram Bot 提供了另一种交互方式，而且可以通过 Webhook 的方式将 Bot 后端部署在 Vercel 的 Serverless Function 上，来实现一个 Maintenance-free 的服务。

这里以 [ddns-telegram-bot](https://github.com/WingLim/ddns-telegram-bot) 为例，这是一个通知用户 DDNS 变动的机器人。

## 创建并设置 Bot

创建机器人十分简单，只需要四步

1. 打开并启动 [@BotFather](https://t.me/BotFather)
2. 发送 /newbot 命令
3. 输入机器人名称（相当于别名）
4. 输入机器人用户名（相当于唯一标识符）

然后就会获取到 Bot 的密钥

## 初始化项目

安装 Vercel CLI

```bash
# 推荐使用 pnpm
pnpm i -g vercel

# 使用 npm
npm i -g vercel

# 使用 yarn
yarn global add vercel
```

初始化项目并下载依赖

```bash
mkdir project-bot
cd project-bot
go mod init project-bot
go get -u github.com/go-telegram-bot-api/telegram-bot-api
```

## 创建 Serverless Function

使用 Go Runtime 时，Serverless Function 要在 `api` 目录下，文件名即对应路径

```bash
mkdir api
# 路径为 /api/go
touch api/bot.go
```

在 `bot.go` 中，需要定义为 `handler` 包，并且需要包含符合 `HandlerFunc` 签名的导出函数

```go
package handler

import (
  "fmt"
  "net/http"
)

func BotHandler(w http.ResponseWriter, r *http.Request) {
  fmt.Fprintf(w, "<h1>Hello from Go!</h1>")
}
```

在 Telegram Bot Webhook 模式下，用户发送信息给 Bot，Bot 会触发 Webhook，然后 Webhook 返回的响应可以直接作为 Telegram Message 发送给用户

![Message Communication Flow](https://cdn.jsdelivr.net/gh/WingLim/winglim.github.io@hugo/static/images/202110251452862.png)

因此我们可以接收到请求并处理完后，直接返回响应

```go
package handler

import (
	"encoding/json"
	"io/ioutil"
	"log"
	"net/http"

	tgbotapi "github.com/go-telegram-bot-api/telegram-bot-api"
)

// BotResponse 是一个简单的消息响应结构体
// 用于回复对话中的消息
type BotResponse struct {
	Msg    string `json:"text"`
	ChatID int64  `json:"chat_id"`
	Method string `json:"method"`
}

func BotHandler(w http.ResponseWriter, r *http.Request) {
	defer r.Body.Close()
	body, _ := ioutil.ReadAll(r.Body)
	// 将 body 解析为 Update Message
	var update tgbotapi.Update
	if err := json.Unmarshal(body, &update); err != nil {
		log.Fatal("Error to parse Telegram request", err)
	}

	chatId := update.Message.Chat.ID
	text := ""
	// 处理命令
	if update.Message.IsCommand() {
		switch update.Message.Command() {
		case "start":
			text = "Welcome to use DDNS Bot!"
		}

		resp := BotResponse{
			Msg:    text,
			ChatID: chatId,
			Method: "sendMessage",
		}
		
		// 写回响应
		w.Header().Add("Content-Type", "application/json")
		_ = json.NewEncoder(w).Encode(resp)
	}
}
```

到这里，我们给我们的机器人添加了 `/start` 命令的处理，当用户发送 `/start` 到 Bot 时，Bot 会回复 `Welcome to use DDNS Bot!`

## 部署并设置 Webhook

使用 Vercel CLI 进行部署

```bash
vercel
```

打开 [dashboard](https://vercel.com/dashboard) 并复制项目 URL `https://your-subdomain.vercel.app`

使用 `curl`, `wget` 或者浏览器访问下面链接将 Telegram Bot 设置为 Webhook 模式

```bash
# `{token}` 替换为你的 Token
# `{url}` 替换为 `https://your-subdomain.vercel.app/api/bot`

https://api.telegram.org/bot{token}/setWebhook?url={url}
```

返回如下响应即设置成功

```json
{
  ok: true,
  result: true,
  description: "Webhook was set"
}
```

最后部署到生产环境中

```bash
vercel --prod
```

## 结果

打开 Bot，发送 `/start`

![Chat with bot](https://cdn.jsdelivr.net/gh/WingLim/winglim.github.io@hugo/static/images/202110251453515.png)

## 总结

因为 Serverless Function 无状态的原因，通过 Vercel 来部署 Telegram Bot 适合一些不需要依赖上下文状态的功能，举个简单的例子，比如说汇率查询，股价查询等。

但 Vercel 在逐步支持[集成第三方服务](https://vercel.com/integrations)，也可以通过 [Upstash](https://vercel.com/integrations/upstash) 这个 Serverless Redis 来读写状态，如果后续还有进一步的需求的话，会接着再多写两篇文章介绍相关内容。
