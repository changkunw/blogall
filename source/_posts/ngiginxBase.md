---
title: nginx基础知识
date: 2017-04-04 20:20:05
tags: nginx
---

## 简介

nignx是高性能的http和反向代理服务器软件，还是imap/pop3/smtp代理服务器，nginx是apache服务器不错的替代品，它能够支持高达5000个并发连接数的响应，而内存、cpu等系统资源消耗低，运行非常稳定。

<!--more-->

## why ningx

1. 高并发
2. 内存消耗少
3. 成本低廉
4. 配置简单，容易上手

## 负载均衡

由多台服务器以对称的方式组成一个服务器集合，平均奉陪客户请求到服务器队列，用最少的投资获得接近于大型主机的性能

## 反向代理

以代理服务器来接受internet上的连接请求，然后将请求转发给内部网络上的服务器，并将conference服务器上得到的结果返回给internet上请求连接的客户端，此时代理服务器对外表现的就是一个服务器

## 正向代理

正向代理,也就是传说中的代理,他的工作原理就像一个跳板

## 正向代理和反向代理的区别

1. 用途：正向代理的典型用途是为在防火墙内的局域网客户端提供访问Internet的途径。正向代理还可以使用缓冲特性减少网络使用率。反向代理的典型用途是将 防火墙后面的服务器提供给Internet用户访问（from internet）
2. 安全：正向代理允许客户端通过它访问任意网站并且隐藏客户端自身，因此你必须采取安全措施以确保仅为经过授权的客户端提供服务，反向代理对外都是透明的，访问者并不知道自己访问的是一个代理。

## lvs 和 nginx 负载均衡的区别

lvs 是在4层（传输层）做负载均衡，而nginx是在7层（应用层）做负载均衡，软件7层做负载均衡大多基于http反向代理