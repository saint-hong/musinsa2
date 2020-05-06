# Musinsa Web-Site Crawler

## 1. 개요
무신사 웹 사이트 크롤러는 크롤링 프로젝트의 목적으로 만든 Scrapy를 활용한 웹 크롤러 입니다. 무신사 웹사이트 크롤러는 랭킹 페이지에 등록 된 상품의 브랜드명, 카테고리, 상품명, 상품번호, 상품의 성별, 판매가, 세일가, 좋아요 개수, 리뷰 개수, 주요 구매고객, 링크 데이터를 수집하고, 수집한 데이터를 데이터 베이스와 CSV 파일로 저장, 슬랙 메신저에 메시지를 보낼 수 있습니다. 

랭킹 페이지의 설정 조건을 다음과 같이 변경할 수 있습니다. 

  - 랭킹의 주기 : 실시간, 일간, 주간, 월간, 연간 
  - 랭킹의 카테고리 : 상의, 아우터, 원피스, 바지, 스커트, 가방, 신발 등

또한 다음과 같은 기능을 사용합니다. 

```
import requests
import scrapy
import requests
import re
from scrapy.http import TextResponse
```

무신사 웹 사이트 크롤러를 사용하면, 랭킹에 오른 상품의 특징, 고객 층별 상품의 특징, 기획 상품의 특징 등을 파악할 수 있는 자료를 수집할 수 있습니다.

## 2. 사용 방법

무신사 웹 사이트 크롤러는 Scrapy의 작동 방식으로 구현 됩니다.

### 2-1. Scrapy Startproject 파일 생성
```
!scrapy startproject musinsa
```

### 2-2. Items.py 생성
  - 필요한 데이터에 맞게 아래 양식대로 추가 하면 됩니다. 
  - (ex) 상품의 누적 판매 수 : total_sales = scrapy.Field()
  
```
%%writefile musinsa/musinsa/items.py
import scrapy


class MusinsaItem(scrapy.Item):
    ranking = scrapy.Field()
    brand_name = scrapy.Field()
    product_name = scrapy.Field()
    product_num = scrapy.Field()
    product_spec = scrapy.Field()
    gender = scrapy.Field()
    origin_price = scrapy.Field()
    sale_price = scrapy.Field()
    good_num = scrapy.Field()
    review_count = scrapy.Field()
    target_name = scrapy.Field()
    link = scrapy.Field()
```









