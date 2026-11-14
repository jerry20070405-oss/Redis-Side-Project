# 🚨圓圈點擊多模式挑戰賽 Redis 遊戲排行榜系統 (Circle Click Redis Leaderboard)

[![Python](https://img.shields.io/badge/Python-3.10%2B-blue.svg)](https://www.python.org/)
[![Streamlit](https://img.shields.io/badge/Streamlit-App-red.svg)](https://streamlit.io/)
[![Redis](https://img.shields.io/badge/Redis-In--Memory-green.svg)](https://redis.io/)

基於 **Python (Streamlit)** 與 **Redis 記憶體資料庫** 開發的高效能多模式線上遊戲排行榜系統。本專案將網頁節奏遊戲（osu! 風格 2D 圓圈極速點擊挑戰）與 NoSQL 資料庫的高階特性完美結合，提供極佳的即時互動體驗與後端效能。

---

## 🌟 核心特色與 Redis 技術亮點

本專案深入應用了 Redis 的多項進階功能，徹底發揮記憶體資料庫的高效能優勢：
1. **有序集合 (`Sorted Sets - ZSET`)**：實現各遊戲模式（經典、競速、生存）的獨立排行榜，支援毫秒級即時積分排序與個人最高分防護 (`gt=True`)。
2. **管線化 (`Pipelines`)**：將多項資料寫入指令（ZADD、PUBLISH、LPUSH、HSET）打包為單一網路請求發送，大幅減少來回延遲（RTT）。
3. **雜湊 (`Hashes`)**：用於安全管理會員帳號密碼認證，以及結構化儲存玩家個人檔案、頭銜與最佳成績。
4. **發布／訂閱與列表 (`Pub/Sub` & `Lists`)**：實現即時動態牆公告與歷史佇列快取（透過 `LTRIM` 固定保留最近動態）。

---

## 🎮 遊戲模式介紹

* **🎮 經典模式 (Classic)**：15 秒內盡可能點擊隨機生成的圓圈，挑戰最高總得分。
* **⚡ 競速模式 (Speedrun)**：計時點完固定 20 個圓圈所花費的總時間（精確至小數點後第二位），以最短時間登頂。
* **🛡️ 生存模式 (Survival)**：每個圓圈限時 **0.7 秒**，隨進度條縮減，逾時未點擊即遊戲結束，考驗極限反應力。

---

## 🛠️ 系統架構與技術棧

* **前端介面 (Frontend)**：Streamlit UI + 內嵌 HTML5 Canvas (60 FPS 高效能 2D 遊戲引擎)
* **後端邏輯 (Backend)**：Python (`streamlit`, `redis-py`)
* **資料庫 (Database)**：Redis 7.x (透過 Docker 容器化部署)

---

## 📊 系統設計與 UML 架構

### 1. 用例圖 (Use Case Diagram)
```mermaid
graph TD
    Player["玩家 / 會員"]
    
    subgraph 系統功能範圍 [osu! Redis Leaderboard System]
        UC1[會員註冊與登入]
        UC2["選擇遊戲模式<br>(經典/競速/生存)"]
        UC3["進行 HTML5 Canvas<br>互動遊戲挑戰"]
        UC4["一鍵安全登記成績<br>(Pipeline 寫入)"]
        UC5["查看各模式獨立排行榜<br>(ZSET 排序)"]
        UC6["瀏覽即時動態牆<br>(Pub/Sub & Lists)"]
    end

    Player --> UC1
    Player --> UC2
    Player --> UC3
    Player --> UC4
    Player --> UC5
    Player --> UC6
```
### 2. 部署圖 (Deployment Diagram)
```mermaid
graph LR
    subgraph ClientDevice ["客戶端裝置 (Client)"]
        Browser["網頁瀏覽器<br>- HTML5 Canvas 遊戲引擎<br>- Streamlit 前端 UI"]
    end

    subgraph ServerEnvironment ["雲端 / 本地伺服器環境"]
        subgraph PythonApp ["應用程式伺服器"]
            Streamlit["Streamlit Python 核心服務<br>- app.py 路由與邏輯控制<br>- redis-py 連線驅動"]
        end

        subgraph DatabaseServer ["資料庫伺服器"]
            Redis["Redis 記憶體資料庫 (Port 6379)<br>- ZSET (多模式排行榜)<br>- Hashes (會員與個資)<br>- Pub/Sub & Lists (動態牆)"]
        end
    end

    Browser -->|HTTP / Query Parameters| Streamlit
    Streamlit -->|"TCP Socket (redis-py)"| Redis
```
