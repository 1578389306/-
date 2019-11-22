/**
* 日期:2019/05/25
* 功能：OLED显示天气屏
* 作者：单片机菜鸟
**/
#include <Arduino.h>
#include <ESP8266WiFi.h>
#include <ArduinoJson.h>
#include <U8g2lib.h>

#ifdef U8X8_HAVE_HW_SPI
#include <SPI.h>
#endif
#ifdef U8X8_HAVE_HW_I2C
#include <Wire.h>
#endif

#define LED D4
#define DEBUG //是否开启debug功能

#ifdef DEBUG
#define DebugPrintln(message)    Serial.println(message)
#else
#define DebugPrintln(message)
#endif

#ifdef DEBUG
#define DebugPrint(message)    Serial.print(message)
#else
#define DebugPrint(message)
#endif

#define WEATHER_CODE_DAY_SUN "0" //晴（国内城市白天晴）
#define WEATHER_CODE_NIGHT_SUN "1" //晴（国内城市夜晚晴）
#define WEATHER_CODE_DAY_SUN1 "2" //晴（国外城市白天晴）
#define WEATHER_CODE_NIGHT_SUN2 "3" //晴（国外城市夜晚晴）
#define WEATHER_CODE_CLOUDY "4" //多云
#define WEATHER_CODE_DAY_PARTLY_CLOUDY "5" //白天晴间多云
#define WEATHER_CODE_NIGHT_PARTLY_CLOUDY "6" //夜晚晴间多云
#define WEATHER_CODE_DAY_MOSTLY_CLOUDY "7" //白天大部多云
#define WEATHER_CODE_NIGHT_MOSTLY_CLOUDY "8" //夜晚大部多云
#define WEATHER_CODE_OVERCAST "9" //阴
#define WEATHER_CODE_SHOWER "10" //阵雨
#define WEATHER_CODE_THUNDERSHOWER "11" //雷阵雨
#define WEATHER_CODE_THUNDERSHOWER_WITH_HAIL "12" //雷阵雨伴有冰雹
#define WEATHER_CODE_LIGHT_RAIN "13" //小雨
#define WEATHER_CODE_MODERATE_RAIN "14" //中雨
#define WEATHER_CODE_HEAVY_RAIN "15" //大雨
#define WEATHER_CODE_STORM "16" //暴雨
#define WEATHER_CODE_HEAVY_STORM "17" //大暴雨
#define WEATHER_CODE_SEVERE_STORM "18" //特大暴雨
#define WEATHER_CODE_ICE_RAIN "19" //冻雨
#define WEATHER_CODE_SLEET "20" //雨夹雪
#define WEATHER_CODE_SNOW_FLURRY "21" //阵雪
#define WEATHER_CODE_LIGHT_SNOW "22" //小雪
#define WEATHER_CODE_MODERATE_SNOW "23" //中雪
#define WEATHER_CODE_HEAVY_SNOW "24" //大雪
#define WEATHER_CODE_SNOW_STORM "25" //暴雪

#define SUN_DAY 0
#define SUN_NIGHT 1
#define SUN_CLOUD 2
#define CLOUD 3
#define RAIN 4
#define THUNDER 5

//声明方法
bool autoConfig();
void smartConfig();
bool sendRequest(const char* host, const char* cityid, const char* apiKey);
bool skipResponseHeaders();
void readReponseContent(char* content, size_t maxSize);
void stopConnect();
void clrEsp8266ResponseBuffer(void);
bool parseUserData(char* content, struct UserData* userData);
void drawWeatherSymbol(u8g2_uint_t x, u8g2_uint_t y, uint8_t symbol);
void drawWeather(uint8_t symbol, int degree);
  
const unsigned long BAUD_RATE = 115200;// serial connection speed
const unsigned long HTTP_TIMEOUT = 5000;               // max respone time from server
const size_t MAX_CONTENT_SIZE = 500;                   // max size of the HTTP response
const char* host = "api.seniverse.com";
const char* APIKEY = "wcmquevztdy1jpca";        //API KEY
const char* city = "guangzhou";
const char* language = "zh-Hans";//zh-Hans 简体中文  会显示乱码

