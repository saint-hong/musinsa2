# Musinsa Web-Site Crawler

# 목차
- 1. 개요
- 2. 사용방법
- 3. 수집한 데이터 확인
- 4. 주의 

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
  - 1). Startproject 파일 생성
  - 2). Items.py 파일 생성
  - 3). Spider 파일 생성
  - 4). CSV 파일 저장
  - 5). 데이터 베이스 저장 (Mongo
  - 6). Pipeline 생성

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

### 2-3. Spider 설정
  - 기본 URL은 주간 랭킹페이지로 설정되어 있으며, 원하는 상품 카테고리 값을 변경할 수 있습니다.
  - 주간 랭킹 페이지의 1페이지 부터 100페이지 까지 크롤링 할 수 있습니다. 
  - 필요시 페이지 수를 조절 할 수 있습니다. (ex) last_page = int(last_page / 20) --->> last_page = 5
  
```
%%writefile musinsa/musinsa/spiders/spider.py
import requests
import scrapy
import re
import time

from scrapy.http import TextResponse
from musinsa.items import MusinsaItem

class Spider(scrapy.Spider):
    
    name = "MusinsaRanking"
    allow_domain = ["store.musinsa.com"]
    
    def __init__(self, **kwargs):
        # 1w : 주간, page : 1
        self.base_url = "https://store.musinsa.com/app/contents/bestranking/?d_cat_cd=&u_cat_cd={}".format(kwargs["category"])
        self.start_urls = []
        last_page = self.get_last_page()
        for page in range(last_page + 1):
            self.start_urls.append(self.base_url + "&range=1w&price=&ex_soldout=&sale_goods=&new_product_yn=&list_kind=&page={}&display_cnt=90&sex=&popup=&sale_rate=&price1=&price2=&chk_new_product_yn=&chk_sale=&chk_soldout=".format(page))
        super().__init__(**kwargs)
        
    def get_last_page(self):
        req = requests.get(self.base_url)
        response = TextResponse(req.url, body=req.text, encoding="utf-8")
        last_page_num = response.xpath('//*[@id="product_list"]/div[2]/div[5]/div/div/a[14]/@onclick').extract()
        last_page_num = re.split('\s', last_page_num[0])[1]
        last_page_num = re.findall('\d+', last_page_num)
        last_page = list(map(int, last_page_num))[-1]
        last_page = int(last_page)
        return last_page
        
    def start_requests(self):
        for url in self.start_urls:
            yield scrapy.Request(url=url, callback=self.parse)

    def parse(self, response):
        links = response.xpath('//*[@id="searchList"]/li/div/div[2]/p[2]/a/@href').extract()
        links = list(map(lambda data:response.urljoin(data), links))
        for link in links:
            time.sleep(1)
            yield scrapy.Request(link, callback=self.get_content)
   
    def get_content(self, response):
        item = MusinsaItem()
        try:
            product_name_1 = response.xpath('//*[@id="page_product_detail"]/div[4]/div[3]/span/span[1]/text()')[0].extract()
            product_name_2 = response.xpath('//*[@id="page_product_detail"]/div[4]/div[3]/span/span[2]/text()')[0].extract()
            product_name_3 = product_name_1 + " " + product_name_2   
            item["product_name"] = product_name_3.replace("|", "")
        except:
            item["product_name"] = "-"
            
        try:
            item["brand_name"] = response.xpath('//*[@id="product_order_info"]/div[1]/ul/li[1]/p[2]/strong/a/text()')[0].extract()
        except:
            item["brand_name"] = "-"
            
        try:
            item["product_num"] = response.xpath('//*[@id="product_order_info"]/div[1]/ul/li[1]/p[2]/strong/text()')[1].extract()
        except:
            item["product_num"] = "-"
        
        try:
            product_spec_1 =response.xpath('//*[@id="page_product_detail"]/div[4]/div[3]/div[2]/p/a[1]/text()')[0].extract()
            product_spec_2 =response.xpath('//*[@id="page_product_detail"]/div[4]/div[3]/div[2]/p/a[2]/text()')[0].extract()
            item["product_spec"] = product_spec_1 + " / " +product_spec_2 
        except:
            item["product_spec"] = "-"
            
        try:
            gender = response.xpath('//*[@id="product_order_info"]/div[1]/ul/li[2]/p[2]/span/span/text()')[0:2].extract()
            item["gender"] = "".join(gender).strip()
        except:
            item["gender"] = "-"
            #gender = response.xpath('//*[@id="product_order_info"]/div[1]/ul/li[2]/p[2]/span[2]/span/text()').extract()
            #item["gender"] = "".join(gender).strip()
        
        try:
            item["good_num"] = response.xpath('//*[@id="product_order_info"]/div[1]/ul/li[5]/p[2]/span/text()')[0].extract()
        except:
            item["good_num"] = "-"
            
        try:
            item["review_count"] = response.xpath('//*[@id="product_order_info"]/div[1]/ul/li[6]/p[2]/a/text()')[0].extract().replace("건", "")
        except:
            item["review_count"] = "-"
            
        try :
            item["origin_price"] = response.xpath('//*[@id="goods_price"]/del/text()')[0].extract()
        except:
            #item["origin_price"] = "기획전"
            origin_price = response.xpath('//*[@id="goods_price"]/text()')[0].extract()
            item["origin_price"] = "".join(origin_price).strip()
            #item["origin_price"] = re.findall(('\d+\,\d+'), origin_price)
        
        try :
            item["sale_price"] = response.xpath('//*[@id="sale_price"]/text()')[0].extract()
        except :
            item["sale_price"] = "기획전"
        
        # 타겟 고객층
        target_name_1 = response.xpath('//*[@id="graph_summary_area"]/strong[1]/em/text()').extract()
        target_name_2 = response.xpath('//*[@id="graph_summary_area"]/span[1]/text()').extract()
        target_name_3 = response.xpath('//*[@id="graph_summary_area"]/span[2]/text()').extract()
        item["target_name"] = target_name_1[0] + " , " + target_name_2[0]
        
        item["link"] = response.url
        
        yield item
```

