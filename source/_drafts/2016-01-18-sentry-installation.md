title: Sentry 簡單安裝指南
date: 2016-01-18 08:34:10
tags:
---

Sentry 是我們拿來記錄 web service error logging 的好工具，主要的優點是界面漂亮，支援多種主要語言的 client ，跟 email & slack 的整合都很完整，目前是由 [David Cramer](https://github.com/dcramer) 和創造Flask 的 [Armin Ronacher](https://github.com/mitsuhiko)  為主要 maintainer ，之前有人問過我跟 ELK stack 的比較，我會說 Sentry 對 OP 來說會比較省事，比ELK 好安裝也好管理，而且也有 SaaS 的服務可以直接用，價格也算蠻便宜的，不過這篇文章主要是想挑戰自己安裝，在業務量不大的情況下，採用 AWS t2.micro 也可以撐住不少 exception log 。

## 安裝 ##

基本上這篇文章是參考 [https://docs.getsentry.com/on-premise/server/installation/](https://docs.getsentry.com/on-premise/server/installation/)

新版的 sentry 8.0.0 已經不支援 MYSQL，而是使用 PostgresSQL or SQLite。

安裝 virtaulenv，並且建立資料夾
```
pip install -U virtualenv
virtualenv /www/sentry/
source /www/sentry/bin/activate
```