int flag = HIGH;//默认当前灭灯
U8G2_SSD1306_128X64_NONAME_F_HW_I2C u8g2(U8G2_R0, /* reset=*/ U8X8_PIN_NONE);

WiFiClient client;
char response[MAX_CONTENT_SIZE];
char endOfHeaders[] = "\r\n\r\n";

long lastTime = 0;
// 请求服务间隔
long Delay = 20000;

// 我们要从此网页中提取的数据的类型
struct UserData {
  char city[16];//城市名称
  char weather_code[4];//天气现象code（多云...）
  char temp[5];//温度
};

/**
* @Desc 初始化操作
*/
void setup() {
  Serial.begin(BAUD_RATE);
  pinMode(LED,OUTPUT);
  digitalWrite(LED, HIGH);

  WiFi.disconnect();
  if(!autoConfig()){
    smartConfig();
    DebugPrint("Connecting to WiFi");//写几句提示，哈哈
    while (WiFi.status() != WL_CONNECTED) {
    //这个函数是wifi连接状态，返回wifi链接状态
       delay(500);
       DebugPrint(".");
    }
  }
  
  delay(1000);
  digitalWrite(LED, LOW);
  DebugPrintln("IP address: ");
  DebugPrintln(WiFi.localIP());//WiFi.localIP()返回8266获得的ip地址
  lastTime = millis();
  u8g2.begin();  
  u8g2.enableUTF8Print();
  //使能软件看门狗的触发间隔
  ESP.wdtEnable(5000);
}
  
/**
* @Desc  主函数
*/
void loop() {
  while (!client.connected()){
     if (!client.connect(host, 80)){
         flag = !flag;
         digitalWrite(LED, flag);
         delay(500);
         //喂狗
         ESP.wdtFeed();
     }
  }

  if(millis()-lastTime>=Delay){
   //每间隔20s左右调用一次
     lastTime = millis();
     if (sendRequest(host, city, APIKEY) && skipResponseHeaders()) {
       clrEsp8266ResponseBuffer();
       readReponseContent(response, sizeof(response));
       UserData userData;
       if (parseUserData(response, &userData)) {
          showWeather(&userData);
       }
     }
  }
  
   //喂狗
   ESP.wdtFeed();
}

/**
* 自动连接20s 超过之后自动进入SmartConfig模式
*/
bool autoConfig(){
  WiFi.mode(WIFI_AP_STA);     //设置esp8266 工作模式
  WiFi.begin();
  delay(2000);//刚启动模块的话 延时稳定一下
  DebugPrintln("AutoConfiging ......");
  for(int index=0;index<10;index++){
    int wstatus = WiFi.status();
    if (wstatus == WL_CONNECTED){
      DebugPrintln("AutoConfig Success");
      DebugPrint("SSID:");
      DebugPrintln(WiFi.SSID().c_str());
      DebugPrint("PSW:");
      DebugPrintln(WiFi.psk().c_str());
      return true;
    }else{
      DebugPrint(".");
      delay(500);
      flag = !flag;
      digitalWrite(LED, flag);
    } 
  }
  DebugPrintln("AutoConfig Faild!");
  return false;
}

/**
* 开启SmartConfig功能
*/
void smartConfig()
{
  WiFi.mode(WIFI_STA);
  delay(1000);
  DebugPrintln("Wait for Smartconfig");
  // 等待配网
  WiFi.beginSmartConfig();
  while (1){
    DebugPrint(".");
    delay(200);
    flag = !flag;
    digitalWrite(LED, flag);
    
    if (WiFi.smartConfigDone()){
      //smartconfig配置完毕
      DebugPrintln("SmartConfig Success");
      DebugPrint("SSID:");
      DebugPrintln(WiFi.SSID().c_str());
      DebugPrint("PSW:");
      DebugPrintln(WiFi.psk().c_str());
      WiFi.mode(WIFI_AP_STA);     //设置esp8266 工作模式
      WiFi.setAutoConnect(true);  // 设置自动连接
      break;
    }
  }
}

