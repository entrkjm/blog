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

**jieba는 그 중에서 그나마 영어로 된 문서가 있는, 중국어 tokenizer입니다.** tokenizer에 대해 간단하게 설명하자면, 우리가 쓰는 단어나 문장을 '의미' 단위로 나눠서 컴퓨터에 입력할 수 있는 형태로 만드는 프로그램이라고 보시면 됩니다.

jieba에서는 미리 여러 기능이 갖춰져 있기 때문에, 이를 활용해서 중국어 문서를 분석하는 프로그램을 만들었다. 그 전에 간단한 사용법부터 보겠습니다.

*자세한 내용은 [**여기**](https://developpaper.com/detailed-use-in-chinese-word-segmentation-based-on-jieba-package-in-python/)를 참조*

## jieba.cut과 text
jieba.cut(sentence, cut_all = False)는 sentence를 input parameter로 넣으면 jieba tokenizer를 활용해 문장을 단어로 나누어주는 method입니다.

```python
import jieba
import jieba.analyse
import jieba.posseg as pseg

st = '县公安局交巡警大队、治安大队、国保大队, 辖区派出所及县消防救援大队相关负责人陪同检查。'

#return type은 generator
cut_1 = jieba.cut(st, cut_all = True)
cut_2 = jieba.cut(st, cut_all = False)

for i in cut_1:
	print('\'%s\''%i, end=' ')

print('\n')

for j in cut_1:
	print('\'%s\''%j, end=' ')
```
결과는 다음과 같습니다. 약간의 차이가 있는게 보이시나요? cut_all을 True로 바꿔주면, tokenizer가 빡세게(?) 쪼갭니다.
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