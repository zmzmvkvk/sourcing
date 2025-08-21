# 💻 온라인 상품 소싱 자동화 프로그램 개발 요청 (A-to-Z)

## 1. 프로그램 개요

**프로젝트명**: 온라인 상품 소싱 자동화 솔루션 (OSS-Bot)

**목표**: 네이버 데이터랩, 아이템스카우트, 쿠팡 윙 등의 데이터를 크롤링하여 수익성이 높고 경쟁이 낮은 상품을 자동으로 발굴하고 사용자에게 추천하는 솔루션을 구축합니다.

## 2. 기술 스택

### Backend
- **언어**: Python
- **웹 프레임워크**: Flask 또는 Django
- **크롤링**: requests, BeautifulSoup, Selenium
- **데이터베이스**: MySQL

### Frontend
- **프레임워크**: React
- **향후 계획**: 데스크탑 앱으로 변경 예정

## 3. 핵심 기능별 세부 요청사항

### Phase 1: 데이터 크롤링 모듈

Python을 기반으로 아래 웹사이트에서 필요한 데이터를 자동으로 수집하는 모듈을 구현합니다. requests, BeautifulSoup, Selenium 라이브러리를 사용하며, Selenium은 로그인 또는 동적 페이지 처리를 위해 필요합니다.

#### 1.1 네이버 데이터랩
- **수집 데이터**: 특정 카테고리 또는 키워드의 월간 검색량 추이
- **구현 방법**: 
```python - coding: utf-8 -
import os
import sys
import urllib.request
client_id = "VoXDKXzudUkyPZEDg0uF"
client_secret = "0P3fl_JiWB"
url = "https://openapi.naver.com/v1/datalab/search";
body = "{\"startDate\":\"2017-01-01\",\"endDate\":\"2017-04-30\",\"timeUnit\":\"month\",\"keywordGroups\":[{\"groupName\":\"한글\",\"keywords\":[\"한글\",\"korean\"]},{\"groupName\":\"영어\",\"keywords\":[\"영어\",\"english\"]}],\"device\":\"pc\",\"ages\":[\"1\",\"2\"],\"gender\":\"f\"}";

request = urllib.request.Request(url)
request.add_header("X-Naver-Client-Id",client_id)
request.add_header("X-Naver-Client-Secret",client_secret)
request.add_header("Content-Type","application/json")
response = urllib.request.urlopen(request, data=body.encode("utf-8"))
rescode = response.getcode()
if(rescode==200):
    response_body = response.read()
    print(response_body.decode('utf-8'))
else:
    print("Error Code:" + rescode)
```

#### 1.2 아이템스카우트
- **수집 데이터**: 키워드별 총 상품 수, 경쟁 강도 등 분석 데이터
- **구현 방법**: API 또는 웹 크롤링

#### 1.3 쿠팡
- **수집 데이터**: 
  - 상품 페이지의 판매가
  - 리뷰 수
  - 별점
  - 상품평 내용(텍스트)
  - 로켓그로스 입점 여부
- **구현 방법**: BeautifulSoup + Selenium

#### 1.4 1688.com
- **수집 데이터**: 키워드를 기반으로 상품 원가와 판매자 평점
- **구현 방법**: 중국 사이트 접근을 위한 특별 설정 필요

### Phase 2: 분석 및 필터링 알고리즘

수집된 데이터를 기반으로 아래와 같은 핵심 지표를 산출하는 알고리즘을 구현합니다.

#### 2.1 수익성 지수
- **계산식**: (판매가) - (원가 + 국제 배송비 + 쿠팡 수수료 + 광고비) = 예상 순수익
- **요청**: 각 비용 변수를 입력받아 최종 마진율을 계산하는 함수를 작성

```python
def calculate_profitability(selling_price, cost_price, shipping_fee, commission_rate, ad_cost):
    """
    수익성 지수 계산 함수
    
    Args:
        selling_price: 판매가
        cost_price: 원가
        shipping_fee: 국제 배송비
        commission_rate: 쿠팡 수수료율 (%)
        ad_cost: 광고비
    
    Returns:
        dict: 순수익, 마진율 등
    """
    commission = selling_price * (commission_rate / 100)
    total_cost = cost_price + shipping_fee + commission + ad_cost
    net_profit = selling_price - total_cost
    margin_rate = (net_profit / selling_price) * 100
    
    return {
        'net_profit': net_profit,
        'margin_rate': margin_rate,
        'total_cost': total_cost
    }
```

#### 2.2 경쟁률 점수 ('블루 오션 지수')
- **계산식**: (월간 검색량) / (총 상품 수) = 경쟁률 점수
- **요청**: 경쟁률 점수를 100점 만점으로 환산하여 시각적으로 쉽게 파악할 수 있도록 구현

```python
def calculate_competition_score(monthly_search_volume, total_products):
    """
    경쟁률 점수 계산 함수 (블루 오션 지수)
    
    Args:
        monthly_search_volume: 월간 검색량
        total_products: 총 상품 수
    
    Returns:
        int: 100점 만점 경쟁률 점수
    """
    if total_products == 0:
        return 100
    
    raw_score = monthly_search_volume / total_products
    # 정규화 로직 (0-100점 범위로 변환)
    normalized_score = min(100, raw_score * 10)  # 조정 필요
    
    return round(normalized_score)
```

#### 2.3 상품 위험 점수
- **로직**: 상품평 텍스트를 분석하여 '불량', '파손', '느린 배송' 등 부정적 키워드가 포함된 리뷰의 비율을 계산
- **요청**: 이 비율을 기반으로 상품의 잠재적 위험성을 나타내는 점수를 산출

