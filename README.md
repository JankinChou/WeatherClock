# WeatherClock

## 项目说明

此项目是根据前人的开源项目的基础，制作出一个WIFI天气时钟成品。可以用于显示实时的日期、时间、天气和室外温度。程序说明部分有相应致谢。

## 使用说明

### 工程模式

产品通过CH340模块（USB转TTL）连接电脑，将开关1置于下端，然后可以通过乐鑫官网提供的串口助手下载我已提供的bin文件，也可以使用Arduino IDE对代码进行编辑下载。

### 运行模式

再没有开启工程模式的情况下，将开关2，置于下端开启运行模式，显示开机动画，然后显示相应的实时的日期、时间、天气和室外温度。

### 配网模式

在运行模式下，将开关1置于下端，进入配网模式，打开手机WIFI搜索名称为“ifi_link_tool”的WIFI，稍等手机会进行跳转，根据提示完成相应的配网操作。

## 程序说明

 *其中WiFi配网部分wifi_link_tool.h致谢B站发明控，时间和天气获取部分致谢天空之城；以上为程序中部分代码来源
 *天气由心知天气平台提供，个人用户免费注册
 *实现功能：可通过手机AP配网，连接WiFi获取实时天气和实时时钟
 *在码次程序之前没有学习过任何C++相关内容，没有使用过Arduino，只使用过ESP8266程序必然存在漏洞，请予以纠正谢谢
 *Jankin制作

## 程序源码

