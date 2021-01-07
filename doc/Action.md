# Action

Action是Stackstorm里面用于执行具体自动化动作的模块，据说可以用任何语言编写，这里我们以python为例。Action可以与[Rule](https://docs.stackstorm.com/rules.html)结合使用，当满足一定的规则时触发，多个Action可以由workflow编排在一起。Action可以通过CLI，API，UI来执行。

## Python脚本

Action脚本包含一个python文件和Metadata元数据文件两部分。

本例中我们的python脚本调用NASA的天文学图片库API，查询指定日期的图片

> 查看或编辑 `/opt/stackstorm/packs/tutorial/actions/nasa_apod.py`

```python
import json
import requests
from st2common.runners.base_action import Action

API_URL = "https://api.nasa.gov/planetary/apod"


class Apod(Action):

    def run(self, api_key, date, hd):
        params = {'api_key': api_key,
                  'hd': hd}
        if date is not None:
            params['date'] = date

        response = requests.get(API_URL, params=params)
        response.raise_for_status()
        data = response.json()
        if hd:
            data['url'] = data['hdurl']
        return data
```

原生python的Action脚本需要遵从以下规则：

- python脚本***必须***只能包含一个 `class` 并且继承自 `st2common.runners.base_action.Action`
- `Action` 子类***必须***定义 `def run(self)` 方法.
举个简单的例子
```python
from st2common.runners.base_action import Action

class HelloWorld(Action):
    def run(self):
        return "hello world"
```



## 元数据脚本

首先我们要创建一个Action Metadata脚本，脚本中的 `runner_type: python-script`，这样将会告诉StackStorm我们要使用原生python action，另外还需要定义action的入参


> 查看或编辑 `/opt/stackstorm/packs/tutorial/actions/nasa_apod.yaml`

```yaml
---
name: nasa_apod
pack: tutorial
description: "Queries NASA's APOD (Astronomy Picture Of the Day) API to get the link to the picture of the day."
runner_type: "python-script"
enabled: true
entry_point: nasa_apod.py
parameters:
  api_key:
    type: string
    description: "API key to use for api.nasa.gov."
    default: "DEMO_KEY"
  hd:
    type: boolean
    description: "Retrieve the high resolution image."
    default: false
  date:
    type: string
    description: "The date [YYYY-MM-DD] of the APOD image to retrieve."
```

从这里可以看到，元数据文件中定义的入参与python文件中`run`方法的入参是一致的，在元数据中可以定义入参的默认值和描述



接下来需要注册action

```
st2ctl reload --register-actions
```

测试

```
st2 run tutorial.nasa_apod date=2021-01-01
```

```
root@185f6453e40a:/opt/stackstorm/packs/tutorial/rules# st2 run tutorial.nasa_apod date=2021-01-01
...
id: 5ff665709f87412a6014b02b
action.ref: tutorial.nasa_apod
context.user: st2admin
parameters: 
  date: '2021-01-01'
status: succeeded
start_timestamp: Thu, 07 Jan 2021 01:35:44 UTC
end_timestamp: Thu, 07 Jan 2021 01:35:50 UTC
result: 
  exit_code: 0
  result:
    copyright: Petr Horalek
    date: '2021-01-01'
    explanation: The South Celestial Pole is easy to spot in star trail images of the southern sky. The extension of Earth's axis of rotation to the south, it's at the center of all the southern star trail arcs. In this starry panorama streching about 60 degrees across deep southern skies the South Celestial Pole is somewhere near the middle though, flanked by bright galaxies and southern celestial gems. Across the top of the frame are the stars and nebulae along the plane of our own Milky Way Galaxy. Gamma Crucis, a yellowish giant star heads the Southern Cross near top center, with the dark expanse of the Coalsack nebula tucked under the cross arm on the left. Eta Carinae and the reddish glow of the Great Carina Nebula shine along the galactic plane near the right edge. At the bottom are the Large and Small Magellanic clouds, external galaxies in their own right and satellites of the mighty Milky Way. A line from Gamma Crucis through the blue star at the bottom of the southern cross, Alpha Crucis, points toward the South Celestial Pole, but where exactly is it? Just look for south pole star Sigma Octantis. Analog to Polaris the north pole star, Sigma Octantis is little over one degree fom the the South Celestial pole.
    hdurl: https://apod.nasa.gov/apod/image/2101/2020_12_16_Kujal_Jizni_Pol_1500px-3.png
    media_type: image
    service_version: v1
    title: Galaxies and the South Celestial Pole
    url: https://apod.nasa.gov/apod/image/2101/2020_12_16_Kujal_Jizni_Pol_1500px-3.jpg
  stderr: ''
  stdout: ''

```

我们也可以增加 `--debug` 标记来查看内容API的调用情况:

```
st2 --debug run tutorial.nasa_apod date=2018-07-04
```

关注每一个cURL的输出情况，可以帮我们更好的理解CLI的行为