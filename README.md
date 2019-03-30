# AndroidWifiPrinter
Connect the HP printer via wifi and printer the pdf file.连接局域网打印机，并打印pdf文件。


安卓自带的打印服务由于各种阉割版问题，不考虑。

研究了一下 [PrinterShare](https://play.google.com/store/apps/details?id=com.dynamixsoftware.printershare) 暂时先使用wifi方式打印文件。
## 两种方式

### 第一种

通过mac或者pc分享打印机，安卓连接同一个局域网的wifi。通过[mdns](https://developer.android.com/reference/android/net/nsd/NsdManager) service搜索打印机，基于[IPP](https://en.wikipedia.org/wiki/Internet_Printing_Protocol) + [cups](https://en.wikipedia.org/wiki/CUPS)打印文件。

#### 寻找打印机

1.初始化

```java
NsdManager nsdManager = (NsdManager) getApplicationContext().getSystemService(Context.NSD_SERVICE);
```

2.注册DiscoveryListener，注册ResolveListener

```java
mResolverListener = new NsdManager.ResolveListener() {
......
            @Override
            public void onServiceResolved(NsdServiceInfo serviceInfo) {
                Log.d("printer", "onServiceResolved:IP " + serviceInfo.getHost());
                Log.d("printer", "onServiceResolved:Port " + serviceInfo.getPort());

            }
        };

mDiscoveryListener = new NsdManager.DiscoveryListener() {     
......

            @Override
            public void onServiceFound(NsdServiceInfo serviceInfo) {

                Log.d("printer", "onServiceFound: " + serviceInfo.getServiceName());
  				nsdManager.resolveService(serviceInfo,mResolverListener);
            }

......
        };
    }
```

3.开始寻找服务

```java
nsdManager.discoverServices(serviceType, NsdManager.PROTOCOL_DNS_SD, mDiscoveryListener);
```

#### 打印文件

使用[cups4j](https://github.com/harwey/cups4j)打印文件，[这里](https://github.com/freevnshns/decser-android) 有jar包。

pdf使用[iText5](https://itextpdf.com/en/products/itext-5-legacy)生成

```java
CupsClient cupsClient = new CupsClient(new URL("http://192.168.223.1:631"));
            List<CupsPrinter> cupsPrinterList = cupsClient.getPrinters();
CupsPrinter printer = cupsClient.getDefaultPrinter();
InputStream inStream = this.getResources().openRawResource(R.raw.pdf);

ByteArrayOutputStream baos = new ByteArrayOutputStream();
byte[] buff = new byte[inStream.available()];
int i = Integer.MAX_VALUE;
while ((i = inStream.read(buff, 0, buff.length)) > 0) {
     baos.write(buff, 0, i);
}


PrintJob.Builder builder = new PrintJob.Builder(baos.toByteArray()).userName("user").copies(1);

printer.print(builder.build());
```

### 第二种

打印机分享wifi，手机直连打印机。通过[mdns](https://developer.android.com/reference/android/net/nsd/NsdManager) service搜索打印机，socket连接打印机，发送打印指令。

#### 连接打印机

```java
Socket socket = new Socket("192.168.223.1", 9100);
```

但是这里无论我发生什么数据，打印机都没有反应。应该是数据格式的问题。

之后通过抓包[PrinterShare](https://play.google.com/store/apps/details?id=com.dynamixsoftware.printershare) 的数据，发现它发送了 [PCL](https://en.wikipedia.org/wiki/Printer_Command_Language) 格式数据。

可通过 [**genPCLm.cpp**](https://github.com/ibevilinc/WFDSPrintPlugin/blob/master/jni/wprint/plugins/genPCLm/src/genPCLm.cpp) 生成。由于ndk 不熟，没有深入研究。
[这里](https://github.com/twaugh/hplip/tree/master/ppd/hpcups) 有所有惠普打印机的适配信息。
