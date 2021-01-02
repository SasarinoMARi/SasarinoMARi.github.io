---
layout: post
title:  "Google Takeout으로 백업한 Google Photo 이미지와 JSON 메타데이터 합치기"
date:   2021-01-02 16:50:00 +0900
categories: [python]
---

# 여는 글

올해 6월부터 `구글 포토의 무제한 사진 저장 정책이 변경`됩니다.
2020년 6월 21일부로 사진 크기에 관계없이 15GB 까지만 무료로 저장 가능하고, 그 이상 사용하려면 유료 요금제가 필요한 모양입니다.

구글 포토는 훌륭한 플랫폼이고 돈을 낼 가치도 있다고 생각하지만,
저는 이미 OneDrive 유료 플랜을 사용하고 있으므로 이번엔 이쪽으로 마이그레이션을 해보고자 합니다.


### 구글 테이크아웃?

구글 테이크아웃 서비스를 이용하면 Google 계정에서 연락처나 캘린더 등등 여러 사용자 데이터를 백업해서 파일로 받을 수 있습니다.

테이크아웃을 하는 과정은 별도의 글에 정리해두겠습니다.


### 그래서 문제가 뭐라고?

구글 포토는 업로드 한 사진을 그대로 저장하기 때문에 사진을 업로드 할 때 메타데이터가 정상적이면 아무 문제 없습니다.

하지만 카카오톡으로 받은 사진이나, 일부 뷰티 카메라나 사진 편집 어플 등을 사용해 메타데이터가 손상된 파일의 경우 이를 그대로 OneDrive에 올릴 경우 `파일을 만든 날짜로 정렬을 해버리기 때문에` 갤러리를 볼 때 굉장히 거슬릴 수 있습니다.

오늘은 python을 사용해 간단하게 모든 사진 파일의 메타데이터를 검사하고, 누락된 파일이 있을 경우 동봉된 json 파일의 메타데이터를 덮어씌우는 작업을 해보겠습니다.

사용된 모듈은 PIL(Python Image Library)와 Exif 태그 편집에 사용한 piexif입니다.

----------------

# 1. JSON 파싱

구글 포토 테이크아웃에 딸려오는 json 파일은 아래와 같은 형식으로 구성됩니다.

```json
{
  "title": "2016-12-22-오전-07-59-17.jpg",
  "description": "",
  "imageViews": "0",
  "creationTime": {
    "timestamp": "1575995678",
    "formatted": "2019. 12. 10. 오후 4시 34분 38초 UTC"
  },
  "modificationTime": {
    "timestamp": "1607291330",
    "formatted": "2020. 12. 6. 오후 9시 48분 50초 UTC"
  },
  "photoTakenTime": {
    "timestamp": "1482361157",
    "formatted": "2016. 12. 21. 오후 10시 59분 17초 UTC"
  },
  "geoData": {
    "latitude": 35.11843611111111,
    "longitude": 129.049575,
    "altitude": 8.754129606099111,
    "latitudeSpan": 0.0,
    "longitudeSpan": 0.0
  },
  "geoDataExif": {
    "latitude": 35.11843611111111,
    "longitude": 129.049575,
    "altitude": 8.754129606099111,
    "latitudeSpan": 0.0,
    "longitudeSpan": 0.0
  },
  "googlePhotosOrigin": {
    "webUpload": {
      "computerUpload": {
      }
    }
  }
}
```

단순 정렬만 원한다면 타임스탬프만 사용해도 되겠네요. 하지만 모처럼이니 위치 데이터도 같이 가공해 보겠습니다.

```python
import os
import json

def stuff(imagePath):
    jsonPath = f'{imagePath}.json'
    if not os.path.isfile(jsonPath): return # 해당하는 json 파일이 없으면 생략합니다.

    with open(jsonPath, 'r', encoding='UTF8') as file:
        jsonStr = file.read().replace('\n', '')
        jsonData = json.loads(jsonStr)
        print(jsonData)
```

