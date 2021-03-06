CoreBluetooth开发文档
========================

[TOC]

## 介绍
本文档介绍了如何使用CoreBluetooth开发蓝牙中心模式的APP，实现蓝牙BLE搜索、连接、断开、接收和发送数据等操作。外设模式在本文档不讨论。

中心模式： 即手机作为中心，蓝牙模块作为外设。手机主动发送连接、断开和数据读写请求。   
外设模式： 即手机配置为一个蓝牙外设，可以被动接受和处理其他蓝牙设备的请求信息。   

语言:Swift 3.0   
开发平台: Xcode 7.1  
调试平台: iphone6s ios 10.1  
Github地址: https://github.com/DennisMao/eswiftBLE

Github上已上传了个较完整的Demo，带有中文注释，可参考

## 框架
### 主要函数
#### CBPeripheral 外设类  （扫描服务、属性，读写数据，设置Notify，读取信号）
#####属性:  
+ identifier: 设备的UUID信息  
+ name: 设备名  
+ delegate: 委托函数  

#####功能操作:  
搜索  

+ discoverServices 搜索服务，调用该函数会扫描该服务的全部信息，比如属性和描述  
+ discoverIncludeServices 获取指定服务，该函数可以限定搜索含有特定UUID的服务  
+ services 可以理解为数组，里面包含了多个服务信息   
+ discoverCharactistic 搜索属性，调用该函数会搜索某个服务下特定属性的全部信息，比如描述和各个值。  
+ discoverDescripter 搜索描述符，调用该函数会搜索某个属性下的特定的描述符  

读写操作  

+ readValue 读取一个属性或者描述符的数值，若读取成功会在委托里响应,对应函数
```
func peripheral(CBPeripheral, did​Update​Value​For:​ CBCharacteristic, error:​ Error?)
```
   
+ writeValue 写入一个数值到属性或者描述符，若写入成功会在委托里响应，
```
func peripheral(_ peripheral: CBPeripheral, 
        didWriteValueFor characteristic: CBCharacteristic, 
                   error: Error?)
```                   
+ readRSSI 读取当前信号值,若读取成功会返回如下响应 
``` 
func peripheral​Did​Update​RSSI(CBPeripheral, error:​ Error?)
   ```  
   
设置Notify

+ setNotifyValue 设置Notify开关，调用后会在委托函数中响应，可查看是否开启/关闭成功  
```
func peripheral​Did​Update​RSSI(CBPeripheral, error:​ Error?)
``` 

#### CBCentralManger 中心管理器（设备扫描，连接，硬件状态）
#####功能操作:  
**搜索**

+ scanForPheripherals 搜索外设，可设置筛选条件，比如服务UUID
+ stopScan 停止搜索    

**连接**  

+ connect 外设连接，传入的参数为CBPeripheral，可以从调用扫描外设函数scanForPherpheral后的didDiscover响应函数中获得  
+ cancelPheralConnection 取消当前连接，在建立连接过程中可以调用该函数取消连接  



### 主要委托
CBPeripheral​Delegate //外设类的委托，用于配置外设操作的响应  
CBCentral​Manager​Delegate //中心管理器委托，用于对外设管理操作的响应  


## 开发过程

假定我们有一个外设,读取电量的最大值，改写电量的最小值  
设备 name = Mobike UUID = FF00  
服务 
Battery  UUID = FFEE  
   属性  
	Max UUID = FFE1  
	Min UUID = FFE2  
	Avg UUID = FFE3  

### 初始化

1. 引入中心库，委托   
```
import CoreBluetooth
``` 
2. 导入委托Delegate 
```
class ServiceViewController: UIViewController,CBCentralManagerDelegate, CBPeripheralDelegate  
```

3. 实例化变量  
```
    var PeripheralToConncet : CBPeripheral!   //实例化外设
```
```
    var trCBCentralManager : CBCentralManager!  //实例化外设管理区
```

4. 初始化变量，绑定delegate  

```
self.PeripheralToConncet.delegate = self
```
```
self.trCentralManager = CBCentralManager(delegate: self , queue: nil)
```

### 获得当前设备状态

```
//检查外设管理器状态
func centralManagerDidUpdateState(_ central: CBCentralManager) {
    switch central.state {
        
    case CBManagerState.poweredOn:  //蓝牙已打开正常
        NSLog("启动成功，开始搜索")
        self.myCentralManager.scanForPeripherals(withServices: nil, options: nil) //不限制
    case CBManagerState.unauthorized: //无BLE权限
        NSLog("无BLE权限")
    case CBManagerState.poweredOff: //蓝牙未打开
        NSLog("蓝牙未开启")
    default:
        NSLog("状态无变化")
    }
}

```

