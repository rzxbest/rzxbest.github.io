
---
title: zookeeper
tags: [zookeeper]
categories: [zookeeper]
top: 201
date: 2022-04-22 12:43:00
---
# zookeeper

# 选举
服务器节点状态 ： LOOKING 竞选状态,FOLLOWING 随从状态,LEADING 领导状态,OBSERVING 观察状态，不参与投票
服务器ID ：sid
事务ID ：zxid

规则：交换投票信息，对方事务id比自己大的，下一次投票投给对方节点，如果事务id相同，服务器id比自己大的，下一次投票给对方节点，多个取最大，投票给最大；当选票大于总节点一半时，选票多的机器变为leader,其余变成follow

假设5台机器
1.服务器1启动，投票给自己，（zxid=0,sid=1）,只有一票
2.服务器2启动，投票给自己，（zxid=0,sid=2）,并和服务器1交换信息，服务器1更改选票投给服务器2 ，服务器2只有两票 
3.服务器3启动，投票给自己，（zxid=0,sid=3）,并和服务器1、2交换信息，服务器1、2更改选票投给服务器3 ，服务器3只有三票 》2.5 ，成为leader ，服务器1，2成功follow
4. 服务器4启动，投票给自己 ，（zxid=0,sid=4）, 并和服务器1，2，3交换选票，但是此时服务器1，2，3已经不是looking状态，服务器3 3票，服务器 4 1票，服务器4成为follow
5. 服务器5启动，投票给自己 ，（zxid=0,sid=3）, 并和服务器1，2，3，4交换选票，但是此时服务器1，2，3，4已经不是looking状态，服务器3 4票，服务器5 1票，服务器5成为follow

leader挂了选举新leader？
服务器投票给自己，服务器1 （zxid=99,sid=1）服务器2 （zxid=102,sid=2）服务器4 （zxid=99,sid=4）服务器5 （zxid=99,sid=5），并交换选票信息，接下来改投票给服务器2，服务器2成为leader

