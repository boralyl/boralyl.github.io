---
title: "Automating Hue Labs Formulas"
excerpt: >
  How to toggle Hue Labs Formulas via the local API.
categories:
  - home automation
tags:
  - hue
  - home automation
  - homeassistant
---

[Phillips Hue Labs](https://labs.meethue.com) provide many interesting formulas
that provide functionality the app iteslf can not do.  This functionality is
really useful and some of these formulas expose toggles in a `Hue Labs Controls`
panel.  This panel lets you switch the formula on and off.

The [Phillips Hue API](https://developers.meethue.com/develop/hue-api/) is really
well documented, so I expected the task of toggling a hue labs formula to be very
straight-forward.  To my dismay, the docs do not provide any information on how
to interact with any of the Hue Labs formulas.

Fortunately, there is a way to toggle these formula controls via the local API.
I give credit to [this post on reddit](https://www.reddit.com/r/Hue/comments/bstpkz/automate_hue_lab/eufeaca/)
 for information on how to accomplish this task.

 The first step is to query your local API's resourcelinks to find the Hue Labs
 Formula that you are interested in toggling.  Replace your Hue Bridge IP Address
 and your local username in the [httpie](https://httpie.org/) command below. I
 modified the output for brevity.

 ```bash
$ http http://192.168.1.2/api/<your-user-name>/resourcelinks

{
    "1234": {
        "classid": 2,
        "description": "xi3hg844QB2iN23DPO3K_A:2:4rK80TTxqDFYZvS3+pY12QHPUdT",
        "links": [
            "/rules/1",
            "/rules/2",
            "/schedules/10",
            "/schedules/11",
            "/sensors/15",
            "/scenes/hhhGmoHbTfzD0Gs",
            "/scenes/p97cGGHbvMYPIE0",
            "/groups/12",
            "/groups/13"
        ],
        "name": "Advanced presence mimicking",
        "owner": "93kd9a03-917b-43d9-1133-ff81e00003e6",
        "recycle": true,
        "type": "Link"
    }
}
```

Find the formula that you want to toggle, in my case I was looking for the
[Advanced presence mimicking](https://labs.meethue.com/formulas/huelabs/presence-mimicking)
formula.  In the `links` key there are a list of endpoints.  Find the one that
begins with `/sensors/`.  The ID in this endpoint will be used to toggle the formula.
In the example above we want to modify the sensor with the ID of `15`.

To toggle this formula do a HTTP PUT request to `/api/<your-user-name>/sensors/<sensor-id>/state`.
The body should include a single key, `status` with the value of `0` to disable it
and `1` to enable it.

```bash
$ http -v PUT http://192.168.1.174/api/<your-user-name>/sensors/<sensor-id>/state status:=1
PUT /api/<your-user-name>/sensors/15/state HTTP/1.1
Accept: application/json
Accept-Encoding: gzip, deflate
Connection: keep-alive
Content-Length: 13
Content-Type: application/json
Host: 192.168.1.174
User-Agent: HTTPie/0.9.2

{
    "status": 1
}

HTTP/1.1 200 OK
Access-Control-Allow-Credentials: true
Access-Control-Allow-Headers: Content-Type
Access-Control-Allow-Methods: POST, GET, OPTIONS, PUT, DELETE, HEAD
Access-Control-Allow-Origin: *
Access-Control-Max-Age: 3600
Cache-Control: no-store, no-cache, must-revalidate, post-check=0, pre-check=0
Connection: close
Content-Type: application/json
Date: Sun, 13 Oct 2019 02:12:45 GMT
Expires: Mon, 1 Aug 2011 09:00:00 GMT
Pragma: no-cache
Server: nginx

[
    {
        "success": {
            "/sensors/15/state/status": 1
        }
    }
]
```

You can now use this HTTP call in scripts and automations within home-assistant
and/or node-red.
