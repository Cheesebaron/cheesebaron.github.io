---
layout: post
title: Make Your Own Azure DevOps Build Light
author: Tomasz Cielecki
comments: true
date: 2019-02-09 16:00:00 +0100
tags:
- Azure Functions
- Azure DevOps
- Xamarin
- PlatformIO
---

Click to watch the YouTube video:
[![ESP8266 Azure DevOps build light](http://img.youtube.com/vi/wo0SESEObrU/0.jpg)](http://www.youtube.com/watch?v=wo0SESEObrU)

I had a colleague at a previous work place, which had made a Philips Hue light, light up red when our bamboo build of our Xamarin Apps was failing, light up green if everything was OK. This is a pretty good way to have a visual indicator in the entire team if something is being checked in which is failing our builds. I decided to give this a go as well. However, with a Micro Controller Unit (MCU) instead, since it costs much less than a Philips Hue setup and it will require less moving parts. Philips Hue would need something else on the network telling it which color to switch to. With the MCU I would just need that.

I have had spoken to a friend about making such a light. He has been dabbling a bit himself with home automation and MCUs himself. He told me he had the right stuff for me to try out and play with. So, he have me a WeMos D1 mini, a ESP8266 based MCU, a NeoPixel ring (a.k.a. W2812) with 8, individually controllable LEDs and a little box to house it in. He himself had made a remotely controllable night lamp out of similar parts.

So my journey began into the world of MCUs and how to control the LED ring I have had gotten. Just like with mobile Apps, there are so many possibilities. A vast amount of libraries, frameworks and operating systems for all sorts of purposes. I fell over something called [PlatformIO][platformio], which allows me to do everything from Visual Studio Code! Writing the code, building it, deploying it, debugging it by stepping around in the code etc. Additionally it has a library manager for libraries that other people have written. Neat!

OK! Let us get started making our own build light. Getting the status from Azure DevOps can be done through their [REST API calling the Builds API][devopslist]. In order to do this you need to generate a Personal Access Token, which you include in the header for the GET request. You can read more about how you authenticate [in the REST API docs][restapidocs]. In my case I have 4 build agents that can run at any given time. So I want the top 4 builds and see what the status is for them.
This can be done by querying the following URL: `https://dev.azure.com/{organization}/{project}/_apis/build/builds?$top={top}&api-version=5.0`, where `{organization}` is the name of your organization, `{project}` is the ID of the project you want to query within and `{top}` being the count of items I want in return. You can decrease the scope further down to specific build definitions and more. However, project and organization are required parameters.

When quering this data from Azure DevOps I quickly realized that the returned payload was _huge_! My ESP8266 only has a couple of MB of RAM and deserializing that huge piece of payload on the MCU was not working at all. Right! Now what? Since I cannot tell the Azure DevOps query that I am only interested in the status of the last 4 builds and not in any of the other properties I had to introduce some middleware of some sort. Just needed something quick without too many bells and whistles and to do minimal work on the MCU itself.

Azure Functions to the rescue! I quickly whipped up a simple Azure Function that gets the status of the builds and returns the following to my MCU:

- _failing_ if _any_ of the last 4 builds had failed
- _running_ if _any_ of the last 4 builds are running
- _finished_ if _all_ the last 4 builds have completed and succeeded

So getting the last 4 builds amounts to this little piece of code:

```csharp
private static async Task<IEnumerable<Build>> GetLatestBuild(int top = 4)
{
    var client = EnsureHttpClient();
        
    var url = $"https://dev.azure.com/{organization}/{project}/_apis/build/builds?$top={top}&api-version=5.0";
    
    var response = await client.GetAsync(url).ConfigureAwait(false);
    using (response)
    {
        if (response.IsSuccessStatusCode)
        {
            var responseStream = await response.Content.ReadAsStreamAsync().ConfigureAwait(false);
            using (responseStream)
            using (var streamReader = new StreamReader(responseStream))
            using (var jsonReader = new JsonTextReader(streamReader))
            {
                var serializer = new JsonSerializer();
                var results = serializer.Deserialize<Builds>(jsonReader);
                
                return results.Builds.Take(4);
            }
        }
    }
    
    return null;
}
```

From those builds I can now check the status with a little bit of LINQ gymnastics:

```csharp
var anyFailed = lastBuild.Any(b => b.Status?.ToLowerInvariant() == "failed");
var anyRunning = lastBuild.Any(b => b.Result.ToLowerInvariant() == "inprogress");
var allFinished = lastBuild.All(b => b.Result.ToLowerInvariant() == "succeeded" && b.Status?.ToLowerInvariant() == "completed");
```

Then return the status depending of the status of these. For these statuses I want to light up the LEDs:
- Red if returned status is _failed_
- Blue if returned status is _running_
- Green if returned status is _finished_

OK. Now I can get a small payload for the MCU to chew on. To communicate with the Azure Function, we need a HTTP client of some sort for this I am including the `WiFiClientSecure.h` headers, these come with with PlatformIO when targeting the ESP8266. The code for downloading the status is a little bit more involved than you might be used to from C# or other managed languages.

First you need to connect to the host, in my case: `buildlight.azurewebsites.net`. The port is `443` since it is HTTPS.

```cpp
WiFiClientSecure client;

bool connect() {
  Serial.print("connecting to ");
  Serial.println(host);

  bool ok = client.connect(host, httpsPort);

  Serial.println(ok ? "Connected Http" : "Connection Http Failed!");
  return ok;
}
```

When the connection is all good we can then send our request. The request in this case is whatever comes after the host part of the url: `/api/{FunctionName}?code={apiCode}` where `{FunctionName}` is the name of the function you are calling and `{code}` the code generated in the Azure Portal for calling your function. So we are sending a request that looks like:

```
GET {request} HTTP/1.1
Host: buildlight.azurewebsites.net
User-Agent: Cheesebulb/1.0
Connection: close
```

```cpp
void sendRequest() {
  client.print("GET ");
  client.print(request);
  client.println(" HTTP/1.1");
  client.print("Host: ");
  client.println(host);
  client.println("User-Agent: Cheesebulb/1.0");
  client.println("Connection: close");
  client.println();
}
```

Now we are only interested in the body of the response and not the headers, so we skip these:

```cpp
void skipHeaders() {
  while (client.connected()) {
    String line = client.readStringUntil('\n');
    if (line == '\r') {
      Serial.println("headers received");
      break;
    }
  }
}
```

Skipping these is reading until you get a carriage return `\r`. Now reading the rest will be the body of the request.

```cpp
String status = client.readStringUntil('\n');
```

Now that we have the status let us shine some light!

The NeoPixels work by providing 5V to a 5V pin and ground to a GND pin. They usually have a DI and DO pin, which are how you tell them which color they should light. The DI pin is the input pin and the DO pin is to chain it to more pixels. So essentially you are only communicating through one pin. For the NeoPixels this happens by sending a 800Hz digital signal on that pin, which can address each individual pixel, neat! The more pixels, then longer it takes to update the pixels. However, in my case with 9 pixels, this does not matter at all!
Before you start powering your NeoPixels, I suggest you [read some general best practices about these][adafruit]. You can damage them if not handled correctly. In my case I added a 1000µF capacitor between the 5V and ground pin to eat up any surges. This is especially importan if you power your NeoPixels using a DC adapter. However, this also helps if you need to change a lot of pixels at the same time and the amount of current drawn by the pixels suddenly changes.

To control the NeoPixels through the MCU I use the NeoPixelBus library. I have connected my NeoPixel ring to the `RX` pin. You might notice there are a lot of other digital output pins on the ESP8266. However, they operate at 3.3V and the NeoPixel require a 5V signal. The `RX` pin on my module is different and is in normal circumstances used for serial input to the MCU. However the `NeoPixelBus` library [supports sending data on this pin using Direct Memory Access (DMA) mode][dmamode]. It requires more memory, but uses significantly less CPU.

In any case, on the ESP8266 to change colors on the LEDs I use the following code.

```cpp
const uint8_t PixelCount = 9;
const uint8_t PixelPin = 0; // doesn't matter what you put here

NeoPixelBus<NeoGrbFeature, NeoEsp8266Dma800KbpsMethod> strip(PixelCount, PixelPin);

// define some colors
RgbColor off(0, 0, 0);
RgbColor red(255, 0, 0);
RgbColor green(0, 255, 0);
RgbColor blue(0, 0, 255);
```

Setting the color off a pixel is just a matter of calling:

```cpp
strip.SetPixelColor(pixelIndex, color);
```

So setting pixel 2 to green you call: `strip.SetPixelColor(1, green)`, the index starts at 0.

If you are interested in seeing the rest of the code I wrote for this little project you can [check out this gist with all of it][gist]. 

Here is a beauty shot of the device.

[![picture of ESP8266 with a NeoPixel ring in a box][image_thumb]][image]

With some basic programming knowledge and a MCU and some LEDs you can make this project on your own. I have only scratched the surface with what you can use a MCU for. I might return in the future with more fun projects. But for now, I can now see a nice visual status of my Xamarin App builds when I am at the office, without having to open a web page and check it there. In addition I have ordered a big white salt crystal which is going to rest over the box so it will act as both a build light and a piece of decoration for the office!

The cost of this little project is around $10, which is significantly less than getting started with Philips Hue or some other remotely controlled light.

[platformio]: https://platformio.org/ "PlatformIO open source ecosystem for IoT development"
[devopslist]: https://docs.microsoft.com/en-us/rest/api/azure/devops/build/builds/list "Azure DevOps documentation for Builds list"
[restapidocs]: https://docs.microsoft.com/en-us/rest/api/azure/devops/ "Azure DevOps REST API docs"
[adafruit]: https://learn.adafruit.com/adafruit-neopixel-uberguide/best-practices "NeoPixel General Best Practices"
[neopixelbus]: https://github.com/Makuna/NeoPixelBus "NeoPixelBus library on GitHub"
[dmamode]: https://github.com/Makuna/NeoPixelBus/wiki/ESP8266-NeoMethods#neoesp8266dma800kbpsmethod "NeoEsp8266Dma800KbpsMethod documentation"
[gist]: https://gist.github.com/Cheesebaron/a302d267e8f20eabee6c1123c96e8899
[image_thumb]: {{ site.url }}/public/images/build-light/box_thumb.jpg
[image]: {{ site.url }}/public/images/build-light/box.jpg