/**
* @发送请求指令
*/
bool sendRequest(const char* host, const char* cityid, const char* apiKey) {
  // We now create a URI for the request
  //心知天气
  String GetUrl = "/v3/weather/now.json?key=";
  GetUrl += apiKey;
  GetUrl += "&location=";
  GetUrl += city;
  GetUrl += "&language=";
  GetUrl += language;
  // This will send the request to the server
  client.print(String("GET ") + GetUrl + " HTTP/1.1\r\n" +
               "Host: " + host + "\r\n" +
               "Connection: close\r\n\r\n");
  DebugPrintln("create a request:");
  DebugPrintln(String("GET ") + GetUrl + " HTTP/1.1\r\n" +
               "Host: " + host + "\r\n" +
               "Connection: close\r\n");
  delay(1000);
  return true;
}
 
/**
* @Desc 跳过 HTTP 头，使我们在响应正文的开头
*/
bool skipResponseHeaders() {
  // HTTP headers end with an empty line
  bool ok = client.find(endOfHeaders);
  if (!ok) {
    DebugPrintln("No response or invalid response!");
  }
  return ok;
}
 
/**
* @Desc 从HTTP服务器响应中读取正文
*/
void readReponseContent(char* content, size_t maxSize) {
  size_t length = client.readBytes(content, maxSize);
  delay(100);
  DebugPrintln("Get the data from Internet!");
  content[length] = 0;
  DebugPrintln(content);
  DebugPrintln("Read data Over!");
  client.flush();//这句代码需要加上  不然会发现每隔一次client.find会失败
}
  
// 关闭与HTTP服务器连接
void stopConnect() {
  client.stop();
}
 
void clrEsp8266ResponseBuffer(void){
    memset(response, 0, MAX_CONTENT_SIZE);      //清空
}

bool parseUserData(char* content, struct UserData* userData) {
//    -- 根据我们需要解析的数据来计算JSON缓冲区最佳大小
//   如果你使用StaticJsonBuffer时才需要
//    const size_t BUFFER_SIZE = 1024;
//   在堆栈上分配一个临时内存池
//    StaticJsonBuffer<BUFFER_SIZE> jsonBuffer;
//    -- 如果堆栈的内存池太大，使用 DynamicJsonBuffer jsonBuffer 代替
  DynamicJsonBuffer jsonBuffer;
  
  JsonObject& root = jsonBuffer.parseObject(content);
  
  if (!root.success()) {
    Serial.println("JSON parsing failed!");
    return false;
  }
   
  //复制我们感兴趣的字符串
  strcpy(userData->city, root["results"][0]["location"]["name"]);
  strcpy(userData->weather_code, root["results"][0]["now"]["code"]);
  strcpy(userData->temp, root["results"][0]["now"]["temperature"]);
  //  -- 这不是强制复制，你可以使用指针，因为他们是指向“内容”缓冲区内，所以你需要确保
  //   当你读取字符串时它仍在内存中
  return true;
}

/**
 * 根据天气接口返回的数据判断显示
 */
