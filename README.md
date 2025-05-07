# 🖥️ AWS EC2 인스턴스 성능 리포트 - Eun-yoo(은유)팀
## 🚀 목표
나는 주식을 잘 모르는 주린이다.  
그래서 어떤 주식들이 있는지 살펴보고, 그 주식이 오를지 내릴지를 오늘의 사회 정세에 따라 판단하고 싶다.  

단, 이 판단은 내가 아닌 **EC2와 GPT가 대신 한다**.
<br></br>

## ☁️ 활용한 인스턴스들
| 인스턴스 타입      | vCPU | 메모리   | 아키텍처 | CPU 종류                                                  | 특징                  |
| ------------ | ---- | ----- | ---- | ------------------------------------------------------- | ------------------- |
| `t4g.micro`  | 2    | 1 GiB | ARM  | AWS Graviton2 (ARM Neoverse N1)                         | 저비용, 에너지 효율, ARM 기반 |
| `t2.micro`   | 1    | 1 GiB | x86  | Intel Xeon E5-2676 v3 / v4 또는 AWS 커스텀 Intel CPU         | 제한적 burst 성능, 무료 티어 |
| `m5.large`   | 2    | 8 GiB | x86  | Intel Xeon Platinum 8175M / 8259CL 또는 AMD EPYC 7000 시리즈 | 안정적인 범용 성능          |
| `c6g.medium` | 1    | 2 GiB | ARM  | AWS Graviton2 (ARM Neoverse N1)                         | 컴퓨팅 최적화, ARM 기반 고성능 |

<br></br>
## 📈 주제 1: 상장된 주식 정보를 모아서 저장해보자
```python
import pandas as pd
import requests
from bs4 import BeautifulSoup
from io import StringIO
import time
import psutil
import os

# ✅ 성능 측정 시작
start_time = time.time()
process = psutil.Process(os.getpid())
cpu_times_start = process.cpu_times()

# ✅ 1. 한국거래소(KRX)에서 전체 상장 종목 리스트 가져오기
def get_krx_stock_list():
    url = "http://kind.krx.co.kr/corpgeneral/corpList.do?method=download"
    res = requests.get(url)
    res.encoding = 'euc-kr'
    data = StringIO(res.text)
    df = pd.read_html(data, header=0)[0]
    df = df[['종목코드', '회사명']]
    df['종목코드'] = df['종목코드'].apply(lambda x: f"{x:06d}")
    return df

# ✅ 2. 네이버 금융에서 종목 상세 정보 가져오기
def get_naver_stock_info(code):
    try:
        url = f"https://finance.naver.com/item/main.nhn?code={code}"
        headers = {"User-Agent": "Mozilla/5.0"}
        res = requests.get(url, headers=headers, timeout=5)
        soup = BeautifulSoup(res.text, "html.parser")

        price = soup.select_one("p.no_today span.blind").text.replace(',', '')
        market_cap = soup.select_one("em#_market_sum").text.replace(',', '').replace('조', '').replace('억원', '').strip()
        volume = soup.select("table.no_info td")[2].select_one("span.blind").text.replace(',', '')

        return {
            "code": code,
            "price": int(price),
            "market_cap(억)": float(market_cap),
            "volume": int(volume)
        }
    except Exception:
        return None

# ✅ 3. 전체 종목 크롤링 실행
krx_df = get_krx_stock_list()
results = []

print(f"📦 전체 종목 수: {len(krx_df)}개, 크롤링 시작...\n")

for idx, row in krx_df.iterrows():
    code = row["종목코드"]
    name = row["회사명"]
    info = get_naver_stock_info(code)
    if info:
        info["name"] = name
        results.append(info)

    if idx % 100 == 0:
        print(f"⏳ 진행중... {idx}/{len(krx_df)}")

    time.sleep(0.3)  # 요청 간격 조절

# ✅ 성능 측정 종료
end_time = time.time()
elapsed = end_time - start_time
avg_time = elapsed / len(krx_df)

# ✅ CPU 사용률 측정 (전체 작업 시간 대비 CPU 사용 시간 비율)
cpu_times_end = process.cpu_times()
cpu_user_time = cpu_times_end.user - cpu_times_start.user
cpu_system_time = cpu_times_end.system - cpu_times_start.system
cpu_total_time = cpu_user_time + cpu_system_time
cpu_usage_percent = (cpu_total_time / elapsed) * 100  # %

# ✅ 메모리 사용량 측정
mem = process.memory_info().rss / (1024 ** 2)  # MB

# ✅ 4. 결과 정리 및 저장
final_df = pd.DataFrame(results)
final_df = final_df[['name', 'code', 'price', 'market_cap(억)', 'volume']]

output_file = "krx_analysis_result.csv"
final_df.to_csv(output_file, index=False, encoding='utf-8-sig')

# ✅ 결과 출력
print(f"\n💾 결과 저장 완료: {output_file}")
print("\n📈 시가총액 기준 상위 10개 종목:")
print(final_df.sort_values(by="market_cap(억)", ascending=False).head(10).to_string(index=False))

# ✅ 성능 출력
print(f"\n📊 Performance Summary")
print(f"📈 Total Execution Time: {elapsed:.2f} sec")
print(f"⏱️  Avg Time per Sample: {avg_time:.4f} sec")
print(f"📦 Memory Usage: {mem:.2f} MB")
print(f"⚙️  CPU Usage (estimated over task): {cpu_usage_percent:.2f}%")
```

