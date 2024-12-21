# Team Example 네이버 API 분석 프로젝트
## 팀원 및 역할
- 박태중, 서한률, 이정태, 정호
  
## 프로젝트 목표
- 네이버 API를 활용한 데이터 수집 및 분석
- 주제: [무신사 매출 증대를 위한 30대 남성의 카테고리 추이 파악]

## 진행 상황
- [네이버 데이터랩의 성별/연령별 검색 트렌드, 시즌별 키워드 분석]
- [ ] 발표자료 작성

## 분석 내용 및 시각화
- 프로젝트 진행 내용

## 인사이트 및 결론
- 인사이트 및 결론

```
import requests
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns
import matplotlib.font_manager as fm 



client_id = "Rdxqa5dG01AumJcUJFg8"
client_secret = "7RfDqVFoEn"

url = "https://openapi.naver.com/v1/datalab/search"

headers = {
    "X-Naver-Client-Id": client_id,
    "X-Naver-Client-Secret": client_secret,
    "Content-Type": "application/json"
}

def get_keyword_trend(keywords, start_date="2024-01-01", end_date="2024-12-01", time_unit="month"):
    """
    keywords: ['무신사', '티셔츠'] 처럼 2개 키워드
    반환: date(YYYY-MM 형태), keyword, ratio 컬럼을 갖는 DataFrame
    """
    keyword_groups = []
    for kw in keywords:
        keyword_groups.append({
            "groupName": kw,
            "keywords": [kw]
        })
    
    body = {
        "startDate": start_date,
        "endDate": end_date,
        "timeUnit": time_unit,  # 'month'
        "keywordGroups": keyword_groups
    }

    response = requests.post(url, headers=headers, json=body)
    if response.status_code == 200:
        data = response.json()
        results = []
        for item in data["results"]:
            group_name = item["title"]
            for point in item["data"]:
                # 'period' = '2024-01' 등
                results.append({
                    "date": point["period"],   
                    "keyword": group_name,
                    "ratio": point["ratio"]
                })
        return pd.DataFrame(results)
    else:
        print("API 요청 실패:", response.status_code, response.text)
        return pd.DataFrame()  # 빈 DF 반환

# ---------------------------
# 1) 무신사 + 10개 키워드 각각 월별 데이터 수집
# ---------------------------
base_keyword = "무신사"
other_keywords = [
    "T-shirt", "Sweatshirt", "Padded Jacket", "Socks", "Scarf",
    "Jeans", "Cardigan", "Sneakers", "Jacket", "Suit"
]

all_list = []  # 전체 결과를 모을 리스트

for kw in other_keywords:
    kw_set = [base_keyword, kw]  # 2개
    df_temp = get_keyword_trend(
        keywords=kw_set,
        start_date="2024-01-01",
        end_date="2024-12-01",
        time_unit="month"
    )
    if df_temp.empty:
        continue
    
    # 'month' 컬럼: 사실 timeUnit="month"면 date가 이미 YYYY-MM,
    #              굳이 변환 없이 'date'를 그대로 써도 됨.
    df_temp["month"] = df_temp["date"]

    all_list.append(df_temp)

# ---------------------------
# 2) 전체 데이터 합치기
# ---------------------------
if not all_list:
    print("데이터가 전혀 수집되지 않았습니다. (모두 실패)")
    exit()

combined_df = pd.concat(all_list, ignore_index=True)
# columns: [date, keyword, ratio, month]

# ---------------------------
# 3) 월별 ratio 표 만들기
#    (행=month, 열=keyword, 값=ratio)
# ---------------------------
pivot_df = combined_df.pivot_table(
    index="month",
    columns="keyword",
    values="ratio",
    aggfunc="mean"  # 무신사가 중복되어 들어올 수도 있으므로 mean
)

# 결측치가 있을 수 있으니, 필요한 경우 제거(옵션)
# pivot_df.dropna(axis=0, how="any", inplace=True)

pivot_df.sort_index(inplace=True)  # month 순서대로 (예: 2024-01 ~ 2024-12)

print("=== 월별 ratio 표 (무신사 + 10개 키워드) ===")
display(pivot_df)

# CSV 저장 (월별 키워드 ratio)
pivot_df.reset_index().to_csv("month_keyword_ratio.csv", index=False, encoding="utf-8-sig")
print("\n월별 키워드 ratio를 'month_keyword_ratio.csv'로 저장했습니다.")

# ---------------------------
# ---------------------------
# 4) 상관분석 (12개월 시계열 기반)
# ---------------------------
# 전체 상관행렬
corr_matrix = pivot_df.corr(method="pearson")

print("=== 키워드 전체 상관행렬 ===")
display(corr_matrix)

# CSV로 저장
corr_matrix.to_csv("keyword_correlation_matrix.csv", encoding="utf-8-sig")
print("\n키워드 전체 상관행렬을 'keyword_correlation_matrix.csv'로 저장했습니다.")

# ---------------------------
# 6) 상관계수 히트맵 시각화
# ---------------------------
plt.figure(figsize=(10, 8))  # 히트맵 크기 설정
sns.heatmap(
    corr_matrix,
    annot=True,  # 상관계수 값을 표시
    fmt=".2f",   # 소수점 두 자리까지 표시
    cmap="coolwarm",  # 색상 팔레트
    linewidths=0.5,  # 셀 간격
    cbar=True  # 컬러바 표시
)
plt.title("Keyword Correlation Matrix")
plt.tight_layout()

# 그래프 저장
plt.savefig("keyword_correlation_heatmap.png", dpi=300)

# 한글 폰트 설정
plt.rcParams["font.family"] = "NanumGothic"  # 나눔고딕
plt.rcParams["axes.unicode_minus"] = False  # 음수 기호 깨짐 방지
plt.show()
print("\n상관계수 히트맵이 'keyword_correlation_heatmap.png'로 저장되었습니다.")
```