void showWeather(struct UserData* userData){
  if(strcmp(userData->weather_code,WEATHER_CODE_DAY_SUN) == 0 
  || strcmp(userData->weather_code,WEATHER_CODE_DAY_SUN1) == 0){
    drawWeather(SUN_DAY,userData->temp,userData->city);
  }else if(strcmp(userData->weather_code,WEATHER_CODE_NIGHT_SUN) == 0 
  || strcmp(userData->weather_code,WEATHER_CODE_NIGHT_SUN2) == 0 ){
    drawWeather(SUN_NIGHT,userData->temp,userData->city);
  }else if(strcmp(userData->weather_code,WEATHER_CODE_DAY_PARTLY_CLOUDY) == 0 
  || strcmp(userData->weather_code,WEATHER_CODE_NIGHT_PARTLY_CLOUDY) == 0 ){
    drawWeather(SUN_CLOUD,userData->temp,userData->city);
  }else if(strcmp(userData->weather_code,WEATHER_CODE_CLOUDY) == 0 
  || strcmp(userData->weather_code,WEATHER_CODE_DAY_MOSTLY_CLOUDY) == 0 
  || strcmp(userData->weather_code,WEATHER_CODE_NIGHT_MOSTLY_CLOUDY) == 0 
  || strcmp(userData->weather_code,WEATHER_CODE_OVERCAST) == 0){
    drawWeather(CLOUD,userData->temp,userData->city); 
  }else if(strcmp(userData->weather_code,WEATHER_CODE_SHOWER) == 0 
  || strcmp(userData->weather_code,WEATHER_CODE_LIGHT_RAIN) == 0 
  || strcmp(userData->weather_code,WEATHER_CODE_MODERATE_RAIN) == 0 
  || strcmp(userData->weather_code,WEATHER_CODE_HEAVY_RAIN) == 0 
  || strcmp(userData->weather_code,WEATHER_CODE_STORM) == 0
  || strcmp(userData->weather_code,WEATHER_CODE_HEAVY_STORM) == 0 
  || strcmp(userData->weather_code,WEATHER_CODE_SEVERE_STORM) == 0){
    drawWeather(RAIN,userData->temp,userData->city);
  }else if(strcmp(userData->weather_code,WEATHER_CODE_THUNDERSHOWER) == 0 
  || strcmp(userData->weather_code,WEATHER_CODE_THUNDERSHOWER_WITH_HAIL) == 0){
    drawWeather(THUNDER,userData->temp,userData->city);
  }else{
    drawWeather(CLOUD,userData->temp,userData->city); 
  }
}

void drawWeather(uint8_t symbol, char* degree,char* city)
{
  DebugPrintln(city);
  u8g2.clearBuffer();         // clear the internal memory
  //绘制天气符号
  drawWeatherSymbol(0, 48, symbol);
  //绘制温度
  u8g2.setFont(u8g2_font_logisoso32_tf);
  u8g2.setCursor(48+3, 42);
  u8g2.print(degree);
  u8g2.print("°C");   // requires enableUTF8Print()
  u8g2.setFont(u8g2_font_unifont_t_chinese3);

  u8g2_uint_t strWidth = u8g2.getUTF8Width(city);
  u8g2_uint_t displayWidth = u8g2.getDisplayWidth();
  
  u8g2.setCursor(displayWidth - strWidth - 5, 60);
  u8g2.print(city);
  u8g2.sendBuffer();        // transfer internal memory to the display
}

/**
 * 绘制天气符号
 */
void drawWeatherSymbol(u8g2_uint_t x, u8g2_uint_t y, uint8_t symbol)
{
  // fonts used:
  // u8g2_font_open_iconic_embedded_6x_t
  // u8g2_font_open_iconic_weather_6x_t
  // encoding values, see: https://github.com/olikraus/u8g2/wiki/fntgrpiconic
  switch(symbol)
  {
    case SUN_DAY://太阳
      u8g2.setFont(u8g2_font_open_iconic_weather_6x_t);
      u8g2.drawGlyph(x, y, 69); 
      break;
    case SUN_NIGHT://太阳
      u8g2.setFont(u8g2_font_open_iconic_weather_6x_t);
      u8g2.drawGlyph(x, y, 66); 
      break;  
    case SUN_CLOUD://晴间多云
      u8g2.setFont(u8g2_font_open_iconic_weather_6x_t);
      u8g2.drawGlyph(x, y, 65); 
      break;
    case CLOUD://多云
      u8g2.setFont(u8g2_font_open_iconic_weather_6x_t);
      u8g2.drawGlyph(x, y, 64); 
      break;
    case RAIN://下雨
      u8g2.setFont(u8g2_font_open_iconic_weather_6x_t);
      u8g2.drawGlyph(x, y, 67); 
      break;
    case THUNDER://打雷
      u8g2.setFont(u8g2_font_open_iconic_embedded_6x_t);
      u8g2.drawGlyph(x, y, 67);
      break;      
  }
}