### 실행 과정
<img width="461" alt="1  Pandas 실행 과정" src="https://github.com/user-attachments/assets/eb8a7cb2-cb66-4cbd-bd50-83b97251cb00" />

### 실행 결과 예시
<img width="461" alt="1  Pandas 실행 결과" src="https://github.com/user-attachments/assets/1e233b8a-d11f-4221-af17-feb377691781" />

### 실행 결과 다운로드
<img width="461" alt="1  Pandas 결과 다운" src="https://github.com/user-attachments/assets/f9d6fea1-998b-40e6-8048-acc835da0d4e" />

### 그래프 비교
<table>
  <tr>
    <td align="center"><strong>총 실행 시간</strong><br><img src="https://github.com/user-attachments/assets/9470132b-08e8-4fab-b366-a3803312fb34" width="400"/></td>
    <td align="center"><strong>샘플당 평균 처리 시간</strong><br><img src="https://github.com/user-attachments/assets/ab97332e-96be-45bb-885a-2366db19f9c5" width="400"/></td>
  </tr>
  <tr>
    <td align="center"><strong>메모리 사용량</strong><br><img src="https://github.com/user-attachments/assets/d45a8a17-868b-47f6-acd8-3bd225f0616c" width="400"/></td>
    <td align="center"><strong>CPU 사용률</strong><br><img src="https://github.com/user-attachments/assets/8d2df269-52cf-4e00-86ae-6e6f3e46f13e" width="400"/></td>
  </tr>
</table>

### 🔍 왜 이렇게 나왔을까? 주식 정보는 내가 직접 읽자..
1. 병렬 처리 없이 **순차적으로 동작**
- 코드가 종목 하나씩 `requests.get()`으로 데이터를 가져오는 구조이다.
- 또한 `time.sleep(0.3)`으로 요청 간 딜레이가 고정되어 있어, CPU 성능이나 코어 수가 좋아도 **속도 개선이 어렵다**.

2. 병목 구간은 **네트워크 통신**
- KRX와 네이버 금융 서버에 요청을 보내고, 응답을 기다리는 시간이 전체 실행 시간의 대부분을 차지한다.
- 이 네트워크 지연(latency)은 인스턴스 성능과 무관하게 일정 수준 이상 개선하기 어렵다.

3. 대부분이 **I/O 중심 작업**
- 주 작업은 계산이 아니라 **HTTP 요청 + HTML 파싱**이다.
- 연산량이 적기 때문에, `m5.large`처럼 고사양 인스턴스라고 해서 속도가 크게 빨라지지 않는다.

<br></br>
## 💰 주제 2: 주식을 예측해보자
### 🧠 LSTM이란?

LSTM(Long Short-Term Memory)은 기억력이 향상된 RNN(순환 신경망) 구조로,  
셀 상태(Cell State)와 게이트(Gate) 메커니즘을 통해 "어떤 정보를 기억하고, 잊고, 출력할지"를 스스로 조절한다.

🔁 시계열 데이터에 적합한 모델로,  
주가처럼 시간에 따라 연속적으로 변화하는 데이터를 예측할 때 유용하며, 과거의 흐름이 현재에 영향을 미칠 수 있는 문제에 효과적이다.

💡 PyTorch에서는 `torch.nn.LSTM` 모듈로 쉽게 구현할 수 있다.

