from datetime import datetime
import os
import redis
import streamlit as st
import streamlit.components.v1 as components

# --- Redis 智慧連線設定 (自動判斷本地 Docker 或雲端 Upstash) ---
REDIS_HOST = st.secrets.get(
    "REDIS_HOST", os.getenv("REDIS_HOST", "localhost")
)
REDIS_PORT = int(st.secrets.get("REDIS_PORT", os.getenv("REDIS_PORT", 6379)))
REDIS_PASSWORD = st.secrets.get(
    "REDIS_PASSWORD", os.getenv("REDIS_PASSWORD", None)
)

# 判斷是否為雲端環境（非 localhost 就啟用 SSL 加密）
is_cloud = REDIS_HOST != "localhost"

r = redis.Redis(
    host=REDIS_HOST,
    port=REDIS_PORT,
    password=REDIS_PASSWORD,
    ssl=is_cloud,  # 雲端自動啟用 SSL
    ssl_cert_reqs=None,
    decode_responses=True,
)

# 頁面基本設定
st.set_page_config(
    page_title="圓圈點擊多模式挑戰賽",
    page_icon="🚨",
    layout="wide",
)

# --- 初始化 Session State 用於管理登入狀態 ---
if "logged_in" not in st.session_state:
  st.session_state.logged_in = False
if "username" not in st.session_state:
  st.session_state.username = ""
if "user_title" not in st.session_state:
  st.session_state.user_title = "新手冒險者"

