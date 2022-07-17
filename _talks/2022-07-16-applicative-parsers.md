---
layout: talk
title: "🇷🇺 Аппликативное программирование или как парсить SQL на чистом Haskell"
conference: "Vladimir Tech Talks #14"
date: 2022-05-27 12:00:00 +0003
slides_link: https://speakerdeck.com/player/e7c2108616b9472bb4d543bb7dd1cdd7
youtube_link: https://www.youtube.com/embed/9t8IXUgybag
---

Функциональное программирование использует подходы, не похожие на те, что мы применяем каждый день: функции чистые, переменных нет, а во многих языках нет даже циклов в привычном нам смысле. Из–за этого возникают специфические проблемы, которые приходится решать, и часто эти решения реализованы довольно интересно. Однако в докладе речь пойдет о другом: мы посмотрим на то, как применение сильных сторон ФП поможет нам построить очень удобную в применении штуку. Мы соберем из подручных средств очень простой, но мощный парсер.

Первая часть доклада — про основы синтаксиса Haskell. Во второй части — поговорим про функторы, из которых мы будем собирать нашу программу. В финале — мы сделаем парсер общего назначения в аппликативном стиле и научим его разбирать небольшое подмножество языка SQL.