### 搜索外设
开启搜索前应先确定系统蓝牙已经打开，状态可以由上个函数获得  
【动作】开启搜索

```
self.myCentralManager.scanForPeripherals(withServices: nil, options: nil) //不设置过滤条件

```

【响应】搜索到设备,其UUID为“FF00”,将外设参数赋值到已经定义的 PeripheralToConnect 保存起来

```
func centralManager(_ central: CBCentralManager, didDiscover peripheral: CBPeripheral, advertisementData: [String : Any], rssi RSSI: NSNumber) {
	if(peripheral.identifier.uuidString.contains("FFEE")){
         PeripheralToConnect = peripheral 
          NSLog("搜索到设备，Name=\(peripheral.name!) UUID=\(peripheral.identifier)")
    }
}

```

### 建立链接和断开链接

【动作】建立连接  

```
trCentralManager.connect(PeripheralToConnect, options: nil)
```

【响应】已建立连接  

```
func centralManager(_ central: CBCentralManager, didConnect peripheral: CBPeripheral) {
    NSLog("已连接\(peripheral.name!)")
    self.myPeripheralToMainView! = peripheral
}
```

【响应】建立连接失败

```
func centralManager(_ central: CBCentralManager, didFailToConnect peripheral: CBPeripheral, error: Error?){
    NSLog("连接失败，设备名\(peripheral.name!),原因\(error)")
}
```

### 扫描服务和特征

声明定义变量，以储存所需服务和特征

```
trCharactisticMax: CBCharactistic!  
trCharactisticMin: CBCharactistic! 
trService: CBService!  
```
【动作】扫描服务

```
PeripheralToConnect.discoverServices(nil)//搜索服务,不做过滤
```
【响应】成功扫描到服务

```
func peripheral(_ peripheral: CBPeripheral, didDiscoverServices error: Error?){
   NSLog("搜索到\(peripheral.services?.count)个服务")
    if peripheral.services?.count != 0{
        for service in peripheral.services! {
            //trTextInfo.insertText("服务:\(service.description)")
            if service.uuid.uuidString.contains("FFEE") { //遍历所有搜索到的服务，查找Battery服务
                NSLog("获取到指定服务 \(service.uuid.uuidString)")
                trService = service as CBService  //获取到指定读写的服务Battery，同理可以用名字做筛选，但比较推荐是用UUID  
            }
        }
    }else{
        NSLog("无有效服务") //未能搜索到服务
    }
}
```
【动作】扫描特征

```
PeripheralToConnect.discoverCharacteristics(nil, for: trService) //将已搜到的服务参数传入
```
【响应】成功扫描到特征

```
//搜索到Charactistic   查找指定读写的属性
func peripheral(_ peripheral: CBPeripheral, didDiscoverCharacteristicsFor service: CBService, error: Error?){
    if service.characteristics!.count != 0 {
      for charactistic in service.characteristics!{
        //此处有未知bug for in 循环无法给变量正确定义属性，所需要我们自己找个中介变量定义一下
         let aC :CBCharacteristic = charactistic as CBCharacteristic
         if trService == service {
            if aC.uuid.uuidString.contains("FFE1") {
                trCharactisticMax = aC as CBCharacteristic   //获取到Max属性
            }else if aC.uuid.uuidString.contains("FFE2"){
            	trCharactisticMin = aC as CBCharacteristic   //获取到Max属性
            }
          }
       }     
	}else{
		NSLog("未能搜索到属性")
	}
}
```

### 读取数据
搜索到属性后，可以对属性进行读写等操作。操作前建议先检查该属性的权限，是否提供读写功能。
此处自己写了个属性显示函数，具体在下一章节有介绍。
【动作】查看功能

```
propertiesString(properties: trCharactisticMin)! //查看Min属性所含有的功能
propertiesString(properties: trCharactisticMax)!

```
#### Read方式

```
PeripheralToConncet.readValue(for: trCharactisticMax)  //读取Max值
```

#### Notify方式
 
```
PeripheralToConncet.setNotifyValue(true, for: trCharactisticMax) //打开Notify
PeripheralToConncet.setNotifyValue(false, for: trCharactisticMax) //关闭Notify

``` 

不管是Read还是Notify，接收到数据都会相应以下函数  
【响应】接收响应

```
func peripheral(_ peripheral: CBPeripheral, didUpdateValueFor characteristic: CBCharacteristic, error: Error?){
    if error == nil {
        NSLog("成功读取到数据:\(characteristic.value!)")
    }else{
        NSLog("读取错误 \(error.debugDescription)")
    }
}
```

### 写入数据
	
>蓝牙4.0 BLE数据限制每次发送Payload即有效数据长度只有20bytes，后续版本该数据包大小会有增加 
	
	
#### Write方式

