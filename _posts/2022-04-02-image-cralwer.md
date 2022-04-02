---
layout: post
title: 파이썬과 selenium을 이용해 구글에서 이미지 다운로드 자동화하기
date: 2022-04-02 04:08:00 +/- 0000
categories: [python, crawling]
tags: [python, selenium]     
author: entrkjm
---


# 파이썬의 Selenium을 이용해 구글에서 이미지 다운로드를 자동화하기

  

## 이 프로그램을 만들게 된 계기

회사에서 브랜드 3000개에 대한 로고 이미지를 다운로드할 일이 있었습니다. 이 일을 전부 제 손으로 하기에는 너무나 무리라는 생각이 들었고, 구글에 검색해서 이미지를 다운로드할 수 있도록 자동화된 프로그램을(구글링을 통해) Selenium으로 만들었습니다.

참고에는 다음 사이트를 활용하였습니다.

*자세한 내용은 [**여기**](https://goodthings4me.tistory.com/535)를 참조*

## 작동 방식
이 프로그램은 아래와 같은 순서로 작동합니다.

1)검색하고자 하는 키워드의 리스트를 입력 받습니다. 

2)입력받은 키워드를 구글에 이미지로 검색합니다.

3)가장 상단(맨 위 가장 왼쪽)에 노출된 이미지를 클릭하고, 클릭시 PC 기준으로 오른쪽에 뜨는 이미지 중 '가장 큰 이미지'를 다운로드 받습니다. 

4)그 다음은 상단에서 그 다음으로 노출된 이미지(맨 위 왼쪽 두번째 이미지)를 클릭하고, 마찬가지로 '가장 큰 이미지'를 다운로드합니다. 키워드 검색시 노출되는 순서에 따라서, 이를 반복하는데요. 이미지를 원하는 숫자만큼 다운로드 받을 수 있습니다.

5)다운로드 받은 이미지를, 키워드 이름으로 만든 폴더에 저장합니다.

예를 들면 '나루토'를 이미지로 검색하고, 검색 결과에서 가장 왼쪽의 이미지를 클릭하면 오른쪽에 검은 배경으로 큰 이미지가 뜹니다. 이 큰 이미지를 다운로드 합니다. 그 다음 검색 결과에서 왼쪽의 이미지를 클릭하고 반복합니다.

![나루토 검색 결과](_posts\image-crawler\naruto1.PNG)
![나루토 검색 결과](_posts\image-crawler\naruto1.PNG)

소스 코드는 아래와 같습니다. 셀레니움을 설치하고 chromedriver를 다운로드 받는 법에 대해서는 생략하였습니다.


```python
from selenium.webdriver.common.keys import Keys
from selenium import webdriver
from urllib import request
import os
import time
import pandas as pd

#폴더를 만들기 위한 메서드
def createFolder(directory):
    try:
        if not os.path.exists(directory):
            os.makedirs(directory)
    except OSError:
        print ('Error: Creating directory. ' +  directory)


#이미지 다운로드를 위한 메서드
def image_downloader(name_list): #name_list: 검색하고자 하는 키워드 목록을 담은 리스트
    for i in name_list:
        
        name = str(i)
        
        createFolder(name)

		#검색 결과가 나오지 않을 수 있기 때문에, try-error를 활용함
        #검색
        try: 
            elem = driver.find_element_by_name('q')  # class='gLFyf gsfi'
            elem.clear()
            elem.send_keys(name)
            elem.send_keys(Keys.RETURN)
            time.sleep(1)

			#검색 결과로 얻은 이미지의 수가 10개가 넘으면 10개만, 10개 미만이면 가능한만큼 다운로드를 시도함
			if len(driver.find_elements_by_css_selector('img.rg_i.Q4LuWd')) > 10: 
                num = 10

            else:
                num = len(driver.find_elements_by_css_selector('img.rg_i.Q4LuWd'))
                
        except:
            continue
            
        for i in range(num):
            try: 
                driver.find_elements_by_css_selector('img.rg_i.Q4LuWd')[i].click() #검색 결과로 나온 이미지를 순서대로 클릭
                time.sleep(1)
                big_image = driver.find_element_by_css_selector('img.n3VNCb')  
                bigImage_url = big_image.get_attribute('src')
                request.urlretrieve(bigImage_url, '%s/%s'%(name, name) + str(i+1) + ".jpg") #이미지를 다운로드해서 폴더에 저장
            except:
                pass
            
    driver.close()
```

나루토의 겸색으로 2개의 이미지를 다운로드 받은 결과는 다음과 같습니다.


