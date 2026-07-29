import os
import time
import requests
import feedparser
import urllib.parse
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from google import genai

# ==========================================
# 1. 환경변수 및 설정
# ==========================================
NAVER_CLIENT_ID = os.environ.get("NAVER_CLIENT_ID")
NAVER_CLIENT_SECRET = os.environ.get("NAVER_CLIENT_SECRET")
GEMINI_API_KEY = os.environ.get("GEMINI_API_KEY")
SENDER_EMAIL = os.environ.get("SENDER_EMAIL")
SENDER_PASSWORD = os.environ.get("SENDER_PASSWORD")
RECEIVER_EMAIL = os.environ.get("RECEIVER_EMAIL")

# 최신 google-genai 클라이언트 초기화
client = genai.Client(api_key=GEMINI_API_KEY)

# ==========================================
# 2. 네이버 뉴스 검색 (국내: HMR, 밀키트, 가공육)
# ==========================================
def fetch_naver_news(keyword, display=7):
    enc_text = urllib.parse.quote(keyword)
    url = f"https://openapi.naver.com/v1/search/news.json?query={enc_text}&display={display}&sort=sim"
    headers = {
        "X-Naver-Client-Id": NAVER_CLIENT_ID,
        "X-Naver-Client-Secret": NAVER_CLIENT_SECRET
    }
    response = requests.get(url, headers=headers)
    if response.status_code == 200:
        items = response.json().get('items', [])
        return [{
            "title": item['title'].replace('<b>','').replace('</b>','').replace('&quot;', '"'),
            "link": item['link'],
            "desc": item['description'].replace('<b>','').replace('</b>','').replace('&quot;', '"')
        } for item in items]
    return []

# ==========================================
# 3. Google News RSS 검색 (해외: Ready Meals, Processed Meat)
# ==========================================
def fetch_google_news(keyword, num_items=7):
    encoded_query = urllib.parse.quote(keyword)
    rss_url = f"https://news.google.com/rss/search?q={encoded_query}&hl=en-US&gl=US&ceid=US:en"
    feed = feedparser.parse(rss_url)
    news_list = []
    for entry in feed.entries[:num_items]:
        news_list.append({
            "title": entry.title,
            "link": entry.link,
            "desc": entry.get('summary', '')
        })
    return news_list

# ==========================================
# 4. Gemini AI 분석 및 리포트 작성
# ==========================================
def generate_report(kr_news, en_news):
    prompt = f"""
너는 식품 산업 중 **HMR(가정간편식), 밀키트, RMR(레스토랑 간편식) 및 가공육(육가공, 햄, 소시지, 대체육 등)** 분야를 전담하는 수석 시장분석가야.
전달받은 최신 국내외 뉴스 기사들을 바탕으로, 관련 제품 기획자 및 마케터를 위한 **[HMR & 가공육 산업 일간 브리핑]**을 작성해줘.

[수집된 데이터]
■ 국내 뉴스 (네이버):
{kr_news}

■ 해외 뉴스 (Google News EN):
{en_news}

---

[작성 가이드라인]
1. HMR(간편식), 밀키트, 가공육, 육가공 기술, 대체육 및 관련 유통/소비 트렌드에 중점을 두어 분석해.
2. 기사별로 핵심 내용 요약뿐만 아니라, **[인사이트 / 제품 기획 시사점]**을 반드시 포함해줘.
3. 기사 제목은 원본 링크로 연결될 수 있도록 Markdown 링크 형식 `[제목](링크)`으로 작성해줘.
4. 전문적이고 간결한 비즈니스 리포트 톤앤매너를 유지해.

[리포트 출력 양식]

# 🍱 HMR & 가공육 산업 일간 동향 브리핑

## 🎯 1. 오늘의 핵심 요약 (Executive Summary)
- (HMR 및 가공육 시장의 오늘 자 가장 중요한 트렌드 3가지 정리)

## 🇰🇷 2. 국내 HMR & 가공육 시장 동향
- **[기사 제목]**(기사 링크)
  - 📌 **주요 내용:** 1-2줄 핵심 요약
  - 💡 **시사점:** 신제품 개발, 타겟 고객층, 포장/제조 기술 관점의 시사점

## 🌐 3. 글로벌 Ready Meals & Meat Industry 동향
- **[기사 제목 (한국어 번역)]**(기사 링크)
  - 📌 **주요 내용:** 1-2줄 한국어 핵심 요약
  - 💡 **시사점:** 글로벌 트렌드의 국내 적용 가능성 및 육가공/HMR 기술 동향

## 📈 4. HMR & 가공육 제품/마케팅 전략 제언
- 최근 소비자 니즈에 맞춘 제품 개발 및 판매 전략 2가지 제안
"""
    # 429 Rate Limit 대비 최대 3회 재시도 (30초 대기)
    max_retries = 3
    for attempt in range(1, max_retries + 1):
        try:
            print(f"[gemini-2.0-flash] 리포트 생성 시도 ({attempt}/{max_retries})...")
            response = client.models.generate_content(
                model='gemini-2.0-flash',
                contents=prompt
            )
            return response.text
        except Exception as e:
            print(f"시도 {attempt} 실패: {e}")
            if attempt < max_retries:
                print("30초 대기 후 재시도합니다...")
                time.sleep(30)
            
    raise Exception("Gemini API 호출 실패 (새 API 키 발급 및 쿼터 확인이 필요합니다).")

# ==========================================
# 5. 이메일 전송 함수
# ==========================================
def send_email(content):
    msg = MIMEMultipart()
    msg['Subject'] = "🍱 [매일 리포트] HMR & 가공육 산업 최신 뉴스 동향 분석"
    msg['From'] = SENDER_EMAIL
    msg['To'] = RECEIVER_EMAIL
    
    msg.attach(MIMEText(content, 'plain', 'utf-8'))
    
    with smtplib.SMTP_SSL('smtp.gmail.com', 465) as server:
        server.login(SENDER_EMAIL, SENDER_PASSWORD)
        server.send_message(msg)

# ==========================================
# 6. 메인 실행부
# ==========================================
if __name__ == "__main__":
    kr_query = "HMR OR 밀키트 OR 간편식 OR 가공육 OR 육가공"
    en_query = '"ready meals" OR "prepared foods" OR "processed meat" OR "delicatessen"'
    
    kr_articles = fetch_naver_news(kr_query, display=7)
    en_articles = fetch_google_news(en_query, num_items=7)
    
    report = generate_report(kr_articles, en_articles)
    send_email(report)
    print("성공적으로 HMR & 가공육 리포트를 발송했습니다!")