Write方式的特点是写入数据后蓝牙外设会返回写入结果以显示上一次写入操作是否成功，
通过判断响应可知道写入的情况，适用于数据量小或者数据类型较分类的情况。    

【动作】写入

```
PeripheralToConncet.writeValue(dataToTrans, for:   trCharactisticMin, type: CBCharacteristicWriteType.withResponse)
```

【响应】已写入数据

```
func peripheral(_ peripheral: CBPeripheral, didWriteValueFor characteristic: CBCharacteristic, error: Error?){
    if error == nil {
        NSLog("写入成功，数据：\(characteristic.value!)成功")
    }else{
        NSLog("写入错误 \(error.debugDescription)")
    }
}
```
#### Write Without Response方式

Write Without Response方式的特点是写入数据后蓝牙外设不返回写入结果以显示上一次写入操作是否成功。适用于数据连续，量大且自带头尾校验情况。    

【动作】写入

```
PeripheralToConncet.writeValue(dataToTrans, for:   trWriteCharactisic, type: CBCharacteristicWriteType.withoutResponse)
```


## 自定义辅助函数

### 权限显示函数
由于当前版本CoreBluetooth里面的特性Properties不带字符串输出，在显示上会不太方便。故此写了个函数来输出当前特性所含的全部Properties，以string类型输出，可直接显示

【使用方法】

```
NSLog("当前Charactistic所含Properties有：" + \(propertiesString(properties: trCharactisticMax.properties)!))
```

【函数】

``` 
    func propertiesString(properties: CBCharacteristicProperties)->(String)!{
        var propertiesReturn : String = ""
        // Just to see what we are dealing with
        if (properties.rawValue & CBCharacteristicProperties.broadcast.rawValue) != 0 {
            propertiesReturn += "broadcast|"
        }
        if (properties.rawValue & CBCharacteristicProperties.read.rawValue) != 0 {
            propertiesReturn += "read|"
        }
        if (properties.rawValue & CBCharacteristicProperties.writeWithoutResponse.rawValue) != 0 {
            propertiesReturn += "write without response|"
        }
        if (properties.rawValue & CBCharacteristicProperties.write.rawValue) != 0 {
            propertiesReturn += "write|"
        }
        if (properties.rawValue & CBCharacteristicProperties.notify.rawValue) != 0 {
            propertiesReturn += "notify|"
        }
        if (properties.rawValue & CBCharacteristicProperties.indicate.rawValue) != 0 {
            propertiesReturn += "indicate|"
        }
        if (properties.rawValue & CBCharacteristicProperties.authenticatedSignedWrites.rawValue) != 0 {
            propertiesReturn += "authenticated signed writes|"
        }
        if (properties.rawValue & CBCharacteristicProperties.extendedProperties.rawValue) != 0 {
            propertiesReturn += "indicate|"
        }
        if (properties.rawValue & CBCharacteristicProperties.notifyEncryptionRequired.rawValue) != 0 {
            propertiesReturn += "notify encryption required|"
        }
        if (properties.rawValue & CBCharacteristicProperties.indicateEncryptionRequired.rawValue) != 0 {
            propertiesReturn += "indicate encryption required|"
        }
        return propertiesReturn
    }
```

### TextView滚动显示函数
在使用TextView时候系统未带插入数据后自动滚动到最新一行，比较不变，故此写了个函数，可解决以下问题：  
1.插入数据后，自动滚动到最新一行   
2.插入数据自动换行     
3.可设置TextView数据缓存的最大长度，超出可自动清空

```
func addScollTextView(text: String){
  if trTextDataRead.text.characters.count > 4000 {    //设置MAX_Length
        trTextDataRead.text.removeAll()
    }
    trTextDataRead.text.append("\n" + text)
trTextDataRead.scrollRangeToVisible(NSMakeRange(trTextDataRead.text.characters.count, 0))
}
```


## 常见问题
整理了自己遇见的和StackOverflow里常见的问题。  

1. 调用CBCentralManager.connect函数后，didConnect无响应
>检查CBCentralManager是否已经绑定了delegate 即  
>var myCentralManager: CBCentralManager!  
>myCentralManager.delegate = self

2. 调用Write方式写入时候，一段时间BLE断开
>检查响应函数，判断写入是否成功，若系统在30秒内未能接收到正确的响应，系统会主动申请断开链接

## 引用文档
官方蓝牙开发指南: https://developer.apple.com/library/content/documentation/NetworkingInternetWeb/Conceptual/CoreBluetooth_concepts/AboutCoreBluetooth/Introduction.html#//apple_ref/doc/uid/TP40013257     

官方API: https://developer.apple.com/reference/corebluetooth 

