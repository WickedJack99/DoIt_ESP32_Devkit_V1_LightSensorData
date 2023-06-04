//@author WickedJack99
//@date 04.06.2023
//-------------------------------------------------------------------------------
//Libraries
//-------------------------------------------------------------------------------
#include <WiFi.h>
#include <NTPClient.h>
#include <WiFiClientSecure.h>
#include <HTTPClient.h>
#include <base64.h>

//-------------------------------------------------------------------------------
// Global variables
//-------------------------------------------------------------------------------

// 1000 milliseconds = 1 second
// 60 seconds = 1 minute
// 60 minutes = 1 hour
// 1000 * 60 * 60 = 3_600_000

//-------------------------------------------------------------------------------
// Delay time in milliseconds (1h)
//-------------------------------------------------------------------------------
const unsigned long DELAY = 3600000;

//-------------------------------------------------------------------------------
// value indicating gpio pin
//-------------------------------------------------------------------------------
const int GPIO34 = 34;

//-------------------------------------------------------------------------------
// ADC value
//-------------------------------------------------------------------------------
int iLuxResistorResolution = 10;

float fLuxValue = 0.0;

//-------------------------------------------------------------------------------
// Wifi settings
//-------------------------------------------------------------------------------
const char* WIFI_SSID = "";  // Enter SSID here
const char* WIFI_PASSWORD = "";  //Enter Password here

//-------------------------------------------------------------------------------
// Github settings
//-------------------------------------------------------------------------------
const char*  GITHUB_ADDRESS = "api.github.com";

const char* GITHUB_ROOT_CA = \
"-----BEGIN CERTIFICATE-----\n" \
"MIIDrzCCApegAwIBAgIQCDvgVpBCRrGhdWrJWZHHSjANBgkqhkiG9w0BAQUFADBh\n" \
"MQswCQYDVQQGEwJVUzEVMBMGA1UEChMMRGlnaUNlcnQgSW5jMRkwFwYDVQQLExB3\n" \
"d3cuZGlnaWNlcnQuY29tMSAwHgYDVQQDExdEaWdpQ2VydCBHbG9iYWwgUm9vdCBD\n" \
"QTAeFw0wNjExMTAwMDAwMDBaFw0zMTExMTAwMDAwMDBaMGExCzAJBgNVBAYTAlVT\n" \
"MRUwEwYDVQQKEwxEaWdpQ2VydCBJbmMxGTAXBgNVBAsTEHd3dy5kaWdpY2VydC5j\n" \
"b20xIDAeBgNVBAMTF0RpZ2lDZXJ0IEdsb2JhbCBSb290IENBMIIBIjANBgkqhkiG\n" \
"9w0BAQEFAAOCAQ8AMIIBCgKCAQEA4jvhEXLeqKTTo1eqUKKPC3eQyaKl7hLOllsB\n" \
"CSDMAZOnTjC3U/dDxGkAV53ijSLdhwZAAIEJzs4bg7/fzTtxRuLWZscFs3YnFo97\n" \
"nh6Vfe63SKMI2tavegw5BmV/Sl0fvBf4q77uKNd0f3p4mVmFaG5cIzJLv07A6Fpt\n" \
"43C/dxC//AH2hdmoRBBYMql1GNXRor5H4idq9Joz+EkIYIvUX7Q6hL+hqkpMfT7P\n" \
"T19sdl6gSzeRntwi5m3OFBqOasv+zbMUZBfHWymeMr/y7vrTC0LUq7dBMtoM1O/4\n" \
"gdW7jVg/tRvoSSiicNoxBN33shbyTApOB6jtSj1etX+jkMOvJwIDAQABo2MwYTAO\n" \
"BgNVHQ8BAf8EBAMCAYYwDwYDVR0TAQH/BAUwAwEB/zAdBgNVHQ4EFgQUA95QNVbR\n" \
"TLtm8KPiGxvDl7I90VUwHwYDVR0jBBgwFoAUA95QNVbRTLtm8KPiGxvDl7I90VUw\n" \
"DQYJKoZIhvcNAQEFBQADggEBAMucN6pIExIK+t1EnE9SsPTfrgT1eXkIoyQY/Esr\n" \
"hMAtudXH/vTBH1jLuG2cenTnmCmrEbXjcKChzUyImZOMkXDiqw8cvpOp/2PV5Adg\n" \
"06O/nVsJ8dWO41P0jmP6P6fbtGbfYmbW0W5BjfIttep3Sp+dWOIrWcBAI+0tKIJF\n" \
"PnlUkiaY4IBIqDfv8NZ5YBberOgOzW6sRBc4L0na4UU+Krk2U886UAb3LujEV0ls\n" \
"YSEY1QSteDwsOoBrp+uvFRTp2InBuThs4pFsiv9kuXclVzDAGySj4dzp30d8tbQk\n" \
"CAUw7C29C79Fv1C5qfPrmAESrciIxpg0X40KPMbp1ZWVbd4=\n" \
"-----END CERTIFICATE-----\n";

