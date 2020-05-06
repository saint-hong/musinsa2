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






