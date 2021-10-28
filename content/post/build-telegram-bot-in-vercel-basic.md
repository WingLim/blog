---
title: "在 Vercel 上部署 Telegram Bot（一）：基本搭建"
date: 2021-10-25T14:47:30+08:00
slug: build-telegram-bot-in-vercel-basic
description: 是机器人！
tags:
- Telegram
- Bot
- Vercel
categories:
- Tech
---

Telegram Bot 提供了另一种用户与服务交互的方式，通过与 Bot 的对话，按钮等方式来简化服务的执行

同时我们可以通过 Webhook 的方式将 Bot 后端部署在 Vercel 的 Serverless Function 上，来实现一个 Maintenance-free 的服务

这里以 [cusdis-telegram-bot](https://github.com/WingLim/cusdsi-telegram-bot) 为例，这是一个通知用户评论，并且可以在 Bot 中通过或回复评论的机器人

## 创建并设置 Bot

创建机器人十分简单，只需要四步

1. 打开并启动 [@BotFather](https://t.me/BotFather)
2. 发送 `/newbot` 命令
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
# Vercel 的类型库
pnpm i -D @vercel/node
# 这是一个 Typescript 的 Telegram Bot 框架
pnpm i grammy
```

## 创建 Serverless Function

Serverless Function 要在 `api` 目录下，文件名即对应路径

```bash
mkdir api
# 路径为 https://your-subdomain.vercel.app/api/bot
touch api/bot.ts
```

在 `bot.ts` 中，需要导出一个默认的函数，接受请求和响应

```ts
import { VercelRequest, VercelResponse } from '@vercel/node';

export default (request: VercelRequest, response: VercelResponse) => {
	const { name = 'World' } = request.query;
	response.status(200).send(`Hello ${name}!`);
};
```

Telegram Bot 有两种模式：
1. Polling
	
	用户需要定期轮询，获取信息并处理
2. Webhook
	
	Bot 收到消息后向用户设置的 Webhook 发送消息

在 Serverless Function 中我们使用 Webhook 的方式来处理请求

然后我们来创建一个简单的命令 `/start`

```ts
// api/bot.ts
import { VercelRequest, VercelResponse } from '@vercel/node'
import { webhookCallback } from 'grammy'
import { bot } from '../index'

bot.command('start', async (ctx) => {
    await ctx.reply('Welcome to use Cusdis Bot')
})

export default async (req: VercelRequest, res: VercelResponse) => {
    webhookCallback(bot, 'http')(req, res)
}

```

到这里，我们给我们的机器人添加了 `/start` 命令的处理，当用户发送 `/start` 到 Bot 时，Bot 会回复 `Welcome to use Cusdis Bot`

## 部署并设置 Webhook

安装依赖 `dotenv`

```shell
pnpm i -D dotenv
```

根目录下添加 `index.js`

```js
// index.js

import { Bot } from 'grammy'

const { BOT_TOKEN, BOT_WEBHOOK } = process.env

const bot = new Bot(BOT_TOKEN)

console.log('Setting Webhook')
let res = await bot.api.setWebhook(BOT_WEBHOOK)
console.log(res)
```

然后添加脚本到 `package.json`

```json
"scripts": {
    "webhook": "vercel env pull && node -r dotenv/config index.js"
},
```

使用 Vercel CLI 进行创建并部署项目

```shell
vercel
```

打开 [dashboard](https://vercel.com/dashboard)，设置项目的环境变量

1. `BOT_TOKEN`: `your bot token`
2. `BOT_WEBHOOK`: `https://your-project-name.vercel.app/api/bot`

最后设置并部署到生产环境中

```shell
pnpm run webhook
vercel --prod
```

## 结果

打开 Bot，发送 `/start`

![Start with bot](https://cdn.jsdelivr.net/gh/WingLim/winglim.github.io@hugo/static/images/cusdis-bot-start.jpeg)

因为 Serverless Function 无状态的原因，通过 Vercel 来部署 Telegram Bot 适合一些不需要依赖上下文状态的功能，举个简单的例子，比如说汇率查询，股价查询等。

但 Vercel 在逐步支持[集成第三方服务](https://vercel.com/integrations)，可以通过 [Upstash](https://vercel.com/integrations/upstash) 这个 Serverless Redis 来读写状态。
