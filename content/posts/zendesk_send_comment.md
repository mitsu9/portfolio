---
title:      "Zendesk APIを使ってコメント（返信）する方法"
date:       2020-01-07T20:00:00+09:00
categories: ["tech"]
tags:       ["zendesk"]
---

ZendeskはAPIが公開されており、様々な操作をAPI経由でおこなうことができます。

今回はRubyでZendesk APIを使ってチケットに返信する方法を紹介します。

## Net::HTTPを利用する方法
Zendesk APIを使って返信したいときは、送信したい内容を所定の構造のjsonでPOSTします。

publicをtrueにすると返信、falseにするとコメントになります。

```rb
require "json"
require "net/http"

## settings
subdomain = "subdomain"
mail = "mail_address"
access_token = "access_token"

## params
ticket_id = "ticket_id"
author_id = "author_id"
is_public = true
reply_text = <<EOS
返信する文章をここに記入する。
ファイルから取得しても良いかもしれない。
EOS

## reply to a ticket
ticket = {"ticket": {"comment": { "body": reply_text, "public": is_public, "author_id": author_id }}}.to_json
url = "https://#{subdomain}.zendesk.com/api/v2/tickets/#{ticket_id}.json"
uri = URI url
req = Net::HTTP::Put.new "#{uri.path}?#{uri.query}"
req.basic_auth "#{mail}/token", access_token
req["Content-Type"] = "application/json"
req.body = ticket
response = Net::HTTP.start(uri.host, uri.port, use_ssl: true) { |http| http.request req }
```

## ZendeskAPIを利用する方法
Zendeskは様々な言語でライブラリを提供しています。
次にライブラリを使うケースを紹介します。

```sh
$ gem install zendesk_api
```

```rb
require "zendesk_api"

## settings
subdomain = "subdomain"
mail = "mail_address"
access_token = "access_token"

## params
ticket_id = "ticket_id"
author_id = "author_id"
is_public = true
reply_text = <<EOS
返信する文章をここに記入する。
ファイルから取得しても良いかもしれない。
EOS

## reply to a ticket
client = ZendeskAPI::Client.new do |config|
  config.url = "https://#{subdomain}.zendesk.com/api/v2"
  config.username = mail
  config.token = access_token
end
ticket = client.tickets.find!(id: ticket_id)
ticket.comment = ZendeskAPI::Ticket::Comment.new(client, value: reply_text, public: is_public, author_id: author_id)
ticket.save!
```

## 参考リンク
- [公式ドキュメント](https://developer.zendesk.com/rest_api/docs/support/ticket_comments#create-ticket-comment)
- [zendesk/zendesk_api_client_rb](https://github.com/zendesk/zendesk_api_client_rb)