```python
import torch
import torch.nn as nn
import time
import psutil
import os

# ✅ LSTM 모델 정의
class SimpleLSTM(nn.Module):
    def __init__(self, input_dim, hidden_dim, output_dim):
        super(SimpleLSTM, self).__init__()
        self.lstm = nn.LSTM(input_dim, hidden_dim, batch_first=True)
        self.fc = nn.Linear(hidden_dim, output_dim)

    def forward(self, x):
        out, _ = self.lstm(x)
        return self.fc(out[:, -1, :])

# ✅ 모델 & 입력 준비
model = SimpleLSTM(input_dim=10, hidden_dim=32, output_dim=1)
model.eval()
input_data = torch.randn(1024, 30, 10)  # (batch_size, sequence_length, input_dim)

# ✅ CPU 시간 추적 시작
process = psutil.Process(os.getpid())
cpu_times_start = process.cpu_times()
start = time.time()

# ✅ 추론 수행
with torch.no_grad():
    output = model(input_data)

# ✅ 시간 측정 종료
end = time.time()
elapsed = end - start
avg_time = elapsed / input_data.shape[0]

# ✅ CPU 사용 시간 계산 (user + system)
cpu_times_end = process.cpu_times()
cpu_user_time = cpu_times_end.user - cpu_times_start.user
cpu_system_time = cpu_times_end.system - cpu_times_start.system
cpu_total_time = cpu_user_time + cpu_system_time
cpu_usage_percent = (cpu_total_time / elapsed) * 100 if elapsed > 0 else 0

# ✅ 메모리 사용량 계산
mem = process.memory_info().rss / (1024 ** 2)  # in MB

# ✅ 결과 출력
print(f"📈 LSTM Inference Time (batch 1024): {elapsed:.2f} sec")
print(f"⏱️  Avg Inference per Sample: {avg_time:.4f} sec")
print(f"📦 Memory Usage: {mem:.2f} MB")
print(f"⚙️  CPU Usage (over task): {cpu_usage_percent:.2f}%")
```

### 그래프 비교
<table>
  <tr>
    <td align="center"><strong>평균 추론 시간<br>(Batch 1024 기준)</strong><br>
      <img src="https://github.com/user-attachments/assets/42dfed58-b082-4a46-87ea-d25d7b5a54f4" width="400"/>
    </td>
    <td align="center"><strong>샘플당 평균 추론 시간</strong><br>
      <img src="https://github.com/user-attachments/assets/b292b3b9-6a18-4f48-bfe0-e48aa7d387c7" width="400"/>
    </td>
  </tr>
  <tr>
    <td align="center"><strong>평균 메모리 사용량</strong><br>
      <img src="https://github.com/user-attachments/assets/d4627c3a-2e4e-40e8-a8a3-f3265fbbd163" width="400"/>
    </td>
    <td align="center"><strong>평균 CPU 사용률</strong><br>
      <img src="https://github.com/user-attachments/assets/4cbd28d3-aafc-4b84-a27c-13d9acd4e8f8" width="400"/>
    </td>
  </tr>
</table>

### 🔍 왜 이렇게 나왔을까? 비용이 중요하면 t2.micro, 성능이 중요하면 m5.large
1. 계산량은 작지만 **벡터 연산 집중**
- LSTM은 내부적으로 `matrix multiplication`, `sigmoid`, `tanh` 등의 연산을 반복한다.
- 특히 `batch_size=1024`로 한 번에 처리하는 양이 많기 때문에, **연산 성능이 좋은 CPU에서 처리 시간이 더 짧게** 나온다.

2. ARM 기반 vs x86 기반 차이
- `t4g.micro`, `c6g.medium`은 **ARM 기반 Graviton2 프로세서**를 사용한다.
- `m5.large`, `t2.micro`는 **Intel 또는 AMD 기반의 x86 아키텍처**이다.
- PyTorch는 내부적으로 **x86 연산에 더 최적화**되어 있어, ARM에서는 상대적으로 느릴 수 있다.
- 특히 `t4g.micro`는 **작은 코어 2개**만 제공되기 때문에 연산 도중 **CPU 사용률이 198%까지 치솟는** 모습을 보였다.

3. 고사양 인스턴스의 효과
- `m5.large`는 2 vCPU와 8GB RAM을 가진 **범용 고성능 인스턴스**이다.
- 가장 빠른 처리 속도와 넉넉한 메모리 사용량을 보여주며, **연산 중심의 추론 작업에서는 성능 차이가 분명히 체감**된다.


