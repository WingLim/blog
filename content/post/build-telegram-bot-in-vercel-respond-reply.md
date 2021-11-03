---
title: "在 Vercel 上部署 Telegram Bot（二）：通过与回复"
date: 2021-10-27T15:07:15+08:00
slug: build-telegram-bot-in-vercel-respond-reply
description: Hey Bot, 帮我同意和回复一下评论
tags:
- Telegram
- Bot
- Vercel
categories:
- Tech
---

在 [上一篇文章](https://blog.limx.dev/post/build-telegram-bot-in-vercel-basic/) 中我们讲了如何在 Vercel 上部署一个简单的 Telegram Bot，这里我们讲讲如何给 Bot 的消息添加可点击的按钮，以及通过回复 Bot 的消息来回复评论

## Cusdis Webhook

首先我们先创建一个 `api/hook/[chatId].ts` 来处理 Cusdis 发送的 Webhook 请求

在 Vercel 中 `[chatId].ts` 可以对应路径 `/api/hook/123456`

这里隐去了一些类型定义，完整代码可以查看 [api/hook/[chatId].ts](https://github.com/WingLim/cusdis-telegram-bot/blob/main/api/hook/%5BchatId%5D.ts)
```ts
import { VercelRequest, VercelResponse } from '@vercel/node'
import { InlineKeyboard } from 'grammy'
import { bot } from '../../index'

// buildNewCommentMsg 构建发送给用户的通知消息
function buildNewCommentMsg(data: NewCommentHookData) {
    return `New comment on website <strong>${
        data.project_title
    }</strong> in page <strong>${data.page_title}</strong>:
<pre>
${data.content.replace(/<[^>]*>?/gm, "")}
</pre>
by: <strong>${data.by_nickname}</strong>`
}

export default async (req: VercelRequest, res: VercelResponse) => {
    if (req.method === 'POST') {
        const chatId = req.query['chatId'] as string
        const { type, data } = req.body as HookBody<NewCommentHookData>

        switch (type) {
            case 'new_comment': {
                const msg = buildNewCommentMsg(data)
                
                // 创建 InlineKeyboard，即附带在消息中的按钮，并且添加 callback_data
                const approveKeyboard = new InlineKeyboard().text('Approve', 'approve')

                let new_msg = await bot.api.sendMessage(chatId, msg, {
                    parse_mode: 'HTML',
                    reply_markup: approveKeyboard
                })
                break
            }
        }

        res.json({
            msg: "works"
        })
    }
}
```

然后我们需要在 Bot 中添加 `/gethook` 命令来获取给 Cusdis 使用的 Webhook 地址

```ts
// api/bot.ts
const { VERCEL_URL } = process.env

bot.command('gethook', async (ctx) => {
    let chanId = ctx.message.chat.id
    let hookUrl = `https://${VERCEL_URL}/api/hook/${chanId}`
    await ctx.reply(`Your Webhook URL:\n ${hookUrl}`)
})
```

## 使用 Redis

因为 [InlineKeyboardButton](https://core.telegram.org/bots/api#inlinekeyboardbutton) 的 `callback_data` 最高支持 64 字节的数据，而 Cusdis 的 `approve_link` 包含了 SHA256 的 token，所以我们没有办法直接将 `approve_link` 包在 Button 里面一起发给用户，所以这里用到 [Upstash](https://upstash.com/)（一个 Serverless 的 Redis 服务）来保存

在 Vercel 中可以通过 Intergrations 来一键集成到项目中：[Upstash in Vercel](https://vercel.com/integrations/upstash)

添加库

```shell
pnpm i redis@next
```

添加 `redis.ts`

```ts
import { createClient } from 'redis'

const { REDIS_URL } = process.env

export const client = createClient({
    url: REDIS_URL
})
```

然后我们再将 `approve_link` 存入 Redis 中，方便后续 Bot 使用

```ts
// api/hook/[chatId].ts
import { client } from '../../redis'

...

let new_msg = await bot.api.sendMessage(chatId, msg, {
    parse_mode: 'HTML',
    reply_markup: approveKeyboard
})

// 设置 key 为聊天 id + 消息 id 这样能保证对于单独的用户来说 key 是唯一的
let key = chatId + new_msg.message_id.toString()
await client.connect()
// 设置过期时间为 3 天，和 approve_link 的 token 的过期时间一致
await client.set(key, data.approve_link, {
    EX: 259200
})
break

...
```

## 通过评论

在一个包含 `callback_data` 的按钮被点击后，Bot 会发送消息到 Webhok 上，我们可以使用 `bot.callbackQuery()` 来监听并处理这个消息

```ts
// api/bot.ts

// 这里的 approve 即为 button 的 callback_data
bot.callbackQuery('approve', async (ctx) => {
    let chatId = ctx.message.chat.id
    let key = chatId.toString() + ctx.message.message_id.toString()
    await client.connect()
    let link = await client.get(key)
    if (link) {
        // Cusdis 的 API 地址为 domain.com/api/open/approve?token=xxx
        // 而 approve_link 为 domain.com/open/approve，所以这里需要进行替换
        let api = link.replace('open', 'api/open')
        let res = await axios.post(api)
        if (res.status == 200) {
            // 成功则返回 'Successed' 消息
            await ctx.answerCallbackQuery({
                text: 'Successed'
            })
        } else {
            // 失败则返回 'Failed' 消息
            await ctx.answerCallbackQuery({
                text: 'Failed'
            })
        }
    } else {
        await ctx.reply('Token has expired')
    }
})
```

## 通过回复添加评论

监听消息，当用户发送的消息是回复消息时，通过回复消息去查找 `approve_link`，并发送带数据的 POST 请求即可添加评论到指定回复上

```ts
// api/bot.ts

bot.on('msg', async (ctx) => {
    // 判断该消息是否回复了某一条消息
    if (ctx.message.reply_to_message) {
        let reply_msg = ctx.message.reply_to_message
        let key = ctx.chat.id.toString() + reply_msg.message_id.toString()
        let replyContent = ctx.message.text

        await client.connect()
        // 查找 approve_link
        let link = await client.get(key)
        if (link) {
            let api = link.replace("open", "api/open")
            // 这里发送的数据为
            // {
            //      replyContent: 'your reply'
            // }
            let res = await axios.post(api, {
                replyContent
            })
            if (res.status == 200) {
                ctx.reply("Success to append comment")
            } else {
                ctx.reply("Fail to append comment")
            }
        } else {
            ctx.reply("Token has expired")
        }
    }
})
```

## 效果

使用 `vercel --prod` 部署到生产环境后，让我们来看看实际效果展示

![cusdis-bot](https://cdn.jsdelivr.net/gh/WingLim/winglim.github.io@hugo/static/images/cusdis-bot-final-result.jpg)

你可以给自己的网站添加 [Cusdis](https://cusdis.com/) 评论服务，并启用 [@CusdisxBot](https://t.me/CusdisxBot) 来尝试这个机器人的功能
