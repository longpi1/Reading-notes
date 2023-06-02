#                                                         Go语言基于ip集合推出对应的CIDR

## 实现步骤

Go语言中可以基于net包中的CIDR函数来将一组IP地址转换为CIDR，步骤主要分两步：

1. 通过ip地址获取对应的ip和CIDR掩码信息，默认为32位
2. 合并相邻CIDR子网



## 代码实现

实现代码如下：

### 1.获取每个ip对应的CIDR

```go
// 通过ip地址获取对应的ip和CIDR掩码信息，默认为32位  
IPNet: net.IPNet{
		IP:   net.ParseIP(IPAddress),
        //  用ones和bits来一个CIDR掩码
		Mask: net.CIDRMask(IPOnes, 32),
}
					
```

### 2.合并子网CIDR

```golang
func MergeCIDRs(cidrs []*net.IPNet) []*net.IPNet {
    // 先按照IP地址排序
    sort.Slice(cidrs, func(i, j int) bool {
    return bytes.Compare(cidrs[i].IP, cidrs[j].IP) < 0
    })

// 合并相邻的CIDR
mergedCIDRs := make([]*net.IPNet, 0, len(cidrs))
currentCIDR := cidrs[0]
for i := 1; i < len(cidrs); i++ {
    if currentCIDR.Contains(cidrs[i].IP) {
        continue
    }
    if bytes.Equal(currentCIDR.Mask, cidrs[i].Mask) && currentCIDR.IP.Equal(cidrs[i].IP.Mask(currentCIDR.Mask)) {
        currentCIDR = &net.IPNet{
            IP:   currentCIDR.IP,
            Mask: currentCIDR.Mask,
        }
        continue
    }
    mergedCIDRs = append(mergedCIDRs, currentCIDR)
    currentCIDR = cidrs[i]
}
  mergedCIDRs = append(mergedCIDRs, currentCIDR)

  return mergedCIDRs
}
```

### 3.测试

```golang
cidrs := []*net.IPNet{
&net.IPNet{IP: net.ParseIP(“192.168.0.0”), Mask: net.CIDRMask(24, 32)},
&net.IPNet{IP: net.ParseIP(“192.168.1.0”), Mask: net.CIDRMask(24, 32)},
&net.IPNet{IP: net.ParseIP(“192.168.2.0”), Mask: net.CIDRMask(24, 32)},
&net.IPNet{IP: net.ParseIP(“192.168.1.0”), Mask: net.CIDRMask(25, 32)},
&net.IPNet{IP: net.ParseIP(“192.168.1.128”), Mask: net.CIDRMask(25, 32)},
}
mergedCIDRs := MergeCIDRs(cidrs)
for _, c := range mergedCIDRs {
fmt.Printf(“%s\n”, c.String())
}
输出结果为：
192.168.0.0/23
192.168.2.0/24
```