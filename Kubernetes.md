Kubectl
```
Kubectl是k8s集群的命令工具，通过kubectl对集群本身进行管理，并能够在集群上进行容器化应用
的安装部署
```
Ingerss
```
通过跟 Ingress 交互得知某个域名对应哪个 service，再通过跟 kubernetes API 交互得知 service 地址等信息；综合以后生成配置文件实时写入负载均衡器，然后负载均衡器 reload 该规则便可实现服务发现，即动态映射
```

Service
```
将一组 Pods 公开为网络服务的抽象方法。
1.ClusterIP 集群内部使用
2.NodePort 对外访问应用
3.LoadBlanacer 对外访问使用

```

controller
```
在集群上管理和运行容器的对象

```
controller和Pod关系
```
Pod是通过controller实现应用的运维，比如弹性伸缩、滚动升级等
```

Pod
```
运行中的一组容器，Pod是kubernetes中应用的最小单位.

livenessProbe 存活检查
```
如果检查失败,将容器杀死
```
readinssProbe 就绪检查
```
如果检查失败,将容器剔除
```
1.有状态
2.无状态
认为Pod都是一样的
没有顺序要求
不用考虑那个先运行
随意进行伸缩和扩展


```