WiFiClientSecure* client;

unsigned long ulGitHubCommitCounter = 1;

String sGithubToken = "";

String sGitHubUser = "";
String sGitHubRepo = "";

//-------------------------------------------------------------------------------
// Time
//-------------------------------------------------------------------------------
// Define NTP Client to get time
WiFiUDP ntpUDP;
NTPClient timeClient(ntpUDP);

unsigned int auiDayLookupTable[12] = {31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31};

String sDate;

//-------------------------------------------------------------------------------
// Initialization
//-------------------------------------------------------------------------------
void setup()
{
  // Set baud rate to 115200
  Serial.begin(115200);
  delay(100);

  // Set wifi credits
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  // Connect to wi-fi network
  connectToWiFi();

  // Initialize a NTPClient to get time
  timeClient.begin();
  // Set offset time in seconds to adjust for your timezone, for example:
  // GMT +1 = 3600
  // GMT +8 = 28800
  // GMT -1 = -3600
  // GMT 0 = 0
  timeClient.setTimeOffset(3600);
}

//-------------------------------------------------------------------------------
// Main loop
//-------------------------------------------------------------------------------
void loop()
{
  // Re-/Connect to wifi
  connectToWiFi();

  // Get actual time
  String sTime = getTime();
  
  // Get value of gpio34 adc1_ch6
  int iADC1_6_Value = analogRead(GPIO34);
  
  // Calculate actual lux value
  fLuxValue = calculateLuxValue(iADC1_6_Value);

  // Send value with timestamp to github repo
  sendPUTRequest(fLuxValue, sTime);
  
  // Sleep for value DELAY (1 hour) to save energy
  delay(DELAY);
}

//-------------------------------------------------------------------------------
// Functions
//-------------------------------------------------------------------------------

//-------------------------------------------------------------------------------
// Connects to wifi
void connectToWiFi()
{
  // Check WiFi is connected to WiFi network
  if (WiFi.status() != WL_CONNECTED)
  {
    Serial.println("WiFi disconnected..");
    Serial.println("Trying to reconnect..");

    // Wait for connection response
    while (WiFi.status() != WL_CONNECTED)
    {
      delay(1000);
    }
    Serial.println("WiFi connected..!");
    Serial.print("Got IP: ");
    Serial.println(WiFi.localIP());
  }
}

//-------------------------------------------------------------------------------
// Gets realtime from WiFi device via udp (specified at beginning)
String getTime()
{
  // Update time
  while(!timeClient.update())
  {
    timeClient.forceUpdate();
  }
  // Seconds elapsed since 00:00:00 UTC 1 January 1970
  unsigned long ulEpoch = timeClient.getEpochTime();
  // Calculating parts of time
  unsigned long ulEpochSeconds = ulEpoch % 60;
  unsigned long ulEpochMinutes = (ulEpoch / 60) % 60;
  unsigned long ulEpochHours = (ulEpoch / (60 * 60)) % 24;

  // Setting global date
  // !!!
  // Performance optimization to prevent calling two times
  sDate = getDate(ulEpoch);
  
  // Returning time
  return ( sDate + "-" + String(ulEpochHours) + ":" + String(ulEpochMinutes) + ":" + String(ulEpochSeconds) );
}

