---
title: 4G hotspot device battery indicator
tags: python build
layout: article
aside:
  toc: true
---


Today we will be building a python script to show Huawei 4G Hostspot battery percentage in mac status bar and which will also play an alert when the battery is low.
<!--more-->

## Why
I was in a zoom meeting the other day and the call got abruptly disrupted because my wifi dongle got turned off due to low battery. Unfortunately there is no plug point available where i get the best network coverage so i can't leave the dongle plugged in to power outlet. You may suggest me to get a extension cord rather than building a specific tool for this , but where is the fun in it :-)

## The status page
The dongle provides a simple web page to show status and to modify settings, this can be accessed by visiting http://192.168.2.1 The landing page contains basic information showing battery, wifi status and network coverage.

<img src="/images/huawei-4g-hostspot-battery-indicator/status.png"/>

Luckily to show this basic info it does not require any password. When battery status changes the icon would change accordingly, so my initial though was to use python requests library and find the battery status icon to get the percentage. The page disables right click, but you can use command-option-I to open developer setting in chrome. Inspecting the battery icon shows the below src.

~~~ html
<li id="battery_gif" class="nav07">
  <img onload="fixPNG(this)" src="../res/battery_level_4.png" 0="" no-repeat="">
</li>
~~~

I wrote this small snippet, to grab the content of the status page.
~~~ python
import requests
resp = requests.get('http://192.168.2.1/html/home.html')
print(resp.text)
~~~
But this just had some javascript functions with some html and had no data, so the status page is rendered at client side, there had to be some network calls to fetch additional data. We could use selenium to load the page and scrape the battery status from there, but i wanted to keep it simple and light weight. A quick peek at networks tab made it clear. There were multiple calls to different endpoint to get specific data, the endpoints were self explainatory.

<img src="/images/huawei-4g-hostspot-battery-indicator/network-logs.png"/>

## Endpoint to get device status
So a get request to http://192.168.2.1/api/monitoring/status would return all the status details in xml format. I wish every device and service would provide a rest API to tinker and customize. So lets modify the above snippet to send a get request to the new url.

~~~ python
import requests
resp = requests.get('http://192.168.2.1/api/monitoring/status')
print(resp.text)
~~~

But the response was different here, it was an error.
~~~ xml
<?xml version="1.0" encoding="UTF-8"?>
<error>
  <code>125002</code>
  <message></message>
</error>
~~~
I copied the curl request from dev tool, to see whether there were some other fields/headers being sent and there it was, the session id in the cookie.

## Dealing with sessions
A quick googling showed that requests library supports creating and using sessions for sending http requests. Just needed to modify a line in our snippet. But the catch is session id will be initialized on the home page only and not on api endpoint. So we need to send a request to home page url and then the status endpoint.

~~~ python
import requests
req = requests.Session()
req.get('http://192.168.2.1')
resp = req.get('http://192.168.2.1/api/monitoring/status')
print(resp.text)
~~~

## Parsing xml and alerting on low battery
We use ElementTree XML library to parse XML, the modified code to print battery percent is below

~~~ python
import requests
import xml.etree.ElementTree as ET

req = requests.Session()
req.get('http://192.168.2.1')
resp = req.get('http://192.168.2.1/api/monitoring/status')

xml_resp = ET.fromstring(resp.text)
batt_percent = int(xml_resp.find('BatteryPercent').text)
print(batt_percent)
~~~

Now that we have battery percentage available, lets notify on low battery percentage. I did not want to have another audio file in the snippet and wanted to find a very simple way of notify with sound. There was one way where we can have base64 encoded beep sound in the snippet and play that to notify. But i found a <a href="https://en.wikipedia.org/wiki/Bell_character">interesting fact</a> that printing '\a'  to unix based terminal played a kind of ding sound which was more than enough for our need.

Here is the new snippet

~~~ python
import requests
import xml.etree.ElementTree as ET
from time import sleep

req = requests.Session()

while true:
  req.get('http://192.168.2.1')
  resp = req.get('http://192.168.2.1/api/monitoring/status')

  xml_resp = ET.fromstring(resp.text)
  batt_percent = int(xml_resp.find('BatteryPercent').text)

  if  batt_percent<=25:
    #beep 5 times
    print('\a'*5)

~~~

## Status bar app with rumps

Now that we know it works, lets clean this up and turn it into an status bar app. We will be using <a href="https://github.com/jaredks/rumps">rumps</a> library. Rumps allows us to set the icon and title of the status bar app at runtime. We will be using icon to indicate whether the laptop is connected to hotspot device and title of status bar will hold the percentage.

~~~ python
import xml.etree.ElementTree as ET
from time import sleep

import requests
import rumps


class BatteryStatusApp(rumps.App):

    connected_icon = 'connected_icon.png'
    disconnected_icon = 'disconnected_icon.png'
    url = 'http://192.168.1.1'
    status_url = url + "/api/monitoring/status"
    timeouts = (3, 3)
    req = None
    threshold = 30

    # initialize session
    def init_session(self):
        self.req = requests.Session()
        self.req.get(self.url)

    def get_battery_status(self, retry_count):
        """
        function which returns battery charging status and battery percent status
        :param retry_count: how many times to retry getting status before rasing error
        """

        resp = self.req.get(self.status_url, timeout=self.timeouts)
        xml_resp = ET.fromstring(resp.text)

        # if error, try re initializing session
        if xml_resp.tag == 'error':
            if retry_count >= 0:
                self.init_session()
                return self.get_battery_status(retry_count-1)
            else:
                raise SessionInitializationError

        return bool(int(xml_resp.find('BatteryStatus').text)), int(xml_resp.find('BatteryPercent').text)

    def __init__(self):
        super(BatteryStatusApp, self).__init__("", icon=self.disconnected_icon)
        self.req = requests.Session()

    @rumps.timer(60)
    def update_battery(self, sender):
        try:
            battery_charging, battery_percent = self.get_battery_status(3)
            print(battery_percent)
            print(battery_charging)
            if battery_percent < self.threshold and not battery_charging:
                rumps.notification(
                    "Battery Status", "Battery low", str(battery_percent))
            self.title = str(battery_percent)+"%"
            self.icon = self.connected_icon
        except Exception as e:
            self.title = ""
            self.icon = self.disconnected_icon


class NoConnectionError(Exception):
    """ Unable to get battery status, may not be connected to hotspot"""


class SessionInitializationError(Exception):
    """Unable to get battery status"""


if __name__ == "__main__":
    BatteryStatusApp().run()
~~~

Thanks to rumps, the code is pretty simple. In constructor we initialize the session icon and title. We have just added some timeouts and error handling to our above code. Rather than calling req.get('http://192.168.2.1') everytime, we call it during initialization and when there is an error. We update the battery status every 60 seconds by using inbuilt rumps decorator

~~~ python
    @rumps.timer(60)
    def update_battery(self, sender):
      ....
~~~

And when battery status is low, we send a notification every minute untill the device is plugged in. Here's how it looks.