### 2-4. CSV 파일 생성

카테고리 값에 해당하는 값을 CSV 파일로 나누어 저장 할 수 있습니다. 

```
%%writefile run.sh
cd musinsa
scrapy crawl MusinsaRanking -o mr_1w_5p_001.csv -a category=001 #상 의 데이터 파일
scrapy crawl MusinsaRanking -o mr_1w_5p_002.csv -a category=002 #아우터 데이터 파일
```

### 2-5. 데이터 베이스 저장 (MongoDB)

MongoDB 데이터 베이스에 수집한 데이터를 저장할 수 있습니다. 
  - mongodb의 서버 주소는 변경하여 사용하면 됩니다.
  - 데이터베이스와 컬렉션 이름을 변경하여 사용 할 수 있습니다. 
  - (ex) db = client.musinsa, collection = db.musinsa_df

```
%%writefile musinsa/musinsa/mongodb.py
import pymongo

client = pymongo.MongoClient('mongodb://**.***.***.**:2701*/')
db = client.mr_1w_5p_001_002
collection = db.data
```

### 2-6. Pipeline 생성

pipeline 파일을 생성하여, 크롤러가 데이터를 수집 후에 수행할 작업들을 설정 할 수 있습니다. 
현재 이 pipeline은 mongodb에 수집한 데이터를 저장하며, 슬랙 메신저에 설정한 키워드에 맞는 상품 데이터를 전송하게끔 설정 되었습니다. 
  - icomming 웹 훅 url을 변경하여 사용할 수 있습니다.
  - self.keyword 값이 수집한 Item 데이터에 있으면, 해당 키워드의 데이터가 전송 됩니다.
  - self.keyword 값과 item[" "] 값을 변경하여 사용할 수 있습니다.
  - 전송할 데이터의 값을 변경할 수 있습니다.

```
%%writefile musinsa/musinsa/pipelines.py
import json
import requests
from .mongodb import collection


class MusinsaRankingPipeline(object):
    
    def __init__(self):
        self.webhook_url = "https://hooks.slack.com/services/~~~"
        self.keyword = "THISISNEVERTHAT"
    
    def send_msg(self, msg):
        payload = {
            "channel": "#musinsa_crawling.",
            "username": "sainthong",
            "text": msg,
        }
        
        requests.post(self.webhook_url, json.dumps(payload))
        time.sleep(1)
    
    def process_item(self, item, spider):
        data = {
            "product_spec": item["product_spec"],
            "brand_name": item["brand_name"],
            "product_name": item["product_name"],
            "product_num": item["product_num"],
            "gender": item["gender"],
            "origin_price": item["origin_price"],
            "sale_price": item["sale_price"],
            "good_num": item["good_num"],
            "review_count": item["review_count"],
            "target_name": item["target_name"],
            "link":item["link"]
        }
        
        collection.insert(data)
        
        if self.keyword in item["brand_name"]:
            self.send_msg("카테고리 : {},     브랜드명 : {},     상품명 : {},     판매가격 : {},     세일가격 : {},     좋아요 : {},     링크 : {}"
                          .format(item["product_spec"], item["brand_name"], item["product_name"], item["origin_price"], item["sale_price"], item["good_num"], item["link"]))
        
        return item
```

## 3. 수집한 데이터 확인

