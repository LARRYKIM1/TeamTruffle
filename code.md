## 


# 1.자료수집 및 가공 
## 물고기 사진 구글에서 크롤링
- pip install icrawler
- 아래는 큰입베스 (학명: Micropterus salmoides) 크롤링 예시 
- 크롤링 방지 우회위해 적절한 이미지 개수 (max_num) 설정 필요
```python
from icrawler.builtin import GoogleImageCrawler
google_crawler = GoogleImageCrawler(parser_threads = 2, downloader_threads = 4,
storage = {"root_dir" : "../datas/큰입베스"})
google_crawler.crawl(keyword = "Micropterus salmoides", max_num = 1000,
min_size = (200, 200), max_size = None)
```

## 크롤링된 물고기 폴더, 이름 변경하여 합치기(1부터 순차적 이름 할당)
- 
``` python
크롤링된 물고기 폴더, 이름 변경하여 합치기(1부터 순차적 이름 할당)
import os

fishListDir =["갈치","강성돔","고등어","광어","농어","돌돔","방어","볼락","붕장어","가자미"]
numList=["",1,2,3,4] # 빈칸은 갈치, 1은 갈치1

idx = 0
for i in range(len(fishListDir)): #10번
    #한 물고기의 폴더들이 끝나면 idx 초기화
    idx = 0
    for j in range(len(numList)): #5번
        input_path_1 = ".\{0}".format(fishListDir[i]+str(numList[j]))

        #가자미 폴더에서 가자미1이 아닌 가자미2로 증가를 지원(폴더명 특수성)
        if not os.path.isdir(input_path_1):
            continue

        #디렉토리내 파일리스트 
        file_list_input_1 = os.listdir(input_path_1)

        #파일들 순차접근, l변수에 담기
        for l in file_list_input_1:
            idx += 1
            os.rename(r'.\{0}\{1}'.format(fishListDir[i]+str(numList[j]),l),r'.\{0}\{1}.jpg'.format(fishListDir[i],idx))
        if str(numList[j]) != "": #갈치 빼고 갈치1, 갈치2 등 지우기 
            os.rmdir(r'.\{0}'.format(fishListDir[i]+str(numList[j])))
```

## 갈치, 감성돔, 고등어 라벨링

```python

갈치, 감성돔, 고등어 라벨링

import os
from pandas import Series, DataFrame
import openpyxl
import pandas as pd
import numpy as np

list=[]


path_dir = 'C:/Users/Administrator/Documents/빅청/데이터정제3 (4)/collectedFishes'
file_list = os.listdir(path_dir)

for i in range(len(file_list)):
    list.append("{}.jpg".format(i+1))

frame = DataFrame(list, columns=['name'])
sLength = len(frame['name'])
frame.loc[:,2] =np.zeros_like(sLength,dtype="i")
frame.loc[:,3] =np.zeros_like(sLength,dtype="i")
frame.loc[:,4] =np.zeros_like(sLength,dtype="i")
frame.loc[:,5] =np.zeros_like(sLength,dtype="i")


frame.iloc[0:386, 1:2]=0
frame.iloc[386:1140, 1:2]=1
frame.iloc[1140:1409, 1:2]=2

frame.iloc[0:386, 2:3]=1
frame.iloc[386:1140, 3:4]=1
frame.iloc[1140:1409, 4:5]=1

frame.columns=['name','구분','갈치','감성돔','고등어']
#if(frame.name.loc[1]==True):
 #   frame.name.loc=1

print(frame)

frame.to_csv('data.csv',encoding = 'cp949',header=True,mode='w')

```

# 2.전처리

- 설명: json파일에서 정확도(confidence) 50퍼센트 이상만 이미지 crop
- 매개변수
    1. param_json_files_path: json과 jpg 파일들이 있는 위치 정보(ex. "F:/TrufflePig/out")
    2. param_label_name: 원하는 label 값(ex. fish, person, bird 등)