<br></br>
## 📰 주제 3: 오늘의 뉴스를 통해 주식 시장 흐름을 파악해보자
```python
from transformers import AutoTokenizer, AutoModelForSequenceClassification, pipeline
import requests
from bs4 import BeautifulSoup
import time
import psutil
import os

# ✅ 뉴스 크롤링
rss_url = "https://www.yonhapnewstv.co.kr/browse/feed/"
res = requests.get(rss_url)
soup = BeautifulSoup(res.content, 'xml')
articles = [item.find('title').text for item in soup.find_all('item')][:100]
print(f"📥 뉴스 개수: {len(articles)}")

# ✅ 감성 분석 모델 로드
model_name = "beomi/KcELECTRA-base-v2022"
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForSequenceClassification.from_pretrained(model_name)
classifier = pipeline("sentiment-analysis", model=model, tokenizer=tokenizer)

# ✅ 성능 측정 시작
process = psutil.Process(os.getpid())
cpu_times_start = process.cpu_times()
start = time.time()

# ✅ 감정 분석 수행
results = classifier(articles)

# ✅ 성능 측정 종료
end = time.time()
elapsed = end - start
avg_time = elapsed / len(articles)

# ✅ CPU 사용률 계산
cpu_times_end = process.cpu_times()
cpu_user_time = cpu_times_end.user - cpu_times_start.user
cpu_system_time = cpu_times_end.system - cpu_times_start.system
cpu_total_time = cpu_user_time + cpu_system_time
cpu_usage_percent = (cpu_total_time / elapsed) * 100 if elapsed > 0 else 0

# ✅ 메모리 사용량
mem = process.memory_info().rss / (1024 ** 2)

# ✅ 결과 출력
print(f"\n📰 감성 분석 전체 시간: {elapsed:.2f} sec")
print(f"⏱️  뉴스 1건당 평균 처리 시간: {avg_time:.4f} sec")
print(f"📦 메모리 사용량: {mem:.2f} MB")
print(f"⚙️  CPU 사용률 (over task): {cpu_usage_percent:.2f}%")

# ✅ 예시 결과
print("\n🔍 예시 결과:")
for i in range(len(articles)):
    label = results[i]['label']
    score = results[i]['score']
    sentiment = "긍정" if label == "LABEL_1" else "부정"
    print(f"- \"{articles[i]}\" → {sentiment} ({score:.2f})")
```

### 그래프 비교
<table>
  <tr>
    <td align="center"><strong>전체 감성 분석 처리 시간</strong><br>
      <img src="https://github.com/user-attachments/assets/911db331-7974-4a64-9327-5722e8cefaf8" width="400"/>
    </td>
    <td align="center"><strong>뉴스 1건당 평균 분석 시간</strong><br>
      <img src="https://github.com/user-attachments/assets/a4dcecd6-a2b5-4c80-9339-a328f1e7d8d3" width="400"/>
    </td>
  </tr>
  <tr>
    <td align="center"><strong>평균 메모리 사용량</strong><br>
      <img src="https://github.com/user-attachments/assets/e8544e05-b999-4f96-b8b1-22cc0e2b7ae7" width="400"/>
    </td>
    <td align="center"><strong>평균 CPU 사용률</strong><br>
      <img src="https://github.com/user-attachments/assets/de3b2263-7ca6-4f35-bd69-967c33d3ddee" width="400"/>
    </td>
  </tr>
</table>

### 🔍 왜 이렇게 나왔을까? x86 기반의 CPU가 좋은 인스턴스가 유리하다.

1. 사전학습 모델 로딩 & 추론은 메모리 의존도가 높음
- `beomi/KcELECTRA`는 BERT 기반 사전학습 모델이다.
- 약 **100MB 이상의 가중치**와 **문장 단위 토큰화 처리**가 필요하며, 실행 시 **600~1200MB의 메모리를 점유**하게 된다.
- 따라서 RAM이 적은 인스턴스에서는 로딩 및 추론 시간이 더 오래 걸릴 수 있다.

2. CPU 성능이 클수록 빠른 추론 가능
- 해당 코드는 **GPU 없이, CPU만 사용하여 추론**을 수행한다.
- `m5.large`는 고성능 x86 CPU(2 vCPU)를 탑재해 **가장 빠른 시간에 모든 뉴스의 감성 분석을 마쳤다.**
- 반면, `t2.micro`는 느린 CPU이지만 프로세서 부하가 낮아 **CPU 사용률이 68% 수준**에서 동작한다.
- `t4g.micro`는 ARM 기반 인스턴스로, PyTorch 최적화가 덜 되어 있어 **CPU 사용률이 149%까지 증가**하면서도 추론 속도는 오히려 느렸다.
  
3. ARM vs x86 구조 차이
- 현재 Hugging Face Transformers 및 PyTorch는 **x86 아키텍처에서 더 안정적이고 빠르게 작동**한다.
- 따라서 ARM 인스턴스는 비용은 저렴하지만, **CPU 부하가 크고 처리 시간이 더 길어질 수 있다.**
