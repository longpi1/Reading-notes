#                               Go语言基于ip集合推出对应的CIDR

## 代码实现

Go语言中可以基于net包中的CIDR函数来将一组IP地址转换为CIDR，步骤主要分两步：

1. 通过ip地址获取对应的ip和CIDR掩码信息，默认为32位
2. 合并CIDR子网

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
type CLUSSubnet struct {
	Subnet net.IPNet `json:"subnet"`
}

// 合并子网
func MergeSubnet(subnets map[string]CLUSSubnet, snet CLUSSubnet) bool {
   for key, sn := range subnets {
      // In case of kubernetes with flannel, containers on each host have their own subnet,
      // such as 172.16.60.0/24, 172.16.14.0/24; and flannel IP has a bigger subnets,
      // 172.16.0.0/16. Here, we check if new subnet and existing subnets containers each other.
      if inc, v := SubnetContains(&sn.Subnet, &snet.Subnet); inc {
         if v == 1 {
            // existing subnet is bigger, ignore new one
            return false
         } else if v == -1 {
            // new subnet is bigger, remove the existing one
            delete(subnets, key)
         }
      }
   }

   subnets[snet.Subnet.String()] = snet
   return true
}

// The first value indicate if two subnets contains one or another, if true then,
// the second value returns 1 if n1 contains n2, -1 if n2 contains n1, 0 if equal.
func SubnetContains(n1, n2 *net.IPNet) (bool, int) {
	l1, r1 := ipnetToRange(n1)
	l2, r2 := ipnetToRange(n2)
	c12 := n1.Contains(l2) && n1.Contains(r2)
	c21 := n2.Contains(l1) && n2.Contains(r1)

	if !c12 && !c21 {
		return false, 0
	} else if c12 && c21 {
		return true, 0
	} else if c12 {
		return true, 1
	} else {
		return true, -1
	}
}
func ipToInt(ip net.IP) (*big.Int, int) {
	val := &big.Int{}
	b := []byte(ip)
	if IsIPv4(ip) && len(b) >= 4 {
		return val.SetBytes(b[len(b)-4:]), 32
	} else if IsIPv6(ip) && len(b) >= 16 {
		return val.SetBytes(b[len(b)-16:]), 128
	} else {
		return nil, 0
	}
}


func intToIP(ipInt *big.Int, bits int) net.IP {
	ipBytes := ipInt.Bytes()
	ret := make([]byte, bits/8)
	// Pack our IP bytes into the end of the return array,
	// since big.Int.Bytes() removes front zero padding.
	for i := 1; i <= len(ipBytes); i++ {
		ret[len(ret)-i] = ipBytes[len(ipBytes)-i]
	}
	return net.IP(ret)
}

func ipnetToRange(network *net.IPNet) (net.IP, net.IP) {
	// the first IP is easy
	firstIP := network.IP

	// the last IP is the network address OR NOT the mask address
	prefixLen, bits := network.Mask.Size()
	if prefixLen == bits {
		// Easy!
		// But make sure that our two slices are distinct, since they
		// would be in all other cases.
		lastIP := make([]byte, len(firstIP))
		copy(lastIP, firstIP)
		return firstIP, lastIP
	}

	firstIPInt, bits := ipToInt(firstIP)
	if firstIPInt == nil {
		return nil, nil
	}

	hostLen := uint(bits) - uint(prefixLen)
	lastIPInt := big.NewInt(1)
	lastIPInt.Lsh(lastIPInt, hostLen)
	lastIPInt.Sub(lastIPInt, big.NewInt(1))
	lastIPInt.Or(lastIPInt, firstIPInt)

	return firstIP, intToIP(lastIPInt, bits)
}

func ParseIPRange(value string) (net.IP, net.IP) {
	ipRange := strings.Split(value, "-")
	switch len(ipRange) {
	case 1:
		ip, ipnet, err := net.ParseCIDR(ipRange[0])
		if err == nil {
			return ipnetToRange(ipnet)
		} else {
			ip = net.ParseIP(ipRange[0])
			if ip == nil {
				return nil, nil
			}
			return ip, ip
		}
	case 2:
		ip := net.ParseIP(ipRange[0])
		if ip == nil {
			return nil, nil
		}
		ipR := net.ParseIP(ipRange[1])
		if ipR == nil {
			return nil, nil
		}
		return ip, ipR
	default:
		return nil, nil
	}
}

```