사진 경로를 받아서 파일명 뒤에 .json만 추가해 메타데이터 파일을 읽어옵니다.

python 내장 json 모듈을 사용해서 간단하게 읽을 수 있습니다. 편리하네요!

# 2. 사진을 찍은 날짜 등록하기

```python
from PIL import Image
from PIL.ExifTags import GPSTAGS
import piexif

from datetime import datetime

def stuff(imagePath):
    # 이미지와 EXIF 태그 읽어오는 코드
    try: 
        image = Image.open(imagePath)
        exif_dict = piexif.load(image.info['exif'])
    except: return

    # 찍은 날짜가 이미 설정되어있는지 확인합니다.
    dateTime = None
    try:
        dateTime = exif_dict['0th'][piexif.ImageIFD.DateTime]
        dateTime = exif_dict['Exif'][piexif.ExifIFD.DateTimeOriginal]
        dateTime = exif_dict['Exif'][piexif.ExifIFD.DateTimeDigitized]
    except KeyError:
        pass

    # 찍은 날짜가 설정되어있지 않다면 json에서 적당한 값을 찾아 등록합니다.
    if dateTime is None:
        newdate_timestamp = None
        if 'photoTakenTime' in jsonData:
            newdate_timestamp = jsonData['photoTakenTime']['timestamp']
            print('dateTime을 photoTakenTime으로 설정했습니다.')
        elif 'creationTime' in jsonData:
            newdate_timestamp = jsonData['creationTime']['timestamp']
            print('dateTime을 creationTime으로 설정했습니다.')
        elif 'modificationTime' in jsonData:
            newdate_timestamp = jsonData['modificationTime']['timestamp']
            print('dateTime을 modificationTime으로 설정했습니다.')
        else:
            print('유효한 Timestamp를 찾지 못했습니다. dateTime 초기화를 생략합니다.')

        if not newdate_timestamp is None: 
            # Unix Timestamp 형식을 아래와 같은 형식의 문자열로 가공하여 넣어야 합니다.
            new_date = datetime.fromtimestamp(int(newdate_timestamp)).strftime("%Y:%m:%d %H:%M:%S")

            print(f'dateTime을 {new_date}로 설정')
            exif_dict['0th'][piexif.ImageIFD.DateTime] = new_date
            exif_dict['Exif'][piexif.ExifIFD.DateTimeOriginal] = new_date
            exif_dict['Exif'][piexif.ExifIFD.DateTimeDigitized] = new_date
            
            # 수정한 파일을 저장합니다.
            exif_bytes = piexif.dump(exif_dict)
            dirname = f'{os.path.dirname(imagePath)}/modified'
            if not os.path.isdir(dirname): os.mkdir(dirname)
            output_fname = os.path.join(dirname, os.path.basename(imagePath))
            image.save(output_fname, exif=exif_bytes)
            print(f"수정된 파일을 저장했습니다: {output_fname}")
```

PIL 모듈로 이미지를 로드하고 exif 데이터를 가져와 piexif 모듈을 초기화합니다.

먼저 기등록된 날짜 정보가 있는지 확인하고, 등록된 날짜가 없다면 세 개의 timestamp를 순차적으로 확인하고 이를 사용해 찍은 날짜 정보를 설정합니다.

exif에 날짜를 기입할 때는 년:월:일 시:분:초 형식으로 포맷을 맞춰 줘야 합니다.
타임스탬프를 그대로 넣었더니 파일에 반영되어 나오지 않아서 뭐가 문젠지 한참을 헤맸네요.

마지막으로 piexif 모듈의 dump 함수를 써서 얻은 바이트 배열을 PIL 라이브러리를 통해 이미지에 씌워 저장합니다.

# 3. 사진의 위치 데이터 등록하기

위치 데이터의 경우는 조금 복잡합니다.
EXIF 태그에 사용되는 좌표계는 DMS 좌표계인데 테이크아웃으로 받을 수 있는 좌표계는 Degree 좌표계라 이를 변환하는 과정이 필요합니다.