```python
import os
 import json
 import cv2

 def searchByLableName(param_json_files_path, param_label_name):

     def im_trim(img, t_x, t_y, b_x, b_y): #함수로 만든다
         #topleft y:bottomright y,topleft x: bottomright x
         if img is None:
             return
         img_trim=img[t_y:b_y, t_x:b_x]
         return img_trim #필요에 따라 결과물을 리턴

     json_jpg_files_path = param_json_files_path
     file_list = os.listdir(json_jpg_files_path)
     file_list_json = [file for file in file_list if file.endswith(".json")]

     searched_list=[]
     for i in range(len(file_list_json)):
         json_name_with_path=json_jpg_files_path+'/'+file_list_json[i]
         # print(json_name_with_path)
         with open(json_name_with_path) as json_file:
             # print("")
             json_data = json.load(json_file)
             # print(json_data)

             # ★★★★★★★★  param_label_name
             # test=[ item for item in json_data if float(item['confidence']) >= 0.5]
             # print(test)
             search_label_dict = (item for item in json_data if item['label'] == param_label_name and float(item['confidence']) >= 0.5)
             # search_label_dict = (item for item in json_data if item['label'] == param_label_name)
             # next_search_label_dict = next(search_label_dict, False)
             next_search_label_dict = next(search_label_dict, False)
             if next_search_label_dict is False:
                 continue
             #JPG 이미지명 튜플 추가
             image_jpg_name=file_list_json[i].split('.')
             next_search_label_dict["imagename"]=image_jpg_name[0]+".jpg"
             searched_list.append(next_search_label_dict)
             # print(next_search_label_dict)
     # print(searched_list)


     # ★★★★★★★★  param_label_name
     save_path=json_jpg_files_path+"/"+param_label_name
     print(json_jpg_files_path+"/"+param_label_name)
     if not os.path.isdir(save_path):
     	os.mkdir(save_path)

     for l in range(len(searched_list)):
         image_name_to_be_cropped=searched_list[l]["imagename"]
         #x,y,w,h ->
         #   topleft의 x, topleft의 y, bottomright의 x - topleft의 x, bottomright의 x - topleft의 y
         image_name_to_be_cropped_t_x=searched_list[l]["topleft"]["x"]
         image_name_to_be_cropped_t_y=searched_list[l]["topleft"]["y"]
         image_name_to_be_cropped_b_x=searched_list[l]["bottomright"]["x"]
         image_name_to_be_cropped_b_y=searched_list[l]["bottomright"]["y"]

         # print(image_name_to_be_cropped_x)
         # print(image_name_to_be_cropped_y)
         # print(image_name_to_be_cropped_w)
         # print(image_name_to_be_cropped_h)
         print(json_jpg_files_path+"/"+image_name_to_be_cropped)

         org_image = cv2.imread(json_jpg_files_path+"/"+image_name_to_be_cropped)

         trim_image = im_trim(org_image,image_name_to_be_cropped_t_x, image_name_to_be_cropped_t_y, image_name_to_be_cropped_b_x, image_name_to_be_cropped_b_y)
         # ★★★★★★★★ param_label_name
         cv2.imwrite("{0}/{1}{2}.jpg".format(save_path, param_label_name, l), trim_image) #이미지 저장


 searchByLableName("F:/TrufflePig/out", "bird")
```

# 3.관련 상세정보 수집 

# 4.딥러닝(이미지분류)

# 5.앱
- 설명: 사진의 메타데이터 (위도, 경도, 주소, 날짜, 맵HTML) 가져오기
- 매개변수
  - 메타데이터를 불러올 jpg가 저장된 위치(e.g. F:/TrufflePig/WebDev/img/)
  - 사진 이름 (e.g. 확장자 붙이기 말기)
```python
from PIL import Image
 import matplotlib.pyplot as plt
 from PIL.ExifTags import TAGS
 import folium
 from bs4 import BeautifulSoup
 import geocoder
 from collections import OrderedDict
 import json


 def GetJsonFishInfo(path,saveFileName):
     image = Image.open(path+saveFileName+".jpg")

     metaDict={}
     info=image._getexif()
     for tag, value in info.items():
         decoded = TAGS.get(tag, tag)
         metaDict[decoded] = value
     # print(metaDict)
     # for key in metaDict.keys():
     #     print(metaDict[key])
     #
     lat_s = ((metaDict["GPSInfo"])[2])[2][0] / ((metaDict["GPSInfo"])[2])[2][1]
     lat_m = (((metaDict["GPSInfo"])[2])[1])[0]
     lat_h = (((metaDict["GPSInfo"])[2])[0])[0]
     s = round((lat_s/60),3)
     s_m = s + lat_m
     m = round((s_m/60),5)
     lat = lat_h + m
     # 위도의 도/분/초 단위를 도 단위로 변환
     lon_s = ((metaDict["GPSInfo"])[4])[2][0] / ((metaDict["GPSInfo"])[4])[2][1]
     lon_m = (((metaDict["GPSInfo"])[4])[1])[0]
     lon_h = (((metaDict["GPSInfo"])[4])[0])[0]
     s = round((lon_s/60),3)
     s_m = s + lon_m
     m = round((s_m/60),5)
     lon = lon_h + m

     # print(lat,lon)

     point_map = folium.Map(location = [lat, lon], zoom_start = 30)
     folium.Marker([lat, lon], popup = "map_Catch Point").add_to(point_map)
     point_map.save(path+"map.html")


     soup = BeautifulSoup(open(path+"map.html"), "html.parser")
     filtered_soup = str(soup).replace("\n","")

     g = geocoder.osm([lat, lon], method="reverse")
     addr=g.address
     print(addr)

     fish_date = metaDict["DateTime"]

     pic_metadata = OrderedDict()

     # 데이터 저장
     pic_metadata["lat"] = lat
     pic_metadata["lon"] = lon
     pic_metadata["addr"] = addr
     pic_metadata["date"] = fish_date
     pic_metadata["point_map"] = filtered_soup


     return pic_metadata

 print(GetJsonFishInfo("F:/TrufflePig/WebDev/img/","test4"))
 ```