```python
def calculate_risk_score(reviews):
    """
    상품 위험 점수 계산 함수
    
    Args:
        reviews: 상품평 리스트
    
    Returns:
        int: 위험 점수 (0-100, 높을수록 위험)
    """
    negative_keywords = ['불량', '파손', '느린 배송', '배송 지연', '품질 문제', '불만']
    total_reviews = len(reviews)
    
    if total_reviews == 0:
        return 0
    
    negative_count = 0
    for review in reviews:
        if any(keyword in review for keyword in negative_keywords):
            negative_count += 1
    
    risk_percentage = (negative_count / total_reviews) * 100
    return round(risk_percentage)
```

### Phase 3: 사용자 인터페이스(UI) 및 데이터 관리

웹 기반의 간단한 인터페이스와 데이터를 저장할 수 있는 시스템을 구축합니다.

#### 3.1 백엔드 (Flask/Django)
- **웹 프레임워크**: Python의 Flask 또는 Django를 사용하여 사용자 인터페이스를 만듭니다.

#### 3.2 대시보드 기능
- **검색창**: 사용자가 키워드를 입력할 수 있도록 합니다.
- **결과 목록**: 입력된 키워드에 대한 분석 결과를 테이블 형태로 보여줍니다.
  - 상품 이미지
  - 판매가
  - 마진율
  - 경쟁률 점수
  - 위험 점수

#### 3.3 데이터베이스 설계
수집하고 분석한 상품 데이터를 MySQL에 저장하여 관리합니다.

```sql
-- 상품 정보 테이블
CREATE TABLE products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    keyword VARCHAR(255) NOT NULL,
    product_name VARCHAR(500),
    selling_price DECIMAL(10,2),
    cost_price DECIMAL(10,2),
    image_url TEXT,
    coupang_url TEXT,
    alibaba_url TEXT,
    review_count INT,
    rating DECIMAL(3,2),
    rocket_growth BOOLEAN,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- 분석 결과 테이블
CREATE TABLE analysis_results (
    id INT AUTO_INCREMENT PRIMARY KEY,
    product_id INT,
    monthly_search_volume INT,
    total_products INT,
    competition_score INT,
    profitability_score DECIMAL(5,2),
    risk_score INT,
    margin_rate DECIMAL(5,2),
    net_profit DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (product_id) REFERENCES products(id)
);

-- 사용자 알림 설정 테이블
CREATE TABLE user_alerts (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_email VARCHAR(255),
    min_margin_rate DECIMAL(5,2),
    min_competition_score INT,
    max_risk_score INT,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### 3.4 알림 기능
사용자가 설정한 기준(예: '마진율 20% 이상', '경쟁률 점수 80점 이상')을 충족하는 상품이 발견되면 이메일로 알림을 보내는 기능을 구현합니다.

```python
def send_alert_email(user_email, product_data):
    """
    조건에 맞는 상품 발견 시 이메일 알림 발송
    
    Args:
        user_email: 사용자 이메일
        product_data: 상품 데이터
    """
    # 이메일 발송 로직 구현
    pass
```

## 4. 개발 로드맵

### Phase 1 (4주): 기본 크롤링 모듈 개발
- [ ] 개발 환경 설정
- [ ] 네이버 데이터랩 크롤러 구현
- [ ] 쿠팡 크롤러 구현
- [ ] 1688.com 크롤러 구현
- [ ] 아이템스카우트 크롤러 구현

### Phase 2 (3주): 데이터 분석 알고리즘 개발
- [ ] 수익성 지수 계산 함수 구현
- [ ] 경쟁률 점수 계산 함수 구현
- [ ] 상품 위험 점수 계산 함수 구현
- [ ] 데이터 정규화 및 검증 로직

### Phase 3 (4주): UI/UX 및 시스템 통합
- [ ] MySQL 데이터베이스 설계 및 구축
- [ ] Flask/Django 백엔드 API 개발
- [ ] React 프론트엔드 개발
- [ ] 대시보드 및 검색 기능 구현
- [ ] 이메일 알림 시스템 구현

### Phase 4 (2주): 테스트 및 최적화
- [ ] 전체 시스템 통합 테스트
- [ ] 성능 최적화
- [ ] 사용자 피드백 반영
- [ ] 배포 준비

## 5. 주요 고려사항

### 5.1 법적 고려사항
- 웹 크롤링 시 각 사이트의 robots.txt 및 이용약관 준수
- 개인정보 처리 방침 수립
- 데이터 수집 및 활용에 대한 법적 검토

### 5.2 기술적 고려사항
- 크롤링 봇 차단 대응 (User-Agent 로테이션, 지연 시간 설정)
- 대용량 데이터 처리를 위한 배치 처리 시스템
- API 호출 제한 및 에러 핸들링
- 데이터 품질 관리 및 검증

### 5.3 확장성 고려사항
- 추가 쇼핑몰 데이터 소스 연동 가능성
- 데스크탑 앱 전환을 위한 아키텍처 설계
- 멀티 사용자 지원을 위한 확장 계획

## 6. 예상 산출물

1. **크롤링 모듈**: Python 패키지 형태의 각 사이트별 크롤러
2. **분석 엔진**: 수익성, 경쟁률, 위험도 분석 알고리즘
3. **웹 애플리케이션**: React + Flask/Django 기반 대시보드
4. **데이터베이스**: MySQL 기반 상품 및 분석 데이터 저장소
5. **알림 시스템**: 이메일 기반 실시간 알림 서비스
6. **API 문서**: 각 모듈별 상세 API 문서
7. **사용자 매뉴얼**: 시스템 사용법 및 설정 가이드

---

이 문서는 온라인 상품 소싱 자동화 프로그램(OSS-Bot)의 전체적인 개발 계획을 담고 있습니다. 각 Phase별로 단계적 개발을 진행하여 안정적이고 효율적인 솔루션을 구축할 예정입니다.
