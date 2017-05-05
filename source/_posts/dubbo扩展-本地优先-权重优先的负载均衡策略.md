---
title: 'dubbo扩展:本地优先,权重优先的负载均衡策略'
date: 2017-05-05 12:55:46
categories: java
tags: 
  - dubbo
---
背景
=============
开发环境经常因为一个服务有多个人启动, 默认的负载均衡策略是随机的, 导致有人调试时老调用不到对应的机器上.
为了稳定调用到自己的机器, 只能去把别人的服务禁用掉只留自己的, 后面又会忘记还原回来, 导致自己的服务关闭之后, 这个服务就没有了可用的提供者, 服务出于不可用状态   
使用本文提供的这种负载均衡方式, 就不用再使用禁用这个功能, 优先调用本机服务, 如果本机没有开启, 固定找到权重最高的服务

先上代码
-------------
```java
import com.alibaba.dubbo.common.URL;
import com.alibaba.dubbo.common.utils.NetUtils;
import com.alibaba.dubbo.rpc.Invocation;
import com.alibaba.dubbo.rpc.Invoker;
import com.alibaba.dubbo.rpc.cluster.loadbalance.AbstractLoadBalance;

import java.util.List;

/**
 * 本地优先的负载均衡方式.
 * Created by Lifeng on 2017/5/4.
 */
public class LocalFirstLoadBalance extends AbstractLoadBalance {

    private static String localIp = NetUtils.getLocalHost();

    /**
     * Invoker选择: 优先选择本机Invoker, 无本机Invoker则选择权重设置最高的Invoker.
     */
    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> list, URL url, Invocation invocation) {

        if (localIp != null) {
            for (Invoker<T> invoker : list) {
                if (localIp.equals(invoker.getUrl().getHost())) {
                    return invoker;
                }
            }
        }

        int maxWeight = Integer.MIN_VALUE;
        Invoker<T> maxWeightInvoker = null;
        for (Invoker<T> invoker : list) {
            int weight = invoker.getUrl().getMethodParameter(invocation.getMethodName(), "weight", 100);
            if (weight >= maxWeight) {
                maxWeight = weight;
                maxWeightInvoker = invoker;
            }
        }
        return maxWeightInvoker;

    }

}
```
扩展声明
-------------
光代码是不行的, 还要配置
首先要添加一个文件来声明这个扩展
位置: META-INF/services/com.alibaba.dubbo.rpc.cluster.LoadBalance
内容: localfirst=com.xxx.LocalFirstLoadBalance

使用自定义负载均衡策略
-------------
dubbo默认的负载均衡策略是random: com.alibaba.dubbo.rpc.cluster.loadbalance.RandomLoadBalance  
要改用自定义负载均衡策略: 请参考[http://dubbo.io/User+Guide-zh.htm](http://dubbo.io/User+Guide-zh.htm "http://dubbo.io/User+Guide-zh.htm")  关键字: "loadbalance="  
配置: loadbalance=localfirst

说明
-------------
这种负载均衡策略只适用于开发环境, 千万别弄到生产上去了