수집한 데이터의 CSV 파일을 DataFrame 으로 변환하여, 데이터 분석을 할 수 있습니다. 

### 3-1. 데이터 확인

수집한 데이터의 카테고리 값을 변수로 설정하여, 각각 저장된 CSV 파일을 합하여 확인 할 수 있습니다.
```
import pandas as pd

categories = ["001", "002"]
top_outer_dfs = [pd.read_csv("musinsa/mr_1w_5p_{}.csv".format(category)) for category in categories]
```

출력결과 (데이터 정렬은 안 되어 있음)
```
[             brand_name gender   good_num  \
 0          GROOVE RHYME      남      6,862   
 1                   LMC      남      4,172   
 2    NATIONALGEOGRAPHIC      남      9,543   
 3                   LMC      남      5,869   
 4                 JEMUT      남      6,765   
 5    NATIONALGEOGRAPHIC      남      6,976   
 6      MUSINSA STANDARD      남     13,878   
 7      MUSINSA STANDARD      남      7,836
 ...
```

### 3-2. 수집한 데이터의 수 확인

수집한 데이터의 카테고리별 데이터 수를 확인 할 수 있습니다.

```
[(category, len(df)) for category, df in zip(categories, top_outer_dfs)]
```


출력결과 상의 (001) 438개, 아우터 (002) 449 개
```
[('001', 438), ('002', 449)]
```

### 3-3. 수집한 데이터를 데이터프레임으로 변환

수집한 데이터 CSV 파일을 불러와 데이터프레임으로 변환할 수 있습니다. 

```
file = !ls ./musinsa # 해당 디렉토리의 파일명을 file 변수 선언

top_df = pd.read_csv("musinsa/{}".format(file[1])) # 상의 데이터 프레임
top_df = top_df[["ranking", "product_spec", "brand_name", "product_name", "product_num", 
    "gender", "origin_price", "sale_price", "good_num", "review_count", "target_name", "link"]]

outer_df = pd.read_csv("musinsa/{}".format(file[2])) # 아우터 데이터 프레임
outer_df = outer_df[["ranking", "product_spec", "brand_name", "product_name", "product_num", 
    "gender", "origin_price", "sale_price", "good_num", "review_count", "target_name", "link"]]

top_outer_df = pd.concat([top_df, outer_df])
top_outer_df.reset_index(drop=True, inplace=True)
```


```
	ranking	product_spec	brand_name	product_name	product_num	gender	origin_price	sale_price	good_num	review_count	target_name	link
882	NaN	아우터 / 나일론/코치 재킷	NIKE	아카데미 바람막이 자켓 블랙	2325968	NaN	129,000	99,000	(결제완료-반품)	-	19~24 , 남성	https://store.musinsa.com/app/product/detail/1...
883	NaN	아우터 / 카디건	OiOi	HEART LOGO KNIT CARDIGAN_navy	OI20SS_07	여	92,000	73,600	943	94	19~24 , 여성	https://store.musinsa.com/app/product/detail/1...
884	NaN	아우터 / 후드 집업	LAFUDGESTORE	[썸머 ver.][SET] 오디너리 아마드 후드집업 셋업_Metal Black	f-1004	남	82,000	65,600	594	31	25~34 , 남성	https://store.musinsa.com/app/product/detail/1...
885	NaN	아우터 / 블루종/MA-1	ALPHA INDUSTRIES	MA-1 레귤러핏 - 세이지 그린 / MJM21000C1-SGR	MJM21000C1-SGR	남	215,000	기획전	2,429	5,599	25~34 , 남성	https://store.musinsa.com/app/product/detail/4...
886	NaN	아우터 / 수트/블레이저 재킷	SIGNATURE	[여름원단추가]세미오버핏 싱글 블레이저 자켓[검정]	BL-004	남	159,000	73,000	9,637	4,995	19~24 , 남성	https://store.musinsa.com/app/product/detail/7...
```

## 4. 주의 사항
Scrapy를 사용하여 데이터를 크롤링 할 시, 데이터 요청과 응답 과정에서 디버깅이 발생 할 수 있습니다. 디버깅이 발생할 경우 데이터 수집의 로딩이
중단 되지 않는 오류가 발생합니다.

settings.py 의 AUTOTHROTTLE 설정을 변경하여, 스크래피의 요청, 응답 과정을 자동으로 안정화 할 수 있습니다.

1) 디렉터리의 settings.py 열기
2) AUTOTHROTTLE_ENABLED = True / AUTOTHROTTLE_TARGET_CONCURRENCY = 1.0 을 활성화
3) AUTOTHROTTLE_TARGET_CONCURRENCY 값을 높여준다. (ex) "2.0" 