# --- 接收遊戲結束後的自動登記請求 (Query Parameters 處理) ---
query_params = st.query_params
if query_params.get("auto_submit") == "1":
  try:
    raw_score = query_params.get("score", 0)
    auto_player = query_params.get("player", "Anonymous")
    auto_title = query_params.get("title", "新手冒險者")
    auto_mode_key = query_params.get("modeKey", "mode:classic")
    auto_mode_label = query_params.get("modeLabel", "經典模式")
    current_time = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

    if auto_player.strip():
      if auto_mode_key == "mode:speedrun":
        new_time = float(raw_score)
        old_score = r.zscore(auto_mode_key, auto_player)
        old_time = float(old_score) if old_score is not None else float("inf")
        is_new_high = new_time < old_time
        best_score = min(new_time, old_time)

        if is_new_high:
          mins = int(new_time // 60)
          secs = new_time % 60
          time_str = f"{mins}分{secs:.2f}秒"
          broadcast_msg = f"⚡ 【競速新紀錄】會員 【{auto_player}】 創下神速：{time_str}！"
          r.zadd(auto_mode_key, {auto_player: new_time})
        else:
          broadcast_msg = f"⚡ 【競速模式】會員 【{auto_player}】 完成挑戰，用時 {new_time:.2f} 秒。"
      else:
        new_count = int(raw_score)
        old_score = r.zscore(auto_mode_key, auto_player)
        old_count = int(old_score) if old_score is not None else 0
        is_new_high = new_count > old_count
        best_score = max(new_count, old_count)

        if is_new_high:
          broadcast_msg = f"🏆 【破紀錄】會員 【{auto_player}】 在【{auto_mode_label}】創下最高分：{new_count} 分！"
          r.zadd(auto_mode_key, {auto_player: new_count}, gt=True)
        else:
          broadcast_msg = f"🎯 【{auto_mode_label}】會員 【{auto_player}】 獲得 {new_count} 分 (個人最高：{old_count}分)"

      with r.pipeline() as pipe:
        pipe.publish("announcements", broadcast_msg)
        pipe.lpush("broadcast_history", broadcast_msg)
        pipe.ltrim("broadcast_history", 0, 4)
        pipe.hset(
            f"player:profile:{auto_player}",
            mapping={
                "title": auto_title,
                f"best_{auto_mode_key}": best_score,
                "last_mode": auto_mode_label,
                "updated_at": current_time,
            },
        )
        pipe.execute()

      st.success(
          f"🎉 會員 **{auto_player}** 的成績已成功自動登記至【{auto_mode_label}】排行榜！"
      )
      st.balloons()
  except Exception as e:
    st.error(f"自動登記發生錯誤: {e}")

  st.query_params.clear()

# 自訂 CSS 樣式
st.markdown(
    """
    <style>
    .stMetric { background-color: #1e2530; padding: 15px; border-radius: 10px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); }
    </style>
""",
    unsafe_allow_html=True,
)

st.title("🚨圓圈點擊多模式挑戰賽")
st.markdown(
    "結合 **Redis 會員系統 (Hashes)**、**HTML5 Canvas 互動遊戲** 與 **Pipeline"
    " 批次寫入**的多模式排行榜系統"
)

# --- 側邊欄：會員登入與註冊系統 ---
st.sidebar.header("🔐 會員專區 (Redis Auth)")

if not st.session_state.logged_in:
  auth_mode = st.sidebar.radio("選擇操作", ["會員登入", "註冊帳號"])
  input_user = st.sidebar.text_input("帳號 (Username)")
  input_pass = st.sidebar.text_input("密碼 (Password)", type="password")

  if auth_mode == "註冊帳號":
    input_title = st.sidebar.selectbox(
        "選擇初始頭銜", ["奶蛙🐸", "滅建國🍆", "嘉豪😎", "大SB🤡"]
    )
    if st.sidebar.button("註冊新帳號", type="primary"):
      if input_user.strip() and input_pass.strip():
        # 檢查 Redis 中帳號是否存在
        user_key = f"user:{input_user.strip()}"
        if r.exists(user_key):
          st.sidebar.error("❌ 此帳號已被註冊，請更換帳號或直接登入！")
        else:
          # 寫入 Redis Hash
          r.hset(
              user_key,
              mapping={"password": input_pass, "title": input_title},
          )
          st.sidebar.success(
              "✅ 註冊成功！請切換至「會員登入」分頁進行登入。"
          )
      else:
        st.sidebar.error("⚠️ 帳號與密碼不得為空！")

  else:  # 登入邏輯
    if st.sidebar.button("登入系統", type="primary"):
      if input_user.strip() and input_pass.strip():
        user_key = f"user:{input_user.strip()}"
        if not r.exists(user_key):
          st.sidebar.error("❌ 找不到此帳號，請先註冊！")
        else:
          stored_pass = r.hget(user_key, "password")
          stored_title = r.hget(user_key, "title") or "奶蛙🐸"
          if stored_pass == input_pass:
            st.session_state.logged_in = True
            st.session_state.username = input_user.strip()
            st.session_state.user_title = stored_title
            st.sidebar.success(f"🎉 歡迎回來，{input_user}！")
            st.rerun()
          else:
            st.sidebar.error("❌ 密碼錯誤，請重新輸入！")
      else:
        st.sidebar.error("⚠️ 請輸入帳號與密碼！")

  st.sidebar.warning("⚠️ 請先登入會員，才能開始遊玩遊戲並登記成績！")

else:
  # 已登入狀態顯示
  st.sidebar.success(f"👤 當前登入會員：**{st.session_state.username}**")
  st.sidebar.info(f"🏆 會員頭銜：{st.session_state.user_title}")

  if st.sidebar.button("登出帳號"):
    st.session_state.logged_in = False
    st.session_state.username = ""
    st.rerun()

st.sidebar.divider()
# 遊戲模式選擇
mode_options = {
    "🎮 經典模式 (15秒點擊挑戰)": "mode:classic",
    "⚡ 競速模式 (極速點完20個圓圈)": "mode:speedrun",
    "🛡️ 生存模式 (0.7秒極限反應)": "mode:survival",
}
selected_mode_label = st.sidebar.selectbox(
    "選擇遊戲模式", list(mode_options.keys())
)
redis_mode_key = mode_options[selected_mode_label]

# --- 主畫面分頁 ---
tab1, tab2, tab3 = st.tabs(
    ["🎯 互動遊戲挑戰區", "🏆 各模式獨立排行榜", "📢 即時動態牆"]
)

with tab1:
  st.subheader(f"🕹️ 當前遊玩模式：{selected_mode_label}")
  st.markdown(
      "- **經典模式**：15秒內點擊越多圓圈越好。<br>- **競速模式**："
      "計時點完 20 個圓圈所花費的時間。<br>- **生存模式**："
      "每個圓圈限時 0.7 秒，逾時即遊戲結束，計算點擊總數。",
      unsafe_allow_html=True,
  )

  if not st.session_state.logged_in:
    st.error(
        "🔒 **您尚未登入！** 請先至左側邊欄完成「會員登入」後，即可解鎖遊戲與排行榜登記功能。"
    )
  else:
    # 已登入：動態將會員資料注入 HTML/JS 遊戲元件
    game_html = f"""
      <!DOCTYPE html>
      <html>
      <head>
      <style>
        body {{ background: #0e1117; color: white; font-family: sans-serif; text-align: center; margin: 0; padding: 5px; }}
        .game-box {{ display: flex; flex-direction: column; align-items: center; justify-content: center; }}
        #gameCanvas {{ background: #1a1c23; border: 2px solid #3b4252; border-radius: 12px; cursor: crosshair; box-shadow: 0 4px 15px rgba(0,0,0,0.5); }}
        .ui-container {{ margin-bottom: 10px; font-size: 18px; font-weight: bold; display: flex; gap: 25px; align-items: center; justify-content: center; }}
        button {{ background: #ff4b4b; color: white; border: none; padding: 10px 24px; font-size: 16px; border-radius: 8px; cursor: pointer; font-weight: bold; transition: 0.2s; }}
        button:hover {{ background: #ff2b2b; transform: scale(1.05); }}
        button:disabled {{ background: #555; cursor: not-allowed; transform: none; }}
        .score-board {{ color: #00ffcc; font-size: 18px; margin-top: 10px; font-weight: bold; min-height: 50px; }}
        .submit-link {{ background: #00ffcc; color: #0e1117; font-size: 16px; padding: 10px 22px; border-radius: 8px; text-decoration: none; font-weight: bold; display: inline-block; margin-top: 8px; transition: 0.2s; }}
        .submit-link:hover {{ background: #00e6b8; transform: scale(1.05); }}
        #timeBarContainer {{ width: 750px; height: 6px; background: #333; border-radius: 3px; margin-bottom: 8px; overflow: hidden; display: none; }}
        #timeBar {{ width: 100%; height: 100%; background: #ffaa00; transition: width 0.05s linear; }}
      </style>
      </head>
      <body>
        <div class="game-box">
          <div class="ui-container" id="uiStats">
            <div><span id="statTitle">狀態</span>: <span id="statVal" style="color: #ffaa00;">準備就緒</span></div>
          </div>
          <div id="timeBarContainer"><div id="timeBar"></div></div>
          <canvas id="gameCanvas" width="750" height="360"></canvas>
          <div id="resultMsg" class="score-board"></div>
          <br>
          <button id="startBtn" onclick="startGame()">開始挑戰</button>
        </div>

        <script>
          const playerName = "{st.session_state.username}";
          const playerTitle = "{st.session_state.user_title}";
          const modeKey = "{redis_mode_key}";
          const modeLabel = "{selected_mode_label}";

          const canvas = document.getElementById('gameCanvas');
          const ctx = canvas.getContext('2d');
          
          let score = 0;
          let isPlaying = false;
          let gameInterval = null;
          let circle = {{ x: 0, y: 0, radius: 26 }};

          let timeLeft = 15.0;
          let speedrunStartTime = 0;
          let survivalTimeLeft = 0.7;
          const survivalMaxTime = 0.7;

          function spawnCircle() {{
            const margin = 40;
            circle.x = margin + Math.random() * (canvas.width - margin * 2);
            circle.y = margin + Math.random() * (canvas.height - margin * 2);
          }}

          function draw() {{
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            if (isPlaying) {{
              ctx.beginPath();
              ctx.arc(circle.x, circle.y, circle.radius + 8, 0, Math.PI * 2);
              ctx.fillStyle = 'rgba(255, 75, 75, 0.25)';
              ctx.fill();

              ctx.beginPath();
              ctx.arc(circle.x, circle.y, circle.radius, 0, Math.PI * 2);
              ctx.fillStyle = '#ff4b4b';
              ctx.fill();
              ctx.lineWidth = 3;
              ctx.strokeStyle = '#ffffff';
              ctx.stroke();

              ctx.beginPath();
              ctx.arc(circle.x, circle.y, 6, 0, Math.PI * 2);
              ctx.fillStyle = '#ffffff';
              ctx.fill();
            }} else {{
              ctx.fillStyle = '#88b2cc';
              ctx.font = '18px sans-serif';
              ctx.textAlign = 'center';
              ctx.fillText("✨ 會員已就緒，點擊下方按鈕開始挑戰！", canvas.width / 2, canvas.height / 2);
            }}
          }}

          function startGame() {{
            score = 0;
            isPlaying = true;
            document.getElementById('startBtn').style.display = 'none';
            document.getElementById('resultMsg').innerHTML = '';
            spawnCircle();

            const timeBarContainer = document.getElementById('timeBarContainer');

            if (modeKey === 'mode:classic') {{
              timeLeft = 15.0;
              timeBarContainer.style.display = 'none';
              document.getElementById('statTitle').innerText = '⏱️ 剩餘時間';
              
              gameInterval = setInterval(() => {{
                timeLeft = Math.max(0, timeLeft - 0.05);
                document.getElementById('statVal').innerText = timeLeft.toFixed(1) + 's';
                document.getElementById('statVal').style.color = '#ffaa00';
                
                if (timeLeft <= 0) {{
                  endGame(`挑戰結束！獲得 <b>${{score}}</b> 分`);
                }} else {{
                  draw();
                }}
              }}, 50);

            }} else if (modeKey === 'mode:speedrun') {{
              speedrunStartTime = Date.now();
              timeBarContainer.style.display = 'none';
              document.getElementById('statTitle').innerText = '🎯 剩餘目標';
              
              gameInterval = setInterval(() => {{
                const elapsed = ((Date.now() - speedrunStartTime) / 1000).toFixed(2);
                document.getElementById('statVal').innerText = (20 - score) + ' 個 (已用时 ' + elapsed + 's)';
                document.getElementById('statVal').style.color = '#00ffcc';
                draw();
              }}, 50);

            }} else if (modeKey === 'mode:survival') {{
              survivalTimeLeft = 0.7;
              timeBarContainer.style.display = 'block';
              document.getElementById('statTitle').innerText = '🛡️ 生存得分';
              
              gameInterval = setInterval(() => {{
                survivalTimeLeft -= 0.05;
                const barPercent = Math.max(0, (survivalTimeLeft / survivalMaxTime) * 100);
                document.getElementById('timeBar').style.width = barPercent + '%';
                document.getElementById('statVal').innerText = score + ' 分';
                document.getElementById('statVal').style.color = '#ff4b4b';

                if (survivalTimeLeft <= 0) {{
                  endGame(`💥 逾時未點擊！生存挑戰結束！總得分：<b>${{score}}</b> 顆`);
                }} else {{
                  draw();
                }}
              }}, 50);
            }}
            draw();
          }}

          function endGame(msg) {{
            clearInterval(gameInterval);
            isPlaying = false;
            document.getElementById('timeBarContainer').style.display = 'none';
            document.getElementById('startBtn').style.display = 'inline-block';
            document.getElementById('startBtn').innerText = '重新挑戰';

            let finalScoreForDb = score;
            if (modeKey === 'mode:speedrun') {{
              finalScoreForDb = ((Date.now() - speedrunStartTime) / 1000).toFixed(2);
              msg = `⚡ 競速完成！總用時：<b>${{finalScoreForDb}}</b> 秒`;
            }}

            const baseUrl = window.parent.location.href.split('?')[0];
            const targetUrl = `${{baseUrl}}?auto_submit=1&score=${{finalScoreForDb}}&player=${{encodeURIComponent(playerName)}}&title=${{encodeURIComponent(playerTitle)}}&modeKey=${{encodeURIComponent(modeKey)}}&modeLabel=${{encodeURIComponent(modeLabel)}}`;

            document.getElementById('resultMsg').innerHTML = `
              ${{msg}}
              <br>
              <a href="${{targetUrl}}" target="_blank" class="submit-link">📥 點擊此處一鍵登記至排行榜</a>
            `;
            draw();
          }}

          canvas.addEventListener('click', (e) => {{
            if (!isPlaying) return;
            const rect = canvas.getBoundingClientRect();
            const mouseX = e.clientX - rect.left;
            const mouseY = e.clientY - rect.top;

            const dist = Math.hypot(mouseX - circle.x, mouseY - circle.y);
            if (dist <= circle.radius) {{
              score++;
              if (modeKey === 'mode:speedrun' && score >= 20) {{
                endGame();
                return;
              }}
              if (modeKey === 'mode:survival') {{
                survivalTimeLeft = 0.7;
              }}
              spawnCircle();
              draw();
            }}
          }});

          draw();
        </script>
      </body>
      </html>
      """

    components.html(game_html, height=560)

with tab2:
  st.subheader("🏆 各模式獨立排行榜")

  selected_leaderboard = st.selectbox(
      "選擇要查看的模式排行榜", list(mode_options.keys())
  )
  target_zset_key = mode_options[selected_leaderboard]

  total_players = r.zcard(target_zset_key)

  m1, m2 = st.columns(2)
  m1.metric("📌 當前排行榜", selected_leaderboard)
  m2.metric("👥 參與會員總數", f"{total_players} 人")

  st.divider()

  if target_zset_key == "mode:speedrun":
    top_players = r.zrange(target_zset_key, 0, 9, withscores=True)
  else:
    top_players = r.zrevrange(target_zset_key, 0, 9, withscores=True)

  if top_players:
    leaderboard_data = []
    for idx, p in enumerate(top_players):
      name = p[0]
      raw_score = p[1]

      profile = r.hgetall(f"player:profile:{name}")
      title = profile.get("title", "一般會員")
      updated = profile.get("updated_at", "-")

      if idx == 0:
        rank_display = "🥇 冠軍"
      elif idx == 1:
        rank_display = "🥈 亞軍"
      elif idx == 2:
        rank_display = "🥉 季軍"
      else:
        rank_display = f"第 {idx + 1} 名"

      if target_zset_key == "mode:speedrun":
        t_val = float(raw_score)
        mins = int(t_val // 60)
        secs = t_val % 60
        score_display = f"{mins}分{secs:.2f}秒 ({t_val}秒)"
      else:
        score_display = f"{int(raw_score)} 分"

      leaderboard_data.append({
          "名次": rank_display,
          "會員名稱": name,
          "頭銜": title,
          "最佳成績": score_display,
          "最後更新時間": updated,
      })

    st.table(leaderboard_data)
  else:
    st.info(
        f"目前【{selected_leaderboard}】尚無資料，趕快登入並挑戰登頂榜首吧！"
    )

with tab3:
  st.subheader("📢 系統即時動態牆")
  st.write("即時監聽 Redis 廣播與歷史挑戰紀錄：")

  messages = r.lrange("broadcast_history", 0, 4)
  if messages:
    for msg in messages:
      st.info(msg)
  else:
    st.caption("目前尚無即時動態消息。")
