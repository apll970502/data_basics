# Chapter 04 자습 과제

- 이름: 안형준
- GitHub ID: apll970502
- 작성일: 2026-09-14

Chapter 04 자습 과제 1번부터 5번까지를 하나의 노트북에서 수행한다.
분석 범위는 과제마다 명시하고, 모든 병합과 집계는 행 수와 합계로 검증한다.

## 공통 준비

과제 전체가 같은 분석셋을 사용하므로 여기서 한 번만 만든다.


```python
from pathlib import Path
import pandas as pd

project_root = Path.cwd()
while project_root.name in ("ch04", "notebooks"):
    project_root = project_root.parent
data_dir = project_root / "data" / "raw"

print("프로젝트 루트:", project_root)
print("데이터 폴더 존재:", data_dir.exists())
```

    프로젝트 루트: C:\dev\llm-data-analysis-course
    데이터 폴더 존재: True
    


```python
customers = pd.read_csv(data_dir / "customers.csv")
products = pd.read_csv(data_dir / "products.csv")
orders = pd.read_csv(data_dir / "orders.csv")
order_items = pd.read_csv(data_dir / "order_items.csv")

for name, df in {
    "customers": customers,
    "products": products,
    "orders": orders,
    "order_items": order_items,
}.items():
    print(name, df.shape, df.columns.tolist())
```

    customers (150, 6) ['customer_id', 'name', 'gender', 'age', 'city', 'signup_date']
    products (100, 4) ['product_id', 'product_name', 'category', 'price']
    orders (300, 5) ['order_id', 'customer_id', 'order_date', 'payment_method', 'order_status']
    order_items (764, 5) ['order_item_id', 'order_id', 'product_id', 'quantity', 'unit_price']
    


```python
# 기준 테이블의 주요 키가 고유한지 먼저 확인한다
key_checks = {
    "customers.customer_id": customers["customer_id"],
    "products.product_id": products["product_id"],
    "orders.order_id": orders["order_id"],
    "order_items.order_item_id": order_items["order_item_id"],
}
for name, series in key_checks.items():
    print(name, "결측:", series.isna().sum(), "중복:", series.duplicated().sum())
```

    customers.customer_id 결측: 0 중복: 0
    products.product_id 결측: 0 중복: 0
    orders.order_id 결측: 0 중복: 0
    order_items.order_item_id 결측: 0 중복: 0
    


```python
# 실제 주문 상태값을 확인한 뒤 조건을 쓴다
print(orders["order_status"].value_counts(dropna=False))
print()
print(orders["payment_method"].value_counts(dropna=False))
```

    order_status
    completed    184
    cancelled     64
    refunded      52
    Name: count, dtype: int64
    
    payment_method
    kakao_pay        79
    naver_pay        77
    bank_transfer    74
    card             70
    Name: count, dtype: int64
    


```python
# 파생 컬럼: 주문상세 한 행의 금액
order_items_work = order_items.copy()
order_items_work["line_total"] = (
    order_items_work["quantity"] * order_items_work["unit_price"]
)

# 수작업 검증
sample = order_items_work.iloc[0]
expected = sample["quantity"] * sample["unit_price"]
actual = sample["line_total"]
print("수작업:", expected, "/ 파생 컬럼:", actual, "/ 일치:", expected == actual)

all_order_amount = order_items_work["line_total"].sum()
print("전체 주문상세 금액:", f"{all_order_amount:,}")
```

    수작업: 306000 / 파생 컬럼: 306000 / 일치: True
    전체 주문상세 금액: 255,610,000
    

위 합계는 취소와 환불 주문을 포함하므로 **매출이 아니라 전체 주문상세 금액**으로 표현한다.
주문 상태를 연결하고 완료 주문만 남긴 뒤에야 매출로 해석할 수 있다.


```python
orders_for_merge = orders[
    ["order_id", "customer_id", "order_date", "payment_method", "order_status"]
].copy()

order_sales = order_items_work.merge(
    orders_for_merge,
    on="order_id",
    how="left",
    validate="many_to_one",
    indicator="order_match",
)

print("병합 전 행 수:", len(order_items_work))
print("병합 후 행 수:", len(order_sales))
print(order_sales["order_match"].value_counts(dropna=False))
```

    병합 전 행 수: 764
    병합 후 행 수: 764
    order_match
    both          764
    left_only       0
    right_only      0
    Name: count, dtype: int64
    


```python
completed_sales = order_sales[order_sales["order_status"] == "completed"].copy()

products_for_merge = products[["product_id", "product_name", "category"]].copy()
completed_items = completed_sales.merge(
    products_for_merge,
    on="product_id",
    how="left",
    validate="many_to_one",
    indicator="product_match",
)

print("완료 주문상세 행:", len(completed_items))
print("완료 주문 수:", completed_items["order_id"].nunique())
print("완료 주문 고객 수:", completed_items["customer_id"].nunique())
print("완료 주문 매출:", f'{completed_items["line_total"].sum():,}')
print(completed_items["product_match"].value_counts(dropna=False))
```

    완료 주문상세 행: 474
    완료 주문 수: 184
    완료 주문 고객 수: 100
    완료 주문 매출: 148,990,000
    product_match
    both          474
    left_only       0
    right_only      0
    Name: count, dtype: int64
    

---

# 과제 1. 분석 질문을 pandas 흐름으로 변환하기

선택한 질문: **결제 수단별 완료 주문 매출**

## 분석 질문

완료된 주문만을 대상으로 할 때, 결제 수단별 매출은 각각 얼마인가?

## 분석 범위

- 매출 기준: `order_status`가 `completed`인 주문만 포함한다
- 금액 기준: `quantity` × `unit_price`
- 주문 수 기준: 고유 `order_id` 개수
- 고객 수 기준: 고유 `customer_id` 개수
- 취소·환불 주문: 집계에서 제외한다

## 필요한 DataFrame

- `order_items`: 수량과 판매 단가를 가지고 있다
- `orders`: 결제 수단과 주문 상태를 가지고 있다

`products`는 결제 수단과 무관하므로 이 질문에는 필요하지 않다.

## 필요한 컬럼

- `order_items`: `order_item_id`, `order_id`, `quantity`, `unit_price`
- `orders`: `order_id`, `customer_id`, `payment_method`, `order_status`

## 각 데이터에서 한 행의 의미

- `order_items` 한 행: 하나의 주문에 포함된 **상품 한 항목**
- `orders` 한 행: **주문 한 건**

## 필터 조건

`order_status == "completed"`

## 파생 컬럼

`line_total = quantity * unit_price` (주문상세 한 항목의 금액)

## 병합 키와 관계 수

`order_items.order_id` → `orders.order_id`, **many_to_one**

왼쪽은 한 주문에 여러 상품 항목이 있어 키가 반복되고, 오른쪽은 주문 번호가 고유하다.

## 그룹바이 기준

`payment_method`

## 집계 지표

`total_sales`, `order_count`, `customer_count`, `quantity_sold`

## 검증 방법

1. 병합 전후 행 수가 같은지 확인한다
2. `indicator` 결과가 모두 `both`인지 확인한다
3. 결제 수단별 매출 합계와 완료 주문 전체 매출이 일치하는지 대조한다

## 결과 한 행의 의미

결제 수단 하나. 예를 들어 `card` 행은 카드로 결제된 완료 주문 전체의 요약이다.


```python
payment_sales = (
    completed_items
    .groupby("payment_method", as_index=False)
    .agg(
        total_sales=("line_total", "sum"),
        order_count=("order_id", "nunique"),
        customer_count=("customer_id", "nunique"),
        quantity_sold=("quantity", "sum"),
    )
    .sort_values("total_sales", ascending=False)
)
display(payment_sales)
```


<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>payment_method</th>
      <th>total_sales</th>
      <th>order_count</th>
      <th>customer_count</th>
      <th>quantity_sold</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>3</th>
      <td>naver_pay</td>
      <td>43500000</td>
      <td>51</td>
      <td>38</td>
      <td>414</td>
    </tr>
    <tr>
      <th>2</th>
      <td>kakao_pay</td>
      <td>39342000</td>
      <td>49</td>
      <td>39</td>
      <td>385</td>
    </tr>
    <tr>
      <th>0</th>
      <td>bank_transfer</td>
      <td>34342000</td>
      <td>45</td>
      <td>38</td>
      <td>324</td>
    </tr>
    <tr>
      <th>1</th>
      <td>card</td>
      <td>31806000</td>
      <td>39</td>
      <td>33</td>
      <td>319</td>
    </tr>
  </tbody>
</table>
</div>



```python
# 검증: 결제 수단별 합계와 완료 주문 전체 합계 대조
summary_total = payment_sales["total_sales"].sum()
source_total = completed_items["line_total"].sum()
print("원본 합계:", f"{source_total:,}")
print("요약 합계:", f"{summary_total:,}")
print("차이:", source_total - summary_total)
print("일치:", source_total == summary_total)
```

    원본 합계: 148,990,000
    요약 합계: 148,990,000
    차이: 0
    일치: True
    

---

# 과제 2. 판매량과 매출 순위 비교


```python
product_sales = (
    completed_items
    .groupby(["product_id", "product_name", "category"], as_index=False)
    .agg(
        quantity_sold=("quantity", "sum"),
        total_sales=("line_total", "sum"),
        order_count=("order_id", "nunique"),
    )
)

print("=== 판매량 상위 10 ===")
display(product_sales.sort_values("quantity_sold", ascending=False).head(10))
print("=== 매출 상위 10 ===")
display(product_sales.sort_values("total_sales", ascending=False).head(10))
```

    === 판매량 상위 10 ===
    


<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>product_id</th>
      <th>product_name</th>
      <th>category</th>
      <th>quantity_sold</th>
      <th>total_sales</th>
      <th>order_count</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>39</th>
      <td>41</td>
      <td>스포츠 상품 041</td>
      <td>스포츠</td>
      <td>35</td>
      <td>5705000</td>
      <td>12</td>
    </tr>
    <tr>
      <th>10</th>
      <td>11</td>
      <td>패션 상품 011</td>
      <td>패션</td>
      <td>31</td>
      <td>3565000</td>
      <td>7</td>
    </tr>
    <tr>
      <th>74</th>
      <td>77</td>
      <td>전자기기 상품 077</td>
      <td>전자기기</td>
      <td>31</td>
      <td>2201000</td>
      <td>7</td>
    </tr>
    <tr>
      <th>86</th>
      <td>89</td>
      <td>생활용품 상품 089</td>
      <td>생활용품</td>
      <td>30</td>
      <td>3090000</td>
      <td>11</td>
    </tr>
    <tr>
      <th>46</th>
      <td>48</td>
      <td>식품 상품 048</td>
      <td>식품</td>
      <td>29</td>
      <td>1943000</td>
      <td>8</td>
    </tr>
    <tr>
      <th>20</th>
      <td>22</td>
      <td>생활용품 상품 022</td>
      <td>생활용품</td>
      <td>29</td>
      <td>3248000</td>
      <td>8</td>
    </tr>
    <tr>
      <th>79</th>
      <td>82</td>
      <td>생활용품 상품 082</td>
      <td>생활용품</td>
      <td>28</td>
      <td>2100000</td>
      <td>8</td>
    </tr>
    <tr>
      <th>85</th>
      <td>88</td>
      <td>생활용품 상품 088</td>
      <td>생활용품</td>
      <td>27</td>
      <td>1161000</td>
      <td>7</td>
    </tr>
    <tr>
      <th>17</th>
      <td>18</td>
      <td>스포츠 상품 018</td>
      <td>스포츠</td>
      <td>26</td>
      <td>2834000</td>
      <td>11</td>
    </tr>
    <tr>
      <th>88</th>
      <td>91</td>
      <td>스포츠 상품 091</td>
      <td>스포츠</td>
      <td>26</td>
      <td>1534000</td>
      <td>9</td>
    </tr>
  </tbody>
</table>
</div>


    === 매출 상위 10 ===
    


<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>product_id</th>
      <th>product_name</th>
      <th>category</th>
      <th>quantity_sold</th>
      <th>total_sales</th>
      <th>order_count</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>39</th>
      <td>41</td>
      <td>스포츠 상품 041</td>
      <td>스포츠</td>
      <td>35</td>
      <td>5705000</td>
      <td>12</td>
    </tr>
    <tr>
      <th>11</th>
      <td>12</td>
      <td>식품 상품 012</td>
      <td>식품</td>
      <td>25</td>
      <td>4375000</td>
      <td>7</td>
    </tr>
    <tr>
      <th>8</th>
      <td>9</td>
      <td>스포츠 상품 009</td>
      <td>스포츠</td>
      <td>20</td>
      <td>3860000</td>
      <td>6</td>
    </tr>
    <tr>
      <th>70</th>
      <td>72</td>
      <td>뷰티 상품 072</td>
      <td>뷰티</td>
      <td>20</td>
      <td>3780000</td>
      <td>6</td>
    </tr>
    <tr>
      <th>69</th>
      <td>71</td>
      <td>전자기기 상품 071</td>
      <td>전자기기</td>
      <td>23</td>
      <td>3703000</td>
      <td>5</td>
    </tr>
    <tr>
      <th>66</th>
      <td>68</td>
      <td>스포츠 상품 068</td>
      <td>스포츠</td>
      <td>26</td>
      <td>3640000</td>
      <td>8</td>
    </tr>
    <tr>
      <th>78</th>
      <td>81</td>
      <td>전자기기 상품 081</td>
      <td>전자기기</td>
      <td>22</td>
      <td>3630000</td>
      <td>6</td>
    </tr>
    <tr>
      <th>10</th>
      <td>11</td>
      <td>패션 상품 011</td>
      <td>패션</td>
      <td>31</td>
      <td>3565000</td>
      <td>7</td>
    </tr>
    <tr>
      <th>20</th>
      <td>22</td>
      <td>생활용품 상품 022</td>
      <td>생활용품</td>
      <td>29</td>
      <td>3248000</td>
      <td>8</td>
    </tr>
    <tr>
      <th>86</th>
      <td>89</td>
      <td>생활용품 상품 089</td>
      <td>생활용품</td>
      <td>30</td>
      <td>3090000</td>
      <td>11</td>
    </tr>
  </tbody>
</table>
</div>



```python
top_quantity = product_sales.loc[product_sales["quantity_sold"].idxmax()]
top_sales = product_sales.loc[product_sales["total_sales"].idxmax()]

print("판매량 1위:", top_quantity["product_name"],
      "| 수량", top_quantity["quantity_sold"],
      "| 매출", f'{top_quantity["total_sales"]:,}')
print("매출  1위:", top_sales["product_name"],
      "| 수량", top_sales["quantity_sold"],
      "| 매출", f'{top_sales["total_sales"]:,}')
print()
print("두 상품이 같은가:", top_quantity["product_id"] == top_sales["product_id"])
```

    판매량 1위: 스포츠 상품 041 | 수량 35 | 매출 5,705,000
    매출  1위: 스포츠 상품 041 | 수량 35 | 매출 5,705,000
    
    두 상품이 같은가: True
    


```python
# 평균 판매 단가를 함께 보면 원인이 드러난다
compare = product_sales.copy()
compare["avg_unit_price"] = (compare["total_sales"] / compare["quantity_sold"]).round(0)
compare["quantity_rank"] = compare["quantity_sold"].rank(ascending=False, method="min").astype(int)
compare["sales_rank"] = compare["total_sales"].rank(ascending=False, method="min").astype(int)
compare["rank_gap"] = compare["quantity_rank"] - compare["sales_rank"]

view_cols = [
    "product_name", "category", "quantity_sold", "total_sales",
    "avg_unit_price", "quantity_rank", "sales_rank",
]
print("=== 매출 상위 8 (판매량 순위와 나란히) ===")
display(compare.sort_values("sales_rank").head(8)[view_cols])
print("=== 판매량 상위 8 (매출 순위와 나란히) ===")
display(compare.sort_values("quantity_rank").head(8)[view_cols])
```

    === 매출 상위 8 (판매량 순위와 나란히) ===
    


<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>product_name</th>
      <th>category</th>
      <th>quantity_sold</th>
      <th>total_sales</th>
      <th>avg_unit_price</th>
      <th>quantity_rank</th>
      <th>sales_rank</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>39</th>
      <td>스포츠 상품 041</td>
      <td>스포츠</td>
      <td>35</td>
      <td>5705000</td>
      <td>163000.0</td>
      <td>1</td>
      <td>1</td>
    </tr>
    <tr>
      <th>11</th>
      <td>식품 상품 012</td>
      <td>식품</td>
      <td>25</td>
      <td>4375000</td>
      <td>175000.0</td>
      <td>13</td>
      <td>2</td>
    </tr>
    <tr>
      <th>8</th>
      <td>스포츠 상품 009</td>
      <td>스포츠</td>
      <td>20</td>
      <td>3860000</td>
      <td>193000.0</td>
      <td>26</td>
      <td>3</td>
    </tr>
    <tr>
      <th>70</th>
      <td>뷰티 상품 072</td>
      <td>뷰티</td>
      <td>20</td>
      <td>3780000</td>
      <td>189000.0</td>
      <td>26</td>
      <td>4</td>
    </tr>
    <tr>
      <th>69</th>
      <td>전자기기 상품 071</td>
      <td>전자기기</td>
      <td>23</td>
      <td>3703000</td>
      <td>161000.0</td>
      <td>16</td>
      <td>5</td>
    </tr>
    <tr>
      <th>66</th>
      <td>스포츠 상품 068</td>
      <td>스포츠</td>
      <td>26</td>
      <td>3640000</td>
      <td>140000.0</td>
      <td>9</td>
      <td>6</td>
    </tr>
    <tr>
      <th>78</th>
      <td>전자기기 상품 081</td>
      <td>전자기기</td>
      <td>22</td>
      <td>3630000</td>
      <td>165000.0</td>
      <td>17</td>
      <td>7</td>
    </tr>
    <tr>
      <th>10</th>
      <td>패션 상품 011</td>
      <td>패션</td>
      <td>31</td>
      <td>3565000</td>
      <td>115000.0</td>
      <td>2</td>
      <td>8</td>
    </tr>
  </tbody>
</table>
</div>


    === 판매량 상위 8 (매출 순위와 나란히) ===
    


<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>product_name</th>
      <th>category</th>
      <th>quantity_sold</th>
      <th>total_sales</th>
      <th>avg_unit_price</th>
      <th>quantity_rank</th>
      <th>sales_rank</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>39</th>
      <td>스포츠 상품 041</td>
      <td>스포츠</td>
      <td>35</td>
      <td>5705000</td>
      <td>163000.0</td>
      <td>1</td>
      <td>1</td>
    </tr>
    <tr>
      <th>10</th>
      <td>패션 상품 011</td>
      <td>패션</td>
      <td>31</td>
      <td>3565000</td>
      <td>115000.0</td>
      <td>2</td>
      <td>8</td>
    </tr>
    <tr>
      <th>74</th>
      <td>전자기기 상품 077</td>
      <td>전자기기</td>
      <td>31</td>
      <td>2201000</td>
      <td>71000.0</td>
      <td>2</td>
      <td>22</td>
    </tr>
    <tr>
      <th>86</th>
      <td>생활용품 상품 089</td>
      <td>생활용품</td>
      <td>30</td>
      <td>3090000</td>
      <td>103000.0</td>
      <td>4</td>
      <td>10</td>
    </tr>
    <tr>
      <th>46</th>
      <td>식품 상품 048</td>
      <td>식품</td>
      <td>29</td>
      <td>1943000</td>
      <td>67000.0</td>
      <td>5</td>
      <td>27</td>
    </tr>
    <tr>
      <th>20</th>
      <td>생활용품 상품 022</td>
      <td>생활용품</td>
      <td>29</td>
      <td>3248000</td>
      <td>112000.0</td>
      <td>5</td>
      <td>9</td>
    </tr>
    <tr>
      <th>79</th>
      <td>생활용품 상품 082</td>
      <td>생활용품</td>
      <td>28</td>
      <td>2100000</td>
      <td>75000.0</td>
      <td>7</td>
      <td>25</td>
    </tr>
    <tr>
      <th>85</th>
      <td>생활용품 상품 088</td>
      <td>생활용품</td>
      <td>27</td>
      <td>1161000</td>
      <td>43000.0</td>
      <td>8</td>
      <td>54</td>
    </tr>
  </tbody>
</table>
</div>



```python
# 두 순위가 실제로 얼마나 일치하는가
rank_corr = compare["quantity_rank"].corr(compare["sales_rank"], method="spearman")
print("판매량 순위와 매출 순위의 스피어만 상관계수:", round(rank_corr, 3))
print("평균 판매 단가 범위:",
      f'{compare["avg_unit_price"].min():,.0f} ~ {compare["avg_unit_price"].max():,.0f}')
print()

print("=== 많이 팔렸지만 매출 순위는 낮은 상품 (rank_gap이 가장 작음) ===")
display(compare.nsmallest(5, "rank_gap")[view_cols])
print("=== 적게 팔렸지만 매출 순위는 높은 상품 (rank_gap이 가장 큼) ===")
display(compare.nlargest(5, "rank_gap")[view_cols])
```

    판매량 순위와 매출 순위의 스피어만 상관계수: 0.58
    평균 판매 단가 범위: 5,000 ~ 200,000
    
    === 많이 팔렸지만 매출 순위는 낮은 상품 (rank_gap이 가장 작음) ===
    


<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>product_name</th>
      <th>category</th>
      <th>quantity_sold</th>
      <th>total_sales</th>
      <th>avg_unit_price</th>
      <th>quantity_rank</th>
      <th>sales_rank</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>45</th>
      <td>생활용품 상품 047</td>
      <td>생활용품</td>
      <td>21</td>
      <td>231000</td>
      <td>11000.0</td>
      <td>22</td>
      <td>90</td>
    </tr>
    <tr>
      <th>5</th>
      <td>전자기기 상품 006</td>
      <td>전자기기</td>
      <td>18</td>
      <td>90000</td>
      <td>5000.0</td>
      <td>33</td>
      <td>97</td>
    </tr>
    <tr>
      <th>40</th>
      <td>패션 상품 042</td>
      <td>패션</td>
      <td>22</td>
      <td>616000</td>
      <td>28000.0</td>
      <td>17</td>
      <td>78</td>
    </tr>
    <tr>
      <th>48</th>
      <td>뷰티 상품 050</td>
      <td>뷰티</td>
      <td>20</td>
      <td>460000</td>
      <td>23000.0</td>
      <td>26</td>
      <td>84</td>
    </tr>
    <tr>
      <th>26</th>
      <td>뷰티 상품 028</td>
      <td>뷰티</td>
      <td>16</td>
      <td>80000</td>
      <td>5000.0</td>
      <td>42</td>
      <td>98</td>
    </tr>
  </tbody>
</table>
</div>


    === 적게 팔렸지만 매출 순위는 높은 상품 (rank_gap이 가장 큼) ===
    


<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>product_name</th>
      <th>category</th>
      <th>quantity_sold</th>
      <th>total_sales</th>
      <th>avg_unit_price</th>
      <th>quantity_rank</th>
      <th>sales_rank</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>96</th>
      <td>뷰티 상품 099</td>
      <td>뷰티</td>
      <td>11</td>
      <td>2200000</td>
      <td>200000.0</td>
      <td>60</td>
      <td>23</td>
    </tr>
    <tr>
      <th>35</th>
      <td>뷰티 상품 037</td>
      <td>뷰티</td>
      <td>10</td>
      <td>1930000</td>
      <td>193000.0</td>
      <td>63</td>
      <td>28</td>
    </tr>
    <tr>
      <th>56</th>
      <td>식품 상품 058</td>
      <td>식품</td>
      <td>14</td>
      <td>2758000</td>
      <td>197000.0</td>
      <td>48</td>
      <td>14</td>
    </tr>
    <tr>
      <th>22</th>
      <td>스포츠 상품 024</td>
      <td>스포츠</td>
      <td>9</td>
      <td>1764000</td>
      <td>196000.0</td>
      <td>69</td>
      <td>37</td>
    </tr>
    <tr>
      <th>36</th>
      <td>생활용품 상품 038</td>
      <td>생활용품</td>
      <td>13</td>
      <td>2249000</td>
      <td>173000.0</td>
      <td>52</td>
      <td>20</td>
    </tr>
  </tbody>
</table>
</div>



```python
# 주문 건수 기준 1위도 함께 확인한다
top_order = product_sales.loc[product_sales["order_count"].idxmax()]
print("주문 건수 1위:", top_order["product_name"],
      "| 주문", top_order["order_count"],
      "| 수량", top_order["quantity_sold"],
      "| 매출", f'{top_order["total_sales"]:,}')
print()

# 기준을 바꾸면 추천 목록이 얼마나 달라지는가
for n in (10, 20):
    by_q = set(product_sales.nlargest(n, "quantity_sold")["product_id"])
    by_s = set(product_sales.nlargest(n, "total_sales")["product_id"])
    overlap = len(by_q & by_s)
    print(f"상위 {n}개 목록: 겹침 {overlap}개 / 다름 {n - overlap}개")
```

    주문 건수 1위: 스포츠 상품 041 | 주문 12 | 수량 35 | 매출 5,705,000
    
    상위 10개 목록: 겹침 5개 / 다름 5개
    상위 20개 목록: 겹침 10개 / 다름 10개
    

## 과제 2 설명

### 판매량 1위와 매출 1위가 같은가?

**1위는 같다.** `스포츠 상품 041`이 수량 35개로 판매량 1위이면서 매출 5,705,000원으로 매출 1위다.
주문 건수 12건으로 그것도 1위라, 세 기준이 모두 같은 상품을 가리킨다.
수량이 1위인 데다 단가도 163,000원으로 높은 편이어서 두 요인이 같은 방향으로 작용했다.

그런데 **1위가 같다고 두 순위가 같은 것은 아니다.** 2위부터 급격히 갈린다.

| 상품 | 판매량 순위 | 매출 순위 | 평균 단가 |
| --- | --- | --- | --- |
| 식품 상품 012 | 13위 | **2위** | 175,000원 |
| 전자기기 상품 077 | 2위 | **22위** | 71,000원 |
| 생활용품 상품 088 | 8위 | **54위** | 43,000원 |

두 순위의 스피어만 상관계수는 약 0.58이다. 같은 방향으로 움직이기는 하지만
순위를 바꿔 써도 되는 수준과는 거리가 멀다.

### 다르다면 판매 단가와 수량 중 어떤 요인이 영향을 주었는가?

**단가다.** 평균 판매 단가가 5,000원에서 200,000원까지 40배 차이가 나는 반면,
판매 수량은 상위권에서 35개와 27개처럼 기껏해야 몇 배 안에서 움직인다.
곱셈의 두 항 중 변동 폭이 훨씬 큰 쪽이 결과를 지배하므로, 매출 순위는 사실상 단가가 결정한다.

`식품 상품 012`가 그 증거다. 25개밖에 안 팔렸는데 단가가 175,000원이라 매출 2위에 올랐다.
반대로 `생활용품 상품 088`은 27개나 팔렸지만 단가가 43,000원이라 매출은 54위로 밀렸다.
개수로는 후자가 더 많이 팔렸는데 매출 기여는 전자가 네 배 가까이 크다.

### "가장 잘 팔린 상품"이라는 표현이 왜 모호한가?

"잘 팔렸다"가 수량인지 금액인지 주문 건수인지를 말하지 않기 때문이다.

이번 데이터에서는 마침 1위가 세 기준 모두 같은 상품이라 문제가 드러나지 않는다.
그래서 오히려 위험하다. **1위만 보고 "기준이 달라도 결과는 같더라"고 결론 내리면
2위 이하에서 순위가 완전히 뒤집히는 것을 놓친다.**

"잘 팔리는 상품 10개를 더 들여오자"는 결정을 할 때, 수량 기준 상위 10개와
매출 기준 상위 10개는 **5개만 겹치고 5개가 다르다** (상위 20개로 넓혀도 절반만 겹친다).
한 상품의 1위 여부가 아니라 **목록 전체가 달라지는 것**이 실제 문제다.

### 재고 관리와 매출 분석에서 어떤 순위를 사용해야 하는가?

- **재고 관리**: `quantity_sold`. 창고에서 빠져나가는 개수가 발주량을 결정하기 때문이다.
- **매출 분석**: `total_sales`. 회사에 들어오는 금액이 기준이기 때문이다.
- **고객 인기도**: `order_count`. 몇 건의 주문에 담겼는지를 보여준다.
  한 사람이 10개를 산 것과 열 사람이 하나씩 산 것은 수량이 같아도 의미가 다르다.

---

# 과제 3. 잘못된 병합 만들고 진단하기

상품 데이터의 첫 행을 복제해 오른쪽 키를 일부러 중복시킨 뒤, `validate`가 이를 잡아내는지 확인한다.


```python
products_bad = pd.concat([products, products.head(1)], ignore_index=True)

print("정상 products 행 수:", len(products))
print("복제 products_bad 행 수:", len(products_bad))
print("products_bad의 product_id 중복 개수:", products_bad["product_id"].duplicated().sum())
display(
    products_bad[products_bad["product_id"].duplicated(keep=False)].sort_values("product_id")
)
```

    정상 products 행 수: 100
    복제 products_bad 행 수: 101
    products_bad의 product_id 중복 개수: 1
    


<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>product_id</th>
      <th>product_name</th>
      <th>category</th>
      <th>price</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>전자기기 상품 001</td>
      <td>전자기기</td>
      <td>160000</td>
    </tr>
    <tr>
      <th>100</th>
      <td>1</td>
      <td>전자기기 상품 001</td>
      <td>전자기기</td>
      <td>160000</td>
    </tr>
  </tbody>
</table>
</div>



```python
# validate를 걸면 병합 자체가 거부된다
try:
    order_items_work.merge(
        products_bad,
        on="product_id",
        how="left",
        validate="many_to_one",
    )
except Exception as e:
    print(type(e).__name__)
    print(e)
```

    MergeError
    Merge keys are not unique in right dataset; not a many-to-one merge
    
    Duplicates in right:
      product_id
              1 ...
    


```python
# validate를 빼면 오류 없이 실행되지만 행이 늘어난다
bad_merged = order_items_work.merge(products_bad, on="product_id", how="left")

print("병합 전 행 수:", len(order_items_work))
print("병합 후 행 수:", len(bad_merged))
print("증가한 행 수:", len(bad_merged) - len(order_items_work))
```

    병합 전 행 수: 764
    병합 후 행 수: 768
    증가한 행 수: 4
    


```python
# 부풀려진 금액을 확인한다
normal_total = order_items_work["line_total"].sum()
bad_total = bad_merged["line_total"].sum()

print("정상 합계:", f"{normal_total:,}")
print("잘못된 합계:", f"{bad_total:,}")
print("부풀려진 금액:", f"{bad_total - normal_total:,}")
print("부풀려진 비율:", f"{(bad_total / normal_total - 1) * 100:.2f}%")

dup_id = products_bad.loc[products_bad["product_id"].duplicated(), "product_id"].iloc[0]
affected = (order_items_work["product_id"] == dup_id).sum()
print()
print("중복된 product_id:", dup_id)
print("영향을 받은 주문상세 행 수:", affected)
```

    정상 합계: 255,610,000
    잘못된 합계: 257,690,000
    부풀려진 금액: 2,080,000
    부풀려진 비율: 0.81%
    
    중복된 product_id: 1
    영향을 받은 주문상세 행 수: 4
    

## 과제 3 기록

### 오류 메시지

```
MergeError: Merge keys are not unique in right dataset; not a many-to-one merge
```

오른쪽 데이터의 연결 키가 고유하지 않아 다대일 관계가 성립하지 않는다는 뜻이다.

### 오른쪽 키 중복 개수

1건이다. `products.head(1)`을 덧붙였으므로 `product_id` 하나가 두 번 등장한다.

### validate를 제거했을 때 병합 후 행 수

위 셀 출력대로 행 수가 늘어난다. 중복된 `product_id`를 참조하던 주문상세 행이
오른쪽의 두 행과 각각 연결되면서 그만큼 복제되기 때문이다.

### 왜 매출 합계가 부풀려질 수 있는가?

복제된 행이 가진 `line_total`이 합계에 **두 번 더해지기 때문이다.**
금액 컬럼은 왼쪽 데이터에서 왔고 이 병합은 상품 이름만 붙이려던 것이었는데,
키가 중복된 탓에 금액 행 자체가 늘어났다. 상품 정보를 붙이는 작업이 매출을 바꿔 놓은 것이고,
코드는 오류 없이 실행되므로 눈치채기 어렵다.

### 어떤 검증이 문제를 조기에 발견했는가?

`validate="many_to_one"`이 병합 시점에 바로 막았다.
이것을 쓰지 않았다면 **병합 전후 행 수 비교**가 다음 방어선이고,
그다음이 **원본 합계와 요약 합계 대조**다. 세 가지 모두 통과해야 병합을 신뢰할 수 있다.

---

# 과제 4. 결과 검증 보고서


```python
def check_merge_result(*, name, left_rows, merged, indicator_column):
    print(f"[{name}]")
    print("병합 전 행 수:", left_rows)
    print("병합 후 행 수:", len(merged))
    print("행 수 유지:", left_rows == len(merged))
    print(merged[indicator_column].value_counts(dropna=False).to_string())
    print()


def check_total(*, name, source_total, summary_total):
    print(f"[{name}]")
    print("원본 합계:", f"{source_total:,}")
    print("요약 합계:", f"{summary_total:,}")
    print("차이:", source_total - summary_total)
    print()


check_merge_result(
    name="주문상세-주문",
    left_rows=len(order_items_work),
    merged=order_sales,
    indicator_column="order_match",
)
check_merge_result(
    name="완료주문상세-상품",
    left_rows=len(completed_sales),
    merged=completed_items,
    indicator_column="product_match",
)
check_total(
    name="결제수단별 매출",
    source_total=completed_items["line_total"].sum(),
    summary_total=payment_sales["total_sales"].sum(),
)
```

    [주문상세-주문]
    병합 전 행 수: 764
    병합 후 행 수: 764
    행 수 유지: True
    order_match
    both          764
    left_only       0
    right_only      0
    
    [완료주문상세-상품]
    병합 전 행 수: 474
    병합 후 행 수: 474
    행 수 유지: True
    product_match
    both          474
    left_only       0
    right_only      0
    
    [결제수단별 매출]
    원본 합계: 148,990,000
    요약 합계: 148,990,000
    차이: 0
    
    


```python
# 예상하지 않은 _x, _y 컬럼이 생기지 않았는지 확인
suffix_cols = [c for c in completed_items.columns if c.endswith("_x") or c.endswith("_y")]
print("접미사 컬럼:", suffix_cols if suffix_cols else "없음")
print()
print("completed_items 컬럼:", completed_items.columns.tolist())
```

    접미사 컬럼: 없음
    
    completed_items 컬럼: ['order_item_id', 'order_id', 'product_id', 'quantity', 'unit_price', 'line_total', 'customer_id', 'order_date', 'payment_method', 'order_status', 'order_match', 'product_name', 'category', 'product_match']
    

## Chapter 04 분석 검증 보고서

### 분석 질문

완료 주문 기준 결제 수단별 매출은 얼마인가?

### 데이터와 분석 범위

- 사용 데이터: `order_items`(764행), `orders`(300행), `products`(100행)
- 분석 범위: `order_status == "completed"`
- 취소 64건, 환불 52건은 매출 집계에서 제외했다

### 파생 컬럼 계산식

`line_total = quantity × unit_price`

상품 마스터의 `price`가 아니라 주문 시점에 기록된 `unit_price`를 사용했다.
두 값은 다를 수 있고, 실제로 판매된 금액은 후자다.

### 병합 관계

| 병합 | 키 | 관계 수 |
| --- | --- | --- |
| order_items → orders | `order_id` | many_to_one |
| completed_sales → products | `product_id` | many_to_one |

### 병합 전후 행 수

위 검증 셀 출력대로 두 병합 모두 행 수가 유지되었다.
다대일 병합에서 행 수가 늘었다면 오른쪽 키 중복을 의심해야 한다.

### 미매칭 건수

두 병합 모두 `indicator` 결과가 전부 `both`였다. 연결되지 않은 행은 없다.

### 주문 수 계산 방식

`nunique()`로 고유 `order_id` 개수를 셌다.
주문상세 행 수를 쓰면 한 주문에 담긴 상품 개수만큼 부풀려진다.

### 그룹바이 기준

`payment_method`. 따라서 결과 표의 한 행은 결제 수단 하나다.

### 결과 한 행의 의미

`card` 행은 카드로 결제된 완료 주문 전체의 매출, 주문 수, 고객 수, 판매 수량이다.
고객 수는 결제 수단별로 세었으므로 **모든 행의 고객 수를 더해도 전체 고객 수가 되지 않는다.**
한 고객이 여러 결제 수단을 썼다면 여러 행에서 각각 세어지기 때문이다.

### 원본 합계와 요약 합계 비교

`check_total` 출력에서 차이가 0이었다. 그룹 집계 과정에서 누락된 행이 없다는 뜻이다.

### 저장 결과 확인

아래 저장 셀에서 파일 존재 여부와 크기를 출력하고, 다시 읽어 행 수와 한글 표시를 확인했다.

### LLM 사용 내용

pandas 코드 초안을 요청할 때 교재 64번의 프롬프트 형식을 따랐다.
DataFrame 이름, 한 행의 의미, 실제 컬럼명, 키 관계, 분석 범위, 검증 요구사항을 함께 전달했다.
고객 이름과 이메일 같은 개인정보, 원본 주문 행 전체는 제공하지 않았다.

### LLM 코드에서 수정한 부분

- 주문 수를 `count()`로 세던 것을 `nunique()`로 바꿨다
- `price`(상품 마스터 가격)를 쓰던 것을 `unit_price`(실제 판매 단가)로 바꿨다
- `merge`에 `validate`와 `indicator`가 없던 것을 추가했다
- 필터 없이 전체 합계를 매출이라 부르던 것을 완료 주문으로 한정했다

### 결과 해석 시 주의점

- 이 매출은 **완료 주문 기준**이며 취소·환불을 포함하지 않는다
- 결제 수단별 고객 수는 중복 계산될 수 있어 합산하면 안 된다
- 데이터 기간은 2025-09-11부터 2026-09-10까지이고, 이 범위 밖으로 일반화할 수 없다
- 샘플 데이터이므로 실제 쇼핑몰의 경향으로 해석하지 않는다

---

# 과제 5. 고객 개인정보 최소화


```python
customer_sales = (
    completed_items
    .groupby("customer_id", as_index=False)
    .agg(
        total_sales=("line_total", "sum"),
        order_count=("order_id", "nunique"),
        quantity_sold=("quantity", "sum"),
    )
)

# 이름을 제외하고 필요한 속성만 연결한다
customer_attributes = customers[["customer_id", "gender", "age", "city"]].copy()

customer_sales_detail = (
    customer_sales
    .merge(
        customer_attributes,
        on="customer_id",
        how="left",
        validate="one_to_one",
        indicator="customer_match",
    )
    .sort_values("total_sales", ascending=False)
)

print(customer_sales_detail["customer_match"].value_counts(dropna=False))
print()

# 과제가 지정한 컬럼만 남긴다
customer_sales_min = customer_sales_detail[[
    "customer_id", "gender", "age", "city",
    "total_sales", "order_count", "quantity_sold",
]]
display(customer_sales_min.head(10))
print("포함된 컬럼:", customer_sales_min.columns.tolist())
print("name 포함 여부:", "name" in customer_sales_min.columns)
```

    customer_match
    both          100
    left_only       0
    right_only      0
    Name: count, dtype: int64
    
    


<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>customer_id</th>
      <th>gender</th>
      <th>age</th>
      <th>city</th>
      <th>total_sales</th>
      <th>order_count</th>
      <th>quantity_sold</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>76</th>
      <td>117</td>
      <td>F</td>
      <td>65</td>
      <td>성남</td>
      <td>4100000</td>
      <td>5</td>
      <td>48</td>
    </tr>
    <tr>
      <th>62</th>
      <td>102</td>
      <td>M</td>
      <td>60</td>
      <td>고양</td>
      <td>3996000</td>
      <td>4</td>
      <td>35</td>
    </tr>
    <tr>
      <th>51</th>
      <td>83</td>
      <td>F</td>
      <td>22</td>
      <td>수원</td>
      <td>3880000</td>
      <td>4</td>
      <td>39</td>
    </tr>
    <tr>
      <th>21</th>
      <td>30</td>
      <td>F</td>
      <td>32</td>
      <td>서울</td>
      <td>3590000</td>
      <td>5</td>
      <td>32</td>
    </tr>
    <tr>
      <th>29</th>
      <td>40</td>
      <td>M</td>
      <td>23</td>
      <td>서울</td>
      <td>3523000</td>
      <td>4</td>
      <td>27</td>
    </tr>
    <tr>
      <th>13</th>
      <td>20</td>
      <td>F</td>
      <td>20</td>
      <td>인천</td>
      <td>3191000</td>
      <td>2</td>
      <td>25</td>
    </tr>
    <tr>
      <th>0</th>
      <td>3</td>
      <td>F</td>
      <td>61</td>
      <td>성남</td>
      <td>3178000</td>
      <td>2</td>
      <td>26</td>
    </tr>
    <tr>
      <th>70</th>
      <td>111</td>
      <td>F</td>
      <td>41</td>
      <td>광주</td>
      <td>3153000</td>
      <td>3</td>
      <td>38</td>
    </tr>
    <tr>
      <th>42</th>
      <td>66</td>
      <td>F</td>
      <td>39</td>
      <td>서울</td>
      <td>3093000</td>
      <td>4</td>
      <td>30</td>
    </tr>
    <tr>
      <th>97</th>
      <td>147</td>
      <td>M</td>
      <td>19</td>
      <td>부산</td>
      <td>2990000</td>
      <td>2</td>
      <td>21</td>
    </tr>
  </tbody>
</table>
</div>


    포함된 컬럼: ['customer_id', 'gender', 'age', 'city', 'total_sales', 'order_count', 'quantity_sold']
    name 포함 여부: False
    

## 고객 이름을 제외한 이유

`name`은 **개인을 직접 식별하는 값**이다. 그런데 이번 분석의 질문은
"어떤 고객이 얼마를 샀는가"가 아니라 "구매 금액이 성별·나이·지역에 따라 어떻게 다른가"다.
이 질문에 답하는 데 이름은 아무 기여도 하지 않는다.

개인정보는 분석에 필요해서 넣는 것이지, 원본에 있으니 따라오는 것이 아니다.
같은 고객을 여러 표에서 이어 붙이려면 `customer_id`만 있으면 충분하고,
이 값은 그 자체로는 누구인지 알려주지 않는 대체 식별자다.

결과 CSV는 저장되어 공유되거나 다른 사람에게 전달될 수 있다.
이름이 들어간 순간 그 파일은 개인정보 파일이 되고, 보관과 폐기에 다른 기준이 적용된다.
처음부터 넣지 않는 것이 가장 확실한 통제다.

## 외부 LLM에 제공해도 되는 정보 범위

### 제공해도 되는 것

- DataFrame 이름과 각 데이터에서 한 행이 무엇을 의미하는지
- 실제 컬럼명과 dtype 요약
- 주요 키와 관계 수 (`many_to_one` 등)
- `order_status`처럼 조건에 쓸 범주값의 실제 표기
- 분석 범위와 원하는 결과 컬럼, 검증 기준
- 구조를 보여주기 위한 **가상의** 예시 행

### 제공하면 안 되는 것

- 고객 이름, 이메일, 전화번호, 주소
- 원본 데이터 행 전체 또는 CSV 파일 자체
- 데이터베이스 접속 정보와 API 키
- 식별자와 속성이 함께 붙은 실제 레코드

핵심은 **구조는 주되 내용은 주지 않는다**는 것이다.
LLM에게 필요한 것은 "이 표에 어떤 컬럼이 있고 어떤 관계인가"이지
"3번 고객이 누구인가"가 아니다. 구조만으로도 정확한 pandas 코드를 받을 수 있다.

---

# 결과 저장


```python
output_dir = project_root / "reports" / "chapter04"
output_dir.mkdir(parents=True, exist_ok=True)

outputs = {
    "payment_sales.csv": payment_sales,
    "product_sales.csv": product_sales.sort_values("total_sales", ascending=False),
    "customer_sales_min.csv": customer_sales_min,
}
for file_name, df in outputs.items():
    output_path = output_dir / file_name
    df.to_csv(output_path, index=False, encoding="utf-8-sig")
    print(file_name, output_path.exists(), output_path.stat().st_size, "bytes")
```

    payment_sales.csv True 191 bytes
    product_sales.csv True 4619 bytes
    customer_sales_min.csv True 2933 bytes
    


```python
# 저장 후 다시 읽어 컬럼, 행 수, 한글 표시를 확인한다
saved = pd.read_csv(output_dir / "payment_sales.csv")
display(saved)
print(saved.shape, saved.columns.tolist())
```


<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>payment_method</th>
      <th>total_sales</th>
      <th>order_count</th>
      <th>customer_count</th>
      <th>quantity_sold</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>naver_pay</td>
      <td>43500000</td>
      <td>51</td>
      <td>38</td>
      <td>414</td>
    </tr>
    <tr>
      <th>1</th>
      <td>kakao_pay</td>
      <td>39342000</td>
      <td>49</td>
      <td>39</td>
      <td>385</td>
    </tr>
    <tr>
      <th>2</th>
      <td>bank_transfer</td>
      <td>34342000</td>
      <td>45</td>
      <td>38</td>
      <td>324</td>
    </tr>
    <tr>
      <th>3</th>
      <td>card</td>
      <td>31806000</td>
      <td>39</td>
      <td>33</td>
      <td>319</td>
    </tr>
  </tbody>
</table>
</div>


    (4, 5) ['payment_method', 'total_sales', 'order_count', 'customer_count', 'quantity_sold']
    

---

# LLM 코드 검증표

| 검증 항목 | 확인 내용 | 결과 |
| --- | --- | --- |
| DataFrame | 실제 변수명과 같은가? | 통과 |
| 컬럼 | 실제 컬럼만 사용하는가? | 통과 |
| 상태값 | `completed` 표기가 맞는가? | 통과 (value_counts로 확인) |
| 계산식 | `quantity × unit_price`인가? | 통과 (수작업 대조) |
| 분석 범위 | 완료 주문만 포함하는가? | 통과 |
| 주문 수 | `nunique()`를 사용하는가? | 통과 (초안은 count였고 수정함) |
| 병합 키 | 실제 관계와 맞는가? | 통과 |
| validate | `many_to_one`이 적용되었는가? | 통과 |
| indicator | 미매칭을 확인하는가? | 통과 (전부 both) |
| 행 수 | 병합 전후를 비교하는가? | 통과 |
| 합계 | 원본과 요약 합계를 비교하는가? | 통과 (차이 0) |
| 개인정보 | 원본 고객 정보를 요구하지 않는가? | 통과 (이름 제외) |

## 실행 성공과 분석 타당성은 다르다

이번 과제에서 그 차이를 가장 분명하게 보여준 것은 과제 3이다.
`validate`를 빼면 잘못된 병합이 **오류 없이 실행되고**, 매출 합계만 조용히 부풀려진다.
코드가 돌았다는 사실은 분석이 맞다는 증거가 되지 못한다.

그래서 이번 노트북에서는 모든 병합에 행 수 비교와 `indicator`를 붙였고,
모든 집계에 원본 합계 대조를 붙였다.
