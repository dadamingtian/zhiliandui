# 搭建过程中的问题
在搭建eps32的过程以及实现温度传感器的测温过程中，因为我们组对程序的了解与掌握程度以及其他的一些问题，
我们花了很长的时间在弄调试程序，最后借助第八组的大神完成了这次的任务。
我们遇到的问题有
1.在安装esp32的开发板时，并没有安装MQTT的开发板，导致开始的时候程序导入一直失败。
2.然后我们对老师给的开发板的用途以及怎样用不熟悉，导致没焊接就直接用了，结果可想而知，一定以失败告终。
3.我们了解到了元件少了杜邦线，于是在网上买了几十根来用。照着老师发的图来连接。
4.后来我们意识到我们仅仅焊接了一个开发板，而另一个开发板没有焊接但是是需要用到的，于是我们“连夜”赶往工程训练中心去做焊接工作。
5.在焊接工作中，我们辛勤劳作了两个小时，大战焊接器的精细时，终于完成焊接了，而后我们开开心心的回到宿舍。
6.然鹅，噩耗传来，当我们第二天去上传代码时，又一次怎么样也不成功，我们请教大神之后，才发现是我们焊接的那个开发板烧坏了
7.而且传感器是在使用中出现冒烟的情况，如此，我们好像又回到了起点
8.经历了一些波折后，我们借了其他组的开发板，在一番熟练的操作后，终于实现的温度的感应。
9.但是此时我们的代码仍有一些问题，我们再次请教了大神，经过一番修改后，我们成功了。
10.经过此次的实践，我们学到了很多东西，也是在发现问题后及时想想问题出在哪，并且借助他人的指点，完成了这一项伟大的任务。
# 解决问题后的搭建步骤
## 焊接
过程过于惨不忍睹，略过一下。
## 接线
![接线]（http://m.qpic.cn/psc?/V1196dws0Zlxmv/ruAMsa53pVQWN7FLK88i5qyhxo36q.QR.4dY3W95YnTDeWQmHbbL1ov28SQEoEq*zte3HEfcDrk5EGIrac6A2qAkX7ryCJhdF6xTQ0QgSD0!/mnull&bo=zwIYBgAAAAABB*M!&rf=photolist&t=5）
根据大神们的教程，接线过程还算简单。
## 烧录程序
由于看不懂代码，只能借鉴大佬们的代码。
网关代码：
```
#include <LoRaNow.h>
#include <WiFi.h>
#include <WebServer.h>
#include <StreamString.h>

#define MISO 19
#define MOSI 23
#define SCK 18
#define SS 5

#define DIO0 4

const char *ssid = "YZ";
const char *password = "101027@qwer";

WebServer server(80);

const char *script = "<script>function loop() {var resp = GET_NOW('loranow'); var area = document.getElementById('area').value; document.getElementById('area').value = area + resp; setTimeout('loop()', 1000);} function GET_NOW(get) { var xmlhttp; if (window.XMLHttpRequest) xmlhttp = new XMLHttpRequest(); else xmlhttp = new ActiveXObject('Microsoft.XMLHTTP'); xmlhttp.open('GET', get, false); xmlhttp.send(); return xmlhttp.responseText; }</script>";

void handleRoot()
{
  String str = "";
  str += "<html>";
  str += "<head>";
  str += "<title>ESP32 - LoRaNow</title>";
  str += "<meta name='viewport' content='width=device-width, initial-scale=1'>";
  str += script;
  str += "</head>";
  str += "<body onload='loop()'>";
  str += "<center>";
  str += "<textarea id='area' style='width:800px; height:400px;'></textarea>";
  str += "</center>";
  str += "</body>";
  str += "</html>";
  server.send(200, "text/html", str);
}

static StreamString string;

void handleLoRaNow()
{
  server.send(200, "text/plain", string);
  while (string.available()) // clear
  {
    string.read();
  }
}

void setup(void)
{

  Serial.begin(115200);

  WiFi.mode(WIFI_STA);
  if (ssid != "")
    WiFi.begin(ssid, password);
  WiFi.begin();
  Serial.println("");

  // Wait for connection
  while (WiFi.status() != WL_CONNECTED)
  {
    delay(500);
    Serial.print(".");
  }

  Serial.println("");
  Serial.print("Connected to ");
  Serial.println(ssid);
  Serial.print("IP address: ");
  Serial.println(WiFi.localIP());

  server.on("/", handleRoot);
  server.on("/loranow", handleLoRaNow);
  server.begin();
  Serial.println("HTTP server started");

  LoRaNow.setFrequencyCN(); // Select the frequency 486.5 MHz - Used in China
  // LoRaNow.setFrequencyEU(); // Select the frequency 868.3 MHz - Used in Europe
  // LoRaNow.setFrequencyUS(); // Select the frequency 904.1 MHz - Used in USA, Canada and South America
  // LoRaNow.setFrequencyAU(); // Select the frequency 917.0 MHz - Used in Australia, Brazil and Chile

  // LoRaNow.setFrequency(frequency);
  // LoRaNow.setSpreadingFactor(sf);
  // LoRaNow.setPins(ss, dio0);

  LoRaNow.setPinsSPI(SCK, MISO, MOSI, SS, DIO0); // Only works with ESP32

  while (!LoRaNow.begin())
  {
    Serial.println("LoRa init failed. Check your connections.");
    delay(5000);
  }

  LoRaNow.onMessage(onMessage);
  LoRaNow.gateway();
}

void loop(void)
{
  LoRaNow.loop();
  server.handleClient();
}

void onMessage(uint8_t *buffer, size_t size)
{
  unsigned long id = LoRaNow.id();
  byte count = LoRaNow.count();

  Serial.print("Node Id: ");
  Serial.println(id, HEX);
  Serial.print("Count: ");
  Serial.println(count);
  Serial.print("Message: ");
  Serial.write(buffer, size);
  Serial.println();
  Serial.println();

  if (string.available() > 512)
  {
    while (string.available())
    {
      string.read();
    }
  }

  string.print("Node Id: ");
  string.println(id, HEX);
  string.print("Count: ");
  string.println(count);
  string.print("Message: ");
  string.write(buffer, size);
  string.println();
  string.println();

  // Send data to the node
  LoRaNow.clear();
  LoRaNow.print("LoRaNow Gateway Message ");
  LoRaNow.print(millis());
  LoRaNow.send();
}
```
节点代码：
```
#include <LoRaNow.h>

//vspi for lora radio module
#define MISO 19
#define MOSI 23
#define SCK 18
#define SS 5

#define DIO0 4

#include <OneWire.h>
#include <DallasTemperature.h>

// GPIO where the DS18B20 is connected to
const int oneWireBus = 25;     

// Setup a oneWire instance to communicate with any OneWire devices
OneWire oneWire(oneWireBus);

// Pass our oneWire reference to Dallas Temperature sensor 
DallasTemperature sensors(&oneWire);

float tmp;

void setup() {
  Serial.begin(115200);
  Serial.println("LoRaNow Simple Node");
  sensors.begin();

   LoRaNow.setFrequencyCN(); // Select the frequency 486.5 MHz - Used in China
  // LoRaNow.setFrequencyEU(); // Select the frequency 868.3 MHz - Used in Europe
  // LoRaNow.setFrequencyUS(); // Select the frequency 904.1 MHz - Used in USA, Canada and South America
  // LoRaNow.setFrequencyAU(); // Select the frequency 917.0 MHz - Used in Australia, Brazil and Chile

  // LoRaNow.setFrequency(frequency);
  // LoRaNow.setSpreadingFactor(sf);
  // LoRaNow.setPins(ss, dio0);

   LoRaNow.setPinsSPI(SCK, MISO, MOSI, SS, DIO0); // Only works with ESP32

  if (!LoRaNow.begin()) {
    Serial.println("LoRa init failed. Check your connections.");
    while (true);
  }

  LoRaNow.onMessage(onMessage);
  LoRaNow.onSleep(onSleep);
  LoRaNow.showStatus(Serial);
}

void loop() {
  sensors.requestTemperatures(); 
  tmp = sensors.getTempCByIndex(0);
  LoRaNow.loop();
}


void onMessage(uint8_t *buffer, size_t size)
{
  Serial.print("Receive Message: ");
  Serial.write(buffer, size);
  Serial.println();
  Serial.println();
}

void onSleep()
{
  Serial.println("Sleep");
  Serial.println(tmp);
  delay(5000); // "kind of a sleep"
  Serial.println("Send Message");
  LoRaNow.print("LoRaNow Node Message ");
  //Serial.println("LoRaNow Message sended");
  LoRaNow.print(millis());
  LoRaNow.print("-");
  LoRaNow.print(tmp);
  LoRaNow.send();
}
```
## 运行结果
由于不会改代码所以结果与实际不符。
![运行结果](http://m.qpic.cn/psc?/V1196dws0Zlxmv/ruAMsa53pVQWN7FLK88i5uWuVMxMO9zg6Br.6luUVBfpEDtINRqcRht1Xwm11kBVaM39i.unzsmtP4OiaTyNphzXCD4fM5U*IPQwotnLUWo!/mnull&bo=zwPDAQAAAAADByw!&rf=photolist&t=5)
## 反思
看懂代码的能力太弱，焊接等基本操作不熟练，但让我们懂得了团队协作的重要性和大学对于我们工程素养的要求，更加激起了我们对这门课程的兴趣！