```python
from fractions import Fraction

def to_deg(value, loc):
    """convert decimal coordinates into degrees, munutes and seconds tuple
    Keyword arguments: value is float gps-value, loc is direction list ["S", "N"] or ["W", "E"]
    return: tuple like (25, 13, 48.343 ,'N')
    """
    if value < 0:
        loc_value = loc[0]
    elif value > 0:
        loc_value = loc[1]
    else:
        loc_value = ""
    abs_value = abs(value)
    deg =  int(abs_value)
    t1 = (abs_value-deg)*60
    min = int(t1)
    sec = round((t1 - min)* 60, 5)
    return (deg, min, sec, loc_value)

def change_to_rational(number):
    """convert a number to rantional
    Keyword arguments: number
    return: tuple like (1, 2), (numerator, denominator)
    """
    f = Fraction(str(number))
    return (f.numerator, f.denominator)

def stuff():
    # json 파일에서 위치 데이터를 불러옵니다.
    latitude = float(jsonData['geoData']['latitude'])
    longitude = float(jsonData['geoData']['longitude'])
    altitude = float(jsonData['geoData']['altitude'])

    if not latitude == "0.0" and longitude == "0.0" and altitude == "0.0":
        # Degree 좌표계를 DMS 좌표계로 변환합니다.
        lat_deg = to_deg(latitude, ["S", "N"])
        lng_deg = to_deg(longitude, ["W", "E"])
        exiv_lat = (change_to_rational(lat_deg[0]), change_to_rational(lat_deg[1]), change_to_rational(lat_deg[2]))
        exiv_lng = (change_to_rational(lng_deg[0]), change_to_rational(lng_deg[1]), change_to_rational(lng_deg[2]))
```

json에서 위도, 경도, 고도 좌표를 가져오고 to_deg 함수를 사용해 변환하는 코드입니다.
고도 좌표는 기본적으로 변환할 필요가 없습니다.

change_to_rational 함수에서 사용하는 Fraction 모듈은 python에서 실수를 다루는 모듈입니다.
분수처럼 분자와 분모로 수를 표시하므로서 조금 더 정밀한 수의 표현이 가능해집니다.

```python
from PIL.ExifTags import GPSTAGS
import piexif

def stuff():
    # 등록된 위치 데이터가 있는지 확인하고 없다면 변환한 좌표를 등록합니다.
    try:
        exif_dict['GPS'][piexif.GPSIFD.GPSLatitude]
        exif_dict['GPS'][piexif.GPSIFD.GPSLongitude]
        exif_dict['GPS'][piexif.GPSIFD.GPSAltitude]
    except KeyError:
            gps_ifd = {
                piexif.GPSIFD.GPSVersionID: (2, 0, 0, 0),
                piexif.GPSIFD.GPSAltitudeRef: 1,
                piexif.GPSIFD.GPSAltitude: change_to_rational(round(altitude)),
                piexif.GPSIFD.GPSLatitudeRef: lat_deg[3],
                piexif.GPSIFD.GPSLatitude: exiv_lat,
                piexif.GPSIFD.GPSLongitudeRef: lng_deg[3],
                piexif.GPSIFD.GPSLongitude: exiv_lng,
            }
            exif_dict['GPS'] = gps_ifd
            print(f'위치 값을 설정합니다: {gps_ifd}')
```

기존 등록된 위치 정보가 없다면 변환한 위치 정보를 등록하는 코드입니다.

# 마치며

![누락되었던 메타데이터가 입혀진 화면](https://raw.githubusercontent.com/sasarinomari/sasarinomari.github.io/master/static/img/_posts/20210102001.jpg)

왼쪽 원본 사진에는 없던 찍은 날짜가 프로그램 실행 후 추가된 모습입니다.

전체 코드는 [gist](https://gist.github.com/SasarinoMARi/ac6524dd272c8554415bafaf36fbd269)에서 볼 수 있습니다.

EXIF를 가볍게나마 다뤄볼 수 있는 좋은 경험이었네요.