String getDate(unsigned long ulEpoch)
{
  // Calculating parts of date
  unsigned long ulEpochDays = (ulEpoch / (60 * 60 * 24));
  unsigned long ulEpochMonths = (ulEpoch / (60 * 60 * 24 * 365)) % 12;
  unsigned long ulEpochYears = (ulEpoch / (60 * 60 * 24 * 365)) + 1970;

  ulEpochDays -= ((ulEpochYears - 1970) * 365);

  // Subtract leapdays
  for (int i = 1970; i < ulEpochYears; i++)
  {
    if (isLeapYear(i) == true)
    {
      ulEpochDays--;
    }
  }

  // Note: maybe still not working in leap years because doing % 365 but leap year has 366 days
  ulEpochDays %= 365;

  unsigned int uiIterator = 0;
  unsigned long ulDays = 0;
  // Subtract days of month till ulEpochDays is smaller / equal to days of a month
  while (ulEpochDays > ulDays)
  {
    ulDays = auiDayLookupTable[uiIterator];
    if (uiIterator == 1)
    {
      if (isLeapYear(ulEpochYears))
      {
        ulDays = 29;
      }
      // It could be possible that the condition of the loop is now wrong
      // Since ulDays was 28
      // 29 > 28
      // So it would normally subtract 28 and say it's 1st March
      // But since it's a leap year, it's 29th February!
      // So we have to break out of loop without subtracting
      if (ulDays == ulEpochDays)
      {
        uiIterator++;
        break;
      }
    }
    ulEpochDays -= ulDays;
    uiIterator++;
  }
  
  // Returning date
  return ( String((ulEpochDays + 1)) + "-" + String(ulEpochMonths + 1) + "-" + String(ulEpochYears) );
}

//-------------------------------------------------------------------------------
// Tests if overhanded year is a leap year, returns true if it is, false, if not
bool isLeapYear(unsigned long ulEpochYear)
{
  bool bIsLeapYear = false;
  if ((((ulEpochYear % 4) == 0 && !((ulEpochYear % 100) == 0))) || ((ulEpochYear % 400) == 0))
  {
    bIsLeapYear = true;
  }
  return bIsLeapYear;
}

//-------------------------------------------------------------------------------
// Get percentage of ADC value and multiplicate it with one percent of 10 lux
float calculateLuxValue(int iADC)
{
  float fADC1Percent = ((float)4095) / 100;
  float fADCPercent = ((float)iADC) / fADC1Percent;
  float fLux1Percent = ((float)iLuxResistorResolution) / 100;
  
  return fADCPercent * fLux1Percent;
}

//-------------------------------------------------------------------------------
void sendPUTRequest(float fLuxValue, String sTimestamp)
{
  HTTPClient oHttpClient;

  String sContent = "{\"timestamp\":\"" + sTimestamp + "\",\"value\":\"" + String(fLuxValue) + "\"}";

  String sEncodedFile = base64::encode(sContent);
  
  String sRequestBody = "{\"message\": \"Sensor data " + sTimestamp + "\",\"content\" :\"" + sEncodedFile + "\"}";

  int iContentLength = sRequestBody.length();

  client = new WiFiClientSecure;
  
  client->setCACert(GITHUB_ROOT_CA);

  // !!! sDate is set at loop when getting timestamp via getTime()
  if (oHttpClient.begin(*client, "https://api.github.com/repos/" + sGitHubUser + "/" + sGitHubRepo + "/contents/Measurements/" + sDate + "/Measurement_" + sTimestamp + ".json"))
  {
    oHttpClient.addHeader("Host", "api.github.com");
    oHttpClient.addHeader("Content-Type", "application/json");
    oHttpClient.addHeader("Authorization", ("Bearer " + sGithubToken));
    oHttpClient.addHeader("Content-Length", String(iContentLength));
    
    // start connection and send HTTP header
    int iHttpCode = oHttpClient.PUT(sRequestBody);

    if (iHttpCode > 0)
    {
    // Print response status code
    Serial.printf("PUT code: %d\n", iHttpCode);
      
      if (iHttpCode == HTTP_CODE_OK || iHttpCode == HTTP_CODE_MOVED_PERMANENTLY)
      {
        // Print server response payload
        String sPayload = oHttpClient.getString();
        Serial.println(sPayload);
      }
    }
    else
    {
      Serial.printf("PUT failed, error: %s\n", oHttpClient.errorToString(iHttpCode).c_str());
    }
      oHttpClient.end();
  }
  else
  {
    Serial.println("Unable to connect to server");
  }
}
