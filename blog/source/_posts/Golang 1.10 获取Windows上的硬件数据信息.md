---
title: Golang获取windows硬件数据
categories: Golang
---

# Golang1.10获取windows硬件数据
   **嗯，工作原因需要使用go语言抓取windows下的硬件信息，包括CPU，GPU，内存，网卡信息，物理硬盘，系统信息等。做Golang语言开发的应该都知道，一手资料都在国外，国内的博客都是千篇一律互相Copy。所以，自己写个博客给大家分享一下，也记录一下。这里，我的golang版本是最新的1.10.1。不多说了，直接上干货**。
   
<!--more-->
获取CPU信息
-------

```
import 	（
    "github.com/StackExchange/wmi"
    "fmt"
)
type cpuInfo struct {
	Name          string
	NumberOfCores uint32
	ThreadCount   uint32
}
func getCPUInfo() {

	var cpuinfo []cpuInfo

	err := wmi.Query("Select * from Win32_Processor", &cpuinfo)
	if err != nil {
		return
	}
	fmt.Printf("Cpu info =",cpuinfo)
}

```
这里使用的是wmi库，关于Query方法中传入的string是通过WQL语句书写的。wmi的文档点击[这里](https://godoc.org/github.com/StackExchange/wmi#CreateQuery)，另外关于Query传入的struct里的参数也是多样的，具体可参考windows官方文档，点击[这里](https://msdn.microsoft.com/en-us/library/aa394373%28v=vs.85%29.aspx)。这个文档我们后续查询其他硬件资料要常用，我这里贴一次后续就不贴了，这里可以查询的参数多种多样，具体如下：

```
[Dynamic, Provider("CIMWin32"), UUID("{8502C4BB-5FBB-11D2-AAC1-006008C78BC7}"), AMENDMENT]
class Win32_Processor : CIM_Processor
{
  uint16   AddressWidth;
  uint16   Architecture;
  string   AssetTag;
  uint16   Availability;
  string   Caption;
  uint32   Characteristics;
  uint32   ConfigManagerErrorCode;
  boolean  ConfigManagerUserConfig;
  uint16   CpuStatus;
  string   CreationClassName;
  uint32   CurrentClockSpeed;
  uint16   CurrentVoltage;
  uint16   DataWidth;
  string   Description;
  string   DeviceID;
  boolean  ErrorCleared;
  string   ErrorDescription;
  uint32   ExtClock;
  uint16   Family;
  datetime InstallDate;
  uint32   L2CacheSize;
  uint32   L2CacheSpeed;
  uint32   L3CacheSize;
  uint32   L3CacheSpeed;
  uint32   LastErrorCode;
  uint16   Level;
  uint16   LoadPercentage;
  string   Manufacturer;
  uint32   MaxClockSpeed;
  string   Name;
  uint32   NumberOfCores;
  uint32   NumberOfEnabledCore;
  uint32   NumberOfLogicalProcessors;
  string   OtherFamilyDescription;
  string   PartNumber;
  string   PNPDeviceID;
  uint16   PowerManagementCapabilities[];
  boolean  PowerManagementSupported;
  string   ProcessorId;
  uint16   ProcessorType;
  uint16   Revision;
  string   Role;
  boolean  SecondLevelAddressTranslationExtensions;
  string   SerialNumber;
  string   SocketDesignation;
  string   Status;
  uint16   StatusInfo;
  string   Stepping;
  string   SystemCreationClassName;
  string   SystemName;
  uint32   ThreadCount;
  string   UniqueId;
  uint16   UpgradeMethod;
  string   Version;
  boolean  VirtualizationFirmwareEnabled;
  boolean  VMMonitorModeExtensions;
  uint32   VoltageCaps;
};
```
## 查询操作系统信息 ##

```
import (
	"runtime"
	"github.com/StackExchange/wmi"
	"fmt"
)

type operatingSystem struct {
	Name    string
	Version string
}

func getOSInfo() {
	var operatingsystem []operatingSystem
	err := wmi.Query("Select * from Win32_OperatingSystem", &operatingsystem)
	if err != nil {
		return
	}
	fmt.Printf("OS info =",operatingsystem)
}
```
同理，此处的operatingSystem struct中的变量和参数也可以增加，具体信息可查此[链接](https://msdn.microsoft.com/en-us/library/aa394239%28v=vs.85%29.aspx)
## 查询内存 ##

```
import (
	"syscall"
	"unsafe"
	"fmt"
)

var kernel = syscall.NewLazyDLL("Kernel32.dll")

type memoryStatusEx struct {
	cbSize                  uint32
	dwMemoryLoad            uint32
	ullTotalPhys            uint64 // in bytes
	ullAvailPhys            uint64
	ullTotalPageFile        uint64
	ullAvailPageFile        uint64
	ullTotalVirtual         uint64
	ullAvailVirtual         uint64
	ullAvailExtendedVirtual uint64
}

func getMemoryInfo() {

	GlobalMemoryStatusEx := kernel.NewProc("GlobalMemoryStatusEx")
	var memInfo memoryStatusEx
	memInfo.cbSize = uint32(unsafe.Sizeof(memInfo))
	mem, _, _ := GlobalMemoryStatusEx.Call(uintptr(unsafe.Pointer(&memInfo)))
	if mem == 0 {
		return
	}
	fmt.printf("total=:",memInfo.ullTotalPhys)
	fmt.printf("free=:",memInfo.ullAvailPhys)
}

```
这里我获取内存信息用的是syscall，使用wmi库应该也可以，具体大家可以看一下Windows的文档
## 网卡信息 ##

```
import (
	"hpc-backend/utils/logs"
	"net"
	"strings"
	"fmt"
)

type Network struct {
	Name       string
	IP         string
	MACAddress string
}

type intfInfo struct {
	Name       string
	MacAddress string
	Ipv4       []string
}

func getNetworkInfo() error {
	intf, err := net.Interfaces()
	if err != nil {
		logs.Error("get network info failed: %v", err)
		return err
	}
	var is = make([]intfInfo, len(intf))
	for i, v := range intf {
		ips, err := v.Addrs()
		if err != nil {
			logs.Error("get network addr failed: %v", err)
			return err
		}
		//此处过滤loopback（本地回环）和isatap（isatap隧道）
		if !strings.Contains(v.Name, "Loopback") && !strings.Contains(v.Name, "isatap") {
			var network Network
			is[i].Name = v.Name
			is[i].MacAddress = v.HardwareAddr.String()
			for _, ip := range ips {
				if strings.Contains(ip.String(), ".") {
					is[i].Ipv4 = append(is[i].Ipv4, ip.String())
				}
			}
			network.Name = is[i].Name
			network.MACAddress = is[i].MacAddress
			if len(is[i].Ipv4) > 0 {
				network.IP = is[i].Ipv4[0]
			}

			fmt.Printf("network:=",network)
		}

	}

	return nil
}
```
## 磁盘 ##

```
import (
	"github.com/StackExchange/wmi"
	"fmt"
)

type Storage struct {
	Name       string
	FileSystem string
	Total      uint64
	Free       uint64
}

type storageInfo struct {
	Name       string
	Size       uint64
	FreeSpace  uint64
	FileSystem string
}

func getStorageInfo() {
	var storageinfo []storageInfo
	var loaclStorages []Storage
	err := wmi.Query("Select * from Win32_LogicalDisk", &storageinfo)
	if err != nil {
		return
	}

	for _, storage := range storageinfo {
		info := Storage{
			Name:       storage.Name,
			FileSystem: storage.FileSystem,
			Total:      storage.Size,
			Free:       storage.FreeSpace,
		}
		localStorages = append(loaclStorages, info)
	}
	fmt.Printf("localStorages:=",localStorages)
}

```
此处使用的也是wmi库，关于磁盘空间的更多参数，请参考[这里](https://msdn.microsoft.com/en-us/library/aa394173%28v=vs.85%29.aspx)
## GPU ##

```
import (
	"github.com/StackExchange/wmi"
	"fmt"
)

type gpuInfo struct {
	Name string
}

func getGPUInfo() {

	var gpuinfo []gpuInfo
	err := wmi.Query("Select * from Win32_VideoController", &gpuinfo)
	if err != nil {
		return
	}
	fmt.Printf("GPU:=",gpuinfo[0].Name)
}
```
此处使用的也是wmi库，更多关于GPU的参数请参考Windows官方提供的[文档](https://msdn.microsoft.com/en-us/library/aa394512%28v=vs.85%29.aspx)

### 希望对大家有所帮助吧，包括关于主板的信息等等，大家都可以通过wmi这个库，参考Windows给出的官方文档，使用WQL语句进行查找 ##

---------

[1]: http://math.stackexchange.com/
[2]: https://github.com/jmcmanus/pagedown-extra "Pagedown Extra"
[3]: http://meta.math.stackexchange.com/questions/5020/mathjax-basic-tutorial-and-quick-reference
[4]: http://bramp.github.io/js-sequence-diagrams/
[5]: http://adrai.github.io/flowchart.js/
[6]: https://github.com/benweet/stackedit