```
#Result

'县公安局' '交巡警' '大队' '、' '治安' '大队' '、' '国保' '大队' '、' '辖区' '派出所' '及县' '消防' '救援' '大队' '相关' '负责人' '陪同' '检查' '。' 

'县公安局' '公安' '公安局' '交巡警' '巡警' '大队' '、' '治安' '大队' '、' '国' '保' '大队' '、' '辖区' '派出' '派出所' '所及' '县' '消防' '救援' '大队' '相关' '负责' '负责人' '责人' '陪同' '检查' '。'
```

jieba_cut의 경우 반환형이 'generator'이기 때문에, list를 사용하는 것이 편할 수 있습니다. 그럴 때는 jieba_lcut을 사용하면 됩니다.

jieba_cut 외에도 jieba에는 각기 다른 알고리즘을 활용한 tokenizer나 text-extract method가 있는데요. pseg.cut의 경우 jieba_cut과 마찬가지로 동작하지만, 반환형이 (단어, 품사)의 꼴인 tuple을 원소로 갖는 'list'여서, 단어만 추출하려면 추가적인 작업이 필요합니다.

여기서는 jieba.analyse.extract_tags를 한 번 사용하겠습니다. 이 method는 TF-IDF 알고리즘을 사용하는데요. 이 알고리즘은 전체 문서에서 너무 많이 등장하는 단어는 별 의미가 없다고 간주하여 가중치를 낮추는 방식입니다. 예를 들어 영어의 'the' 같은 단어는 TF-IDF 알고리즘 스코어가 낮게 되겠습니다. 

그런데 TF-IDF를 제대로 사용하려면, 분석하고자 하는 문서의 각 키워드에 대해서 TF-IDF 스코어를 먼저 구해야하는데요. **jieba에서는 사전에 구현된 기본 dictionary와 IDF score가 준비된 상태입니다.** 여기서  jieba.analyse.set_ idf_ path(file_ name) method를 활용하면, 제가 따로 만든 custom corpus에 대해서 IDF 스코어를 구하고, 그것을 바탕으로 TF-IDF를 계산할 수 있습니다. 마찬가지로 [**링크1**](https://developpaper.com/detailed-use-in-chinese-word-segmentation-based-on-jieba-package-in-python/), [**링크2**](https://pythonmana.com/2021/12/202112130117064371.html)를 참조하시면 되겠습니다.

지금은 Tutorial이기 때문에 jieba에서 이미 사전에 준비된 기본 값들을 사용해보도록 하겠습니다.
```python
words = jieba.analyse.extract_tags(st, topK = 10) # 상위 10개를 추출
words
```
```
#result 
['大队', '国保', '及县', '交巡警', '县公安局', '消防', '陪同', '治安', '辖区', '派出所']
```

---

## Custom-dictionary 활용하기: add_word
앞서 jieba에서는 사전에 준비된 dictionary와 사전에 학습된 문서 데이터를 바탕으로 한 parameter 값을 갖고 있다고 말씀드렸습니다.

여기에 제가 원하는 다른 단어를 추가-제거하는 것도 가능합니다.

이를테면, 위의 예시를 활용한 아래의 코드에서는 '县公安局' 이 단어가 추가됐다가 빠진 것을 확인하실 수 있습니다.
```python
st = '县公安局交巡警大队、治安大队、国保大队、辖区派出所及县消防救援大队相关负责人陪同检查。'
jieba.del_word('县公安局')

cut1 = jieba.lcut(st, cut_all = False)
print(cut1)

jieba.add_word('县公安局')
#jieba.add_word(word, freq=None, tag=None)

cut2 = jieba.lcut(st, cut_all = False)
print(cut2)
```
```
#result
['县', '公安局', '交巡警', '大队', '、', '治安', '大队', '、', '国保', '大队', '、', '辖区', '派出所', '及县', '消防', '救援', '大队', '相关', '负责人', '陪同', '检查', '。'] 
['县公安局', '交巡警', '大队', '、', '治安', '大队', '、', '国保', '大队', '、', '辖区', '派出所', '及县', '消防', '救援', '大队', '相关', '负责人', '陪同', '检查', '。']
```

아예 텍스트 파일로부터 User_custom_dictionary를 load하는 것도 가능한데요. 

```python
jieba.load_userdict(file_path) #file_path에 txt파일 입력
```
이때 텍스트 파일의 형식은 다음과 같아야 합니다
```
단어(띄어쓰기)빈도(띄어쓰기)품사(\n)
단어(띄어쓰기)....
```
이 때 빈도나 품사는 꼭 필수가 아니며, 빈도의 경우 예측기의 성능에 영향을 미치는 관계로 suggest_freq()라는 method를 이용해 구한 freq를 넣어주는 것도 가능합니다.