```c++
#include <wifi_link_tool.h>
#include <Wire.h>
#include "SSD1306Wire.h"
#include "images.h"
#include <ESP8266WiFi.h>
#include <ArduinoJson.h>                
#include <Time.h>
#include <Timezone.h>
#include "NTP.h"

int json_state;//天气返回编码
int json_state2;//温度返回值
String json_temp;

// 北京时间时区
#define STD_TIMEZONE_OFFSET +8    // Standard Time offset (-7 is mountain time)

TimeChangeRule mySTD = {"", First,  Sun, Jan, 0, STD_TIMEZONE_OFFSET * 60};
Timezone myTZ(mySTD, mySTD);

WiFiClient client;                       //创建一个网络对象

String key = "SpUwa33HxwZrSNTgP";   //心知天气的秘钥，可以自己去注册也可以用我的
String weizhi = "hefei";        //这里写你的地址

/*====================================液晶部分====================================*/
SSD1306Wire display(0x3C,14,12);   //SDA SCL
typedef void (*Demo)(void);


/*====================================画天气图标函数===============================*/
void drawzhongyu()
{
  display.drawXbm(drawposition, 0, WiFi_Logo_width, WiFi_Logo_height, zhongyu);
}

void drawdayu()
{
  display.drawXbm(drawposition, 0, WiFi_Logo_width, WiFi_Logo_height, dayu);
}

void drawxiaoyu()
{
  display.drawXbm(drawposition, 0, WiFi_Logo_width, WiFi_Logo_height, xiaoyu);
}

void drawqingtian()
{
  display.drawXbm(drawposition, 0, WiFi_Logo_width, WiFi_Logo_height, qingtian);
}

void drawduoyun() 
{
  display.drawXbm(drawposition, 0, WiFi_Logo_width, WiFi_Logo_height, duoyun);
}

void drawzhu() 
{
  display.drawXbm(32, 0, 64, 64, WiFi_Logo_bits);
}

void drawseconds_iconclear()
{
  display.drawXbm(0, 40, 24, 24, icon_clear);
}
void combination()
{
  display.drawXbm(16,0, WiFi_Logo_width, WiFi_Logo_height, qingtian);
  display.drawXbm(80, 0, WiFi_Logo_width, WiFi_Logo_height, duoyun);
  display.drawXbm(16, 32, WiFi_Logo_width, WiFi_Logo_height, xiaoyu);
  display.drawXbm(80,32, WiFi_Logo_width, WiFi_Logo_height, dayu);
}
Demo demos[] = {drawqingtian, drawduoyun, drawxiaoyu, drawzhongyu, drawdayu, drawzhu, drawseconds_iconclear,combination}; //6


void parseUserData(String content)
{
  const size_t capacity = JSON_ARRAY_SIZE(1) + JSON_OBJECT_SIZE(1) + 2 * JSON_OBJECT_SIZE(3) + JSON_OBJECT_SIZE(6) + 210;
  DynamicJsonBuffer jsonBuffer(capacity);
  JsonObject& root = jsonBuffer.parseObject(content);
  JsonObject& results_0 = root["results"][0];
  JsonObject& results_0_now = results_0["now"];

  //const char* results_0_now_text = results_0_now["text"];  // 天气情况
  int results_0_now_code = results_0_now["code"];  //天气情况识别码
  const char* results_0_now_temperature = results_0_now["temperature"];//气温

  display.setTextAlignment(TEXT_ALIGN_LEFT);
  display.setFont(ArialMT_Plain_24);

  json_temp = results_0_now_temperature;   
  json_state = results_0_now_code;
}


/*====================================调用api函数====================================*/
void apicmd()
{
  if (client.connect("api.seniverse.com", 80) == 1)    //服务连接
  {
    client.print("GET /v3/weather/now.json?key=");
    client.print(key);
    client.print("&location=");
    client.print(weizhi);
    client.print("&language=zh-Hans&unit=c HTTP/1.1\r\n");
    client.print("Host:api.seniverse.com\r\n");
    client.print("Accept-Language:zh-cn\r\n");
    client.print("Connection:close\r\n\r\n");                 //向心知天气的服务器发送请求。
    

    if (client.find("\r\n\r\n") == 1)                         //跳过返回的json数据头，
    {
      String json_from_server = client.readStringUntil('\n'); //读取返回的JSON数据
      Serial.println(json_from_server);                      //打印json数据
      parseUserData(json_from_server);                       //调用josn解析函数，并传参
    }

  }
  else
  {
    Serial.println("服务器连接失败");
    delay(5000);
  }
  client.stop();                                            //关闭HTTP
}


String Middle_minutes;
String Middle_hours;
/*====================================时间获取函数====================================*/
void updateDisplay(void) 
{
  TimeChangeRule *tcr;        // Pointer to the time change rule
  // Read the current UTC time from the NTP provider
  time_t utc = now();
  // Convert to local time taking DST into consideration
  time_t localTime = myTZ.toLocal(utc, &tcr);

  // Map time to pixel positions
  //int weekdays =   weekday(localTime);
  String days    =  (String) day(localTime);
  String months  = (String) month(localTime);
 // String years   = (String) year(localTime);
 // String seconds = (String) second(localTime);
  String minutes = (String) minute(localTime);
  String hours   = (String) hour(localTime); 

    /*串口输出时间*/

//  Serial.println("");
//  Serial.print("Current local time:");
//  Serial.print(days);
//  Serial.print("/");
//  Serial.print(months);
//  Serial.print("/");
//  Serial.print(years);
//  Serial.print(" - ");
//  Serial.print(hours);
//  Serial.print(":");
//  Serial.print(minutes);
//  Serial.print(":");
//  Serial.print(seconds);
//  Serial.print(" - ");
//  Serial.print(dayStr(weekdays));
//  Serial.println("");

  if (Middle_minutes != minutes)
  {
    display.clear();
    display.setTextAlignment(TEXT_ALIGN_LEFT);
    display.setFont(ArialMT_Plain_24);
    Serial.println("调用天气");
    apicmd();
    display.drawString(75, 40, json_temp);
    display.drawString(100, 40, "°C");
    

    /*====================================小时显示格式处理====================================*/
    if(hours=="0"){display.drawString(10, 5, "0");display.drawString(24, 5, hours);}
    else if(hours=="1"){display.drawString(10, 5, "0");display.drawString(24, 5, hours);}
    else if(hours=="2"){display.drawString(10, 5, "0");display.drawString(24, 5, hours);}
    else if(hours=="3"){display.drawString(10, 5, "0");display.drawString(24, 5, hours);}
    else if(hours=="4"){display.drawString(10, 5, "0");display.drawString(24, 5, hours);}
    else if(hours=="5"){display.drawString(10, 5, "0");display.drawString(24, 5, hours);}
    else if(hours=="6"){display.drawString(10, 5, "0");display.drawString(24, 5, hours);}
    else if(hours=="7"){display.drawString(10, 5, "0");display.drawString(24, 5, hours);}
    else if(hours=="8"){display.drawString(10, 5, "0");display.drawString(24, 5, hours);}
    else if(hours=="9"){display.drawString(10, 5, "0");display.drawString(24, 5, hours);}
    else{display.drawString(10, 5, hours);}
    
    display.drawString(38, 3, ":");
    
    /*====================================分钟显示格式处理====================================*/
    if(minutes=="0"){display.drawString(45, 5, "0");display.drawString(59, 5, minutes);}
    else if(minutes=="1"){display.drawString(45, 5, "0");display.drawString(59, 5, minutes);}
    else if(minutes=="2"){display.drawString(45, 5, "0");display.drawString(59, 5, minutes);}
    else if(minutes=="3"){display.drawString(45, 5, "0");display.drawString(59, 5, minutes);}
    else if(minutes=="4"){display.drawString(45, 5, "0");display.drawString(59, 5, minutes);}
    else if(minutes=="5"){display.drawString(45, 5, "0");display.drawString(59, 5, minutes);}
    else if(minutes=="6"){display.drawString(45, 5, "0");display.drawString(59, 5, minutes);}
    else if(minutes=="7"){display.drawString(45, 5, "0");display.drawString(59, 5, minutes);}
    else if(minutes=="8"){display.drawString(45, 5, "0");display.drawString(59, 5, minutes);}
    else if(minutes=="9"){display.drawString(45, 5, "0");display.drawString(59, 5, minutes);}
    else{display.drawString(45, 5, minutes);}
    
    /*====================================天气图标显示处理====================================*/
    if (json_state == 0){demos[0]();}
    else if (json_state > 0  && json_state < 4){demos[0]();}
    else if (json_state > 3  && json_state < 10){demos[1]();}
    else if (json_state > 9  && json_state < 14){demos[2]();}
    else if (json_state == 14){demos[3]();}
    else if (json_state > 14  && json_state < 19){demos[4]();}
    
    /*====================================月份显示格式处理====================================*/
    if(months=="10"){display.drawString(0, 40, months);}
    else if(months=="11"){display.drawString(0, 40, months);}
    else if(months=="12"){display.drawString(0, 40, months);}
    else{display.drawString(0, 40, "0");display.drawString(14, 40, months);}
    
    display.drawString(27, 40, "/");
    
    /*====================================日期显示格式处理====================================*/
    if(days=="1"){display.drawString(33, 40, "0");display.drawString(47, 40, days);}
    else if(days=="2"){display.drawString(33, 40, "0");display.drawString(47, 40, days);}
    else if(days=="3"){display.drawString(33, 40, "0");display.drawString(47, 40, days);}
    else if(days=="4"){display.drawString(33, 40, "0");display.drawString(47, 40, days);}
    else if(days=="5"){display.drawString(33, 40, "0");display.drawString(47, 40, days);}
    else if(days=="6"){display.drawString(33, 40, "0");display.drawString(47, 40, days);}
    else if(days=="7"){display.drawString(33, 40, "0");display.drawString(47, 40, days);}
    else if(days=="8"){display.drawString(33, 40, "0");display.drawString(47, 40, days);}
    else if(days=="9"){display.drawString(33, 40, "0");display.drawString(47, 40, days);}
    else{display.drawString(33, 40, days);}
    
    Middle_minutes = minutes;

  }
    display.display();
}


void setup()
{
  Serial.begin( 115200 );           /*开启串口*/

  rstb = 0;                         /* 重置io                     D3*/
  stateled = 2;                     /* 指示灯io                   D4*/
  Hostname = "WeatherClock";        /* 设备名称 允许中文名称 不建议太长 */
  wxscan = true;                    /* 是否被小程序发现设备 开启意味该设备具有后台 true开启 false关闭 */
  load();                           /* 初始化WiFi link tool */

  display.init();
  display.flipScreenVertically();//反向显示
  display.clear();
  demos[5]();
  display.display();

  udp.begin(LOCALPORT);

  if(link())
  {
   setSyncProvider(getNTPTime);
  }
  else
  {
   display.clear();
   display.setFont(ArialMT_Plain_16);
   display.drawString(14,4,"Weather Clock");
   display.drawString(10,28,"Please set WiFi");
   display.setFont(ArialMT_Plain_10);
   display.drawString(30,52,"Made by Jankin");
   display.display();
  }
}

time_t previousSecond = 0;
void loop()
{
  pant();              /* WiFi link tool 服务维持函数  请勿修改位置 */

  if (timeStatus() != timeNotSet) 
  {
    if (second() != previousSecond) 
    {
      previousSecond = second();
      updateDisplay();
      }
   }
   delay(1000);
}
```

