
#!/usr/bin/env python
# coding: utf-8

# In[5]:


# ==========================================================
# 每日早盤 AI 財經與潛力公司焦點速遞（多模型自動容錯版）
# ==========================================================

from datetime import datetime
import os
import time
from google import genai
import requests
import schedule

# ==========================================================
# Gemini 初始化
# ==========================================================

API_KEY = os.getenv("GEMINI_API_KEY")
if not API_KEY:
  raise RuntimeError("請設定 GEMINI_API_KEY")

client = genai.Client(api_key=API_KEY)

# 定義優先使用的模型清單（按順序嘗試）
PREFERRED_MODELS = [
    "gemini-2.5-flash",
    "gemini-flash-latest",
    "gemini-2.5-pro",
    "gemini-pro-latest",
]


# 透過 Gemini 產出市場報告（支援模型輪替與自動重試）
def generate_market_report():
  prompt = """
    請幫我彙整今日（台股、美股、虛擬貨幣）的焦點新聞，以及 2-3 家具備潛力的科技／生技／AI 成長型公司新聞。
    【重要限制】：請將總字數控制在 1500 字以內，精簡扼要，避免過長。
    請用繁體中文輸出，並以清晰的 Markdown 格式排版，包含以下區塊：
    1. 🇹🇼 台股焦點與趨勢
    2. 🇺🇸 美股焦點與總經動態
    3. 🪙 虛擬貨幣（Crypto）市場動態
    4. 🚀 潛力公司追蹤（具備高成長潛力之標的）
    """

  # 輪流嘗試清單中的每一個模型
  for model_name in PREFERRED_MODELS:
    max_retries = 2
    for attempt in range(1, max_retries + 1):
      try:
        print(
            f"[{datetime.now()}] 嘗試使用模型 [{model_name}]"
            f" 生成報告（第 {attempt} 次）..."
        )
        response = client.models.generate_content(
            model=model_name,
            contents=prompt,
            config={
                "system_instruction": (
                    "你是一位專業的金融分析師與資深AI助理，擅長統整全球市場動態與發掘潛力成長股。"
                )
            },
        )
        print(f"[{datetime.now()}] 模型 [{model_name}] 生成成功！")
        return response.text
      except Exception as e:
        print(f"[{datetime.now()}] 模型 [{model_name}] 發生錯誤: {str(e)}")
        if "503" in str(e) or "UNAVAILABLE" in str(e):
          print(f"[{datetime.now()}] 遇到伺服器繁忙 (503)，短暫等待後重試...")
          time.sleep(3)
        else:
          # 若非 503 錯誤（例如語法或其他問題），直接跳過該模型換下一個
          break

  return "生成市場報告時發生錯誤：所有備用模型目前皆處於高負載或無法使用狀態。"


# 發送到 Discord 的函式（含字數防護）
def send_to_discord(content):
  webhook_url = "https://discord.com/api/webhooks/1565912614721298462/aCWjwU6Zu3NMnrHSY2i0NrZtLxp5v9n9odmt4mbyBWrzBKY5mQlb1Dk0ueFtL8xU8lyp"

  header_text = (
      f"📊 **【每日早盤 AI 財經與潛力公司焦點速遞】** -"
      f" {datetime.now().strftime('%Y-%m-%d')}\n\n"
  )

  max_content_length = 1900 - len(header_text)
  if len(content) > max_content_length:
    content = content[:max_content_length] + "\n\n...(內容過長已自動截斷)"

  payload = {"content": header_text + content}

  response = requests.post(webhook_url, json=payload)
  if response.status_code == 204:
    print(f"[{datetime.now()}] 成功推送到 Discord！")
  else:
    print(f"推送到 Discord 失敗: {response.text}")


# 定時執行的主工作包裝
def job():
  print(f"[{datetime.now()}] 開始執行每日財經報告生成與推播...")
  report = generate_market_report()
  send_to_discord(report)


if __name__ == "__main__":
  print(
      "已啟動 AI 財經新聞背景排程服務（含多模型容錯機制）：將在每天上午 09:00"
      " 自動發送報告到 Discord。"
  )

  # 若你在啟動時想順便立刻測試一次，可將下方這一行的註解(#)拿掉
  # job()

  # 設定每天上午 09:00 執行
  schedule.every().day.at("09:00").do(job)

  # 保持程式持續在背景運行以檢查排程
  while True:
    schedule.run_pending()
    time.sleep(1)


# In[ ]:




