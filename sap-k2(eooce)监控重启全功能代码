// 环境变量配置(必填)
let email = "antidanatoritania@gmail.com";      // SAP登录邮箱
let password = "Hh090909";   // SAP登录密码
let CHAT_ID = "";            // Telegram聊天CHAT_ID(可选)
let BOT_TOKEN = "";          // Telegram机器人TOKEN(可选)

// 应用配置(必填)：支持显式指定区域，未指定则自动识别
const MONITORED_APPS = [
  { url: "https://sggudgxx.cfapps.ap21.hana.ondemand.com", name: "s1", region: "AP" },  
  { url: "https://us0.cfapps.us10-001.hana.ondemand.com", name: "us0", region: "US" }
];

// 区域配置常量
const REGIONS = {
  US: {
    CF_API: "https://api.cf.us10-001.hana.ondemand.com",
    UAA_URL: "https://uaa.cf.us10-001.hana.ondemand.com",
    DOMAIN_PATTERN: /\.us10(-001)?\.hana\.ondemand\.com$/,
    name: "美国区域"
  },
  AP: {
    CF_API: "https://api.cf.ap21.hana.ondemand.com",
    UAA_URL: "https://uaa.cf.ap21.hana.ondemand.com",
    DOMAIN_PATTERN: /\.ap21\.hana\.ondemand\.com$/,
    name: "新加坡区域"
  }
};

// ------------------------------ 工具函数 ------------------------------
const pad = n => String(n).padStart(2, "0");
const sleep = ms => new Promise(r => setTimeout(r, ms));
const json = (o, c = 200) => new Response(JSON.stringify(o), {
  status: c,
  headers: { "content-type": "application/json" }
});

// 格式化为上海时间字符串
function formatShanghaiTime(date) {
  const utcTime = date.getTime() + (date.getTimezoneOffset() * 60000);
  const shanghaiTime = new Date(utcTime + (8 * 60 * 60 * 1000));
  return `${shanghaiTime.getFullYear()}-${pad(shanghaiTime.getMonth() + 1)}-${pad(shanghaiTime.getDate())} ${pad(shanghaiTime.getHours())}:${pad(shanghaiTime.getMinutes())}`;
}

// 从URL识别区域
function detectRegionFromUrl(url) {
  for (const [code, config] of Object.entries(REGIONS)) {
    if (config.DOMAIN_PATTERN.test(url)) return code;
  }
  return null;
}

// 获取应用所属区域（优先显式配置，其次自动识别）
function getAppRegion(app) {
  return app.region || detectRegionFromUrl(app.url) || "UNKNOWN";
}

// ------------------------------ KV存储（最后重启记录） ------------------------------
// 需在Cloudflare创建KV命名空间：RESTART_RECORDS
async function updateLastRestart(appName, region, operator, env) {
  if (!env.RESTART_RECORDS) return console.warn("[KV] 未配置RESTART_RECORDS命名空间");
  const key = `restart:${region}:${appName}`;
  const record = {
    time: formatShanghaiTime(new Date()),
    operator: operator.toLowerCase(),
    timestamp: Date.now()
  };
  try {
    await env.RESTART_RECORDS.put(key, JSON.stringify(record));
    console.log(`[KV] 更新记录: ${key} -> ${JSON.stringify(record)}`);
  } catch (e) {
    console.error(`[KV-error] 更新失败: ${e.message}`);
  }
}

async function getLastRestart(appName, region, env) {
  if (!env.RESTART_RECORDS) {
    console.warn("[KV] 未配置RESTART_RECORDS命名空间");
    return null;
  }
  const key = `restart:${region}:${appName}`;
  try {
    const data = await env.RESTART_RECORDS.get(key);
    return data ? JSON.parse(data) : null;
  } catch (e) {
    console.error(`[KV-error] 获取失败: ${e.message}`);
    return null;
  }
}

// ------------------------------ Telegram通知 ------------------------------
async function sendTelegramMessage(message) {
  if (!CHAT_ID || !BOT_TOKEN) return console.log("[Telegram] 未配置，跳过发送");
  try {
    const res = await fetch(`https://api.telegram.org/bot${BOT_TOKEN}/sendMessage`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ chat_id: CHAT_ID, text: message, parse_mode: "Markdown" })
    });
    const result = await res.json();
    res.ok ? console.log("[Telegram] 发送成功") : console.error(`[Telegram-error] ${result.description}`);
  } catch (e) {
    console.error(`[Telegram-error] ${e.message}`);
  }
}

// ------------------------------ CF API交互 ------------------------------
async function getUAAToken(uaaUrl) {
  console.log(`[Auth] 认证: ${email} @ ${uaaUrl}`);
  const authHeader = "Basic " + btoa("cf:");
  const body = new URLSearchParams({
    grant_type: "password",
    username: email,
    password: password,
    response_type: "token"
  });
  const res = await fetch(`${uaaUrl}/oauth/token`, {
    method: "POST",
    headers: { authorization: authHeader, "content-type": "application/x-www-form-urlencoded" },
    body: body
  });
  const text = await res.text();
  if (!res.ok) throw new Error(`UAA认证失败: ${res.status} ${text.slice(0, 200)}`);
  return JSON.parse(text).access_token;
}

async function cfRequest(url, token, method = "GET", payload = null) {
  const options = {
    method: method,
    headers: { authorization: `Bearer ${token}`, "content-type": "application/json" }
  };
  if (payload) options.body = JSON.stringify(payload);
  
  try {
    const res = await fetch(url, options);
    const text = await res.text();
    if (!res.ok) {
      throw new Error(`CF API错误: ${res.status} ${url} -> ${text.slice(0, 200)}`);
    }
    return text ? JSON.parse(text) : {};
  } catch (e) {
    console.error(`[cfRequest-error] ${e.message}`);
    throw e; // 抛出错误让调用者处理，保持异步错误链完整
  }
}

// ------------------------------ 应用操作 ------------------------------
// 获取应用GUID
async function getAppGuid(regionCode, appName, token) {
  const apiUrl = REGIONS[regionCode].CF_API;
  const res = await cfRequest(`${apiUrl}/v3/apps?names=${encodeURIComponent(appName)}`, token);
  if (!res.resources?.length) throw new Error(`应用${appName}未找到`);
  return res.resources[0].guid;
}

// 等待应用启动完成
async function waitAppReady(regionCode, appGuid, token) {
  const apiUrl = REGIONS[regionCode].CF_API;
  let delay = 2000;

  // 等待应用状态为STARTED
  for (let i = 0; i < 8; i++) {
    await sleep(delay);
    const app = await cfRequest(`${apiUrl}/v3/apps/${appGuid}`, token);
    if (app.state === "STARTED") break;
    if (i === 7) throw new Error(`应用未启动: 最终状态=${app.state}`);
    delay = Math.min(delay * 1.5, 15000);
  }

  // 等待进程运行
  const processes = await cfRequest(`${apiUrl}/v3/apps/${appGuid}/processes`, token);
  const webProcess = processes.resources.find(p => p.type === "web") || processes.resources[0];
  for (let i = 0; i < 10; i++) {
    await sleep(delay);
    const stats = await cfRequest(`${apiUrl}/v3/processes/${webProcess.guid}/stats`, token);
    if (stats.resources.some(s => s.state === "RUNNING")) return;
    if (i === 9) throw new Error("进程未运行");
    delay = Math.min(delay * 1.5, 15000);
  }
}

// 检查应用URL可用性
async function checkAppHealth(appUrl) {
  try {
    const res = await fetch(appUrl, { signal: AbortSignal.timeout(30000) });
    console.log(`[Health] ${appUrl} -> ${res.status}`);
    return res.status === 200;
  } catch (e) {
    console.log(`[Health] ${appUrl} 不可用: ${e.message}`);
    return false;
  }
}

// 重启单个应用
async function restartSingleApp(app, operator, env) {
  const { name, url } = app;
  const regionCode = getAppRegion(app);
  const region = REGIONS[regionCode];
  if (!region) throw new Error(`应用${name}区域无效: ${regionCode}`);

  try {
    // 1. 认证与准备
    const token = await getUAAToken(region.UAA_URL);
    const appGuid = await getAppGuid(regionCode, name, token);
    console.log(`[Restart] 开始: ${name}(${url}) | 区域: ${regionCode}`);

    // 2. 启停应用
    await cfRequest(`${region.CF_API}/v3/apps/${appGuid}/actions/stop`, token, "POST");
    await sleep(3000); // 等待停止完成
    await cfRequest(`${region.CF_API}/v3/apps/${appGuid}/actions/start`, token, "POST");
    await waitAppReady(regionCode, appGuid, token);

    // 3. 验证与记录
    const isHealthy = await checkAppHealth(url);
    if (isHealthy) {
      await updateLastRestart(name, regionCode, operator, env);
      const msg = `✅ *应用重启成功*\n名称: ${name}\nURL: ${url}\n区域: ${region.name}\n操作者: ${operator}\n时间: ${formatShanghaiTime(new Date())}`;
      await sendTelegramMessage(msg);
      return { app: name, status: "success", url: url, region: regionCode };
    } else {
      throw new Error("重启后URL访问异常");
    }
  } catch (e) {
    const msg = `❌ *应用重启失败*\n名称: ${name}\nURL: ${url}\n区域: ${region.name}\n错误: ${e.message}\n时间: ${formatShanghaiTime(new Date())}`;
    await sendTelegramMessage(msg);
    throw e;
  }
}

// 重启指定区域所有应用（后端核心函数，原代码缺失函数体）
async function restartRegionApps(regionCode, operator, env) {
  const region = REGIONS[regionCode];
  if (!region) throw new Error(`无效区域: ${regionCode}，有效区域: ${Object.keys(REGIONS).join(",")}`);

  // 筛选当前区域的应用
  const regionApps = MONITORED_APPS.filter(app => getAppRegion(app) === regionCode);
  if (regionApps.length === 0) throw new Error(`区域${region.name}无监控应用`);

  console.log(`[Restart-Region] ${region.name} 共${regionApps.length}个应用 | 操作者: ${operator}`);
  const results = [];

  // 逐个重启（避免并发导致API压力）
  for (const app of regionApps) {
    try {
      const res = await restartSingleApp(app, operator, env);
      results.push(res);
    } catch (e) {
      results.push({ app: app.name, status: "failed", url: app.url, error: e.message });
    }
    await sleep(2000); // 间隔2秒，降低API调用频率
  }

  // 发送区域重启汇总通知
  const successCount = results.filter(r => r.status === "success").length;
  const failCount = results.filter(r => r.status === "failed").length;
  const summaryMsg = `📊 *${region.name}重启汇总*\n操作者: ${operator}\n时间: ${formatShanghaiTime(new Date())}\n总数量: ${regionApps.length}\n成功: ${successCount}\n失败: ${failCount}\n失败应用: ${failCount ? results.filter(r => r.status === "failed").map(r => r.app).join(",") : "无"}`;
  await sendTelegramMessage(summaryMsg);
  return results;
}

// 监控应用状态（定时任务触发）
async function monitorAllApps(env) {
  console.log(`[Monitor] 开始监控所有应用`);
  const results = [];
  for (const app of MONITORED_APPS) {
    try {
      const isHealthy = await checkAppHealth(app.url);
      if (isHealthy) {
        results.push({ app: app.name, status: "healthy" });
        continue;
      }
      // 应用离线，自动重启（操作者标记为system）
      const offlineMsg = `⚠️ *应用离线*\n名称: ${app.name}\nURL: ${app.url}\n区域: ${REGIONS[getAppRegion(app)]?.name || "未知"}\n时间: ${formatShanghaiTime(new Date())}\n正在自动重启...`;
      await sendTelegramMessage(offlineMsg);
      const res = await restartSingleApp(app, "system", env);
      results.push(res);
    } catch (e) {
      results.push({ app: app.name, status: "error", error: e.message });
      console.error(`[Monitor-error] ${app.name}: ${e.message}`);
    }
    await sleep(1000); // 间隔1秒，避免并发
  }
  return results;
}

// ------------------------------ 前端页面生成 ------------------------------
function generateStatusPage(appsByRegion, lastUpdateTime) {
  // 生成区域分组HTML
  const regionSections = Object.entries(appsByRegion).map(([regionCode, apps]) => {
    const region = REGIONS[regionCode] || { name: `未知区域(${regionCode})` };
    
    // 生成应用卡片
    const appCards = apps.map(app => {
      const statusClass = app.healthy ? "status-up" : "status-down";
      const statusText = app.healthy ? "运行中" : "已停止";
      
      // 最后重启记录显示
      let restartHtml = '<p class="restart-record"><strong>最后重启:</strong> 暂无记录</p>';
      if (app.lastRestart) {
        restartHtml = `
          <p class="restart-record">
            <strong>最后重启:</strong> ${app.lastRestart.time}<br>
            <strong>操作者:</strong> ${app.lastRestart.operator === "system" ? "系统(定时)" : "用户(手动)"}
          </p>
        `;
      }
      
      // 辅助函数：截断长URL（避免卡片换行混乱）
      const truncateUrl = (url) => url.length > 45 ? url.slice(0, 42) + "..." : url;
      
      return `
        <div class="app-card ${statusClass}">
          <div class="card-header">
            <h3>${app.name}</h3>
            <span class="status-badge ${statusClass}">${statusText}</span>
          </div>
          <div class="card-body">
            <p><strong>URL:</strong> <a href="${app.url}" target="_blank" rel="noopener">${truncateUrl(app.url)}</a></p>
            ${restartHtml}
          </div>
        </div>
      `;
    }).join("");
    
    return `
      <div class="region-section">
        <div class="region-header">
          <h2>${region.name}</h2>
          <button class="btn restart-region-btn" onclick="restartRegion('${regionCode}')">
            重启该区域所有应用
          </button>
        </div>
        <div class="app-grid">${appCards}</div>
      </div>
    `;
  }).join("");

  return `
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>SAP Cloud 多区域监控</title>
  <style>
    :root {
      --up: #4CAF50;
      --down: #F44336;
      --bg: #f5f5f5;
      --card-bg: #fff;
      --text: #333;
      --border-radius: 6px;
      --shadow: 0 2px 4px rgba(0,0,0,0.1);
    }
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; background: var(--bg); color: var(--text); line-height: 1.5; }
    
    .container { max-width: 1100px; margin: 0 auto; padding: 15px; }
    header { 
      background: #667eea; color: #fff; padding: 18px 0; 
      border-radius: var(--border-radius); text-align: center;
      margin-bottom: 20px; box-shadow: var(--shadow);
    }
    h1 { font-size: 1.8rem; margin-bottom: 5px; }
    .subtitle { font-size: 0.95rem; opacity: 0.9; }
    
    /* 区域分组样式 */
    .region-section { 
      background: var(--card-bg); border-radius: var(--border-radius);
      padding: 15px; margin-bottom: 18px; box-shadow: var(--shadow);
    }
    .region-header { 
      display: flex; justify-content: space-between; align-items: center;
      padding-bottom: 12px; margin-bottom: 15px; border-bottom: 1px solid #eee;
    }
    .region-header h2 { font-size: 1.2rem; color: #444; }
    
    /* 应用卡片网格 */
    .app-grid { 
      display: grid; grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
      gap: 12px;
    }
    .app-card { 
      background: var(--card-bg); border-radius: var(--border-radius);
      box-shadow: var(--shadow); overflow: hidden;
    }
    .card-header { 
      padding: 12px 15px; display: flex; justify-content: space-between;
      align-items: center; border-bottom: 1px solid #eee;
    }
    .card-header h3 { font-size: 1rem; }
    .status-badge { 
      padding: 3px 10px; border-radius: 20px; font-size: 0.8rem;
      font-weight: 600;
    }
    .status-up { background: rgba(76,175,80,0.1); color: var(--up); }
    .status-down { background: rgba(244,67,54,0.1); color: var(--down); }
    
    .card-body { padding: 12px 15px; font-size: 0.85rem; }
    .card-body p { margin: 6px 0; }
    .card-body a { color: #1976D2; text-decoration: none; word-break: break-all; }
    .restart-record { color: #666; }
    
    /* 按钮样式 */
    .btn { 
      border: none; border-radius: var(--border-radius); padding: 8px 16px;
      font-size: 0.85rem; font-weight: 500; cursor: pointer;
      transition: opacity 0.2s;
    }
    .refresh-btn { 
      background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
      color: #fff;
    }
    .restart-region-btn { 
      background: linear-gradient(135deg, #ff6b6b 0%, #ee5a52 100%);
      color: #fff;
    }
    .btn:hover { opacity: 0.9; }
    
    /* 加载状态 */
    .loading { 
      position: fixed; top: 0; left: 0; width: 100%; height: 100%;
      background: rgba(0,0,0,0.5); display: flex; justify-content: center;
      align-items: center; z-index: 9999; color: #fff; font-size: 1rem;
    }
    .spinner { 
      border: 3px solid rgba(255,255,255,0.3); border-top: 3px solid #fff;
      border-radius: 50%; width: 30px; height: 30px; animation: spin 1s linear infinite;
      margin-right: 12px;
    }
    @keyframes spin { 0% { transform: rotate(0deg); } 100% { transform: rotate(360deg); } }
    
    .last-update { text-align: center; color: #666; font-size: 0.8rem; margin: 15px 0; }
    footer { 
      text-align: center; padding: 15px; color: #666; font-size: 0.8rem;
      border-top: 1px solid #eee; margin-top: 20px;
    }
    .footer-links a { color: #1976D2; text-decoration: none; margin: 0 8px; }
    
    /* 响应式适配 */
    @media (max-width: 768px) {
      .app-grid { grid-template-columns: 1fr; }
      .region-header { flex-direction: column; align-items: flex-start; gap: 10px; }
      .restart-region-btn { width: 100%; }
    }
  </style>
</head>
<body>
  <div class="container">
    <header>
      <h1>SAP Cloud 多区域应用监控</h1>
      <div class="subtitle">按区域管理 | 实时状态 | 重启记录</div>
    </header>
    
    <!-- 全局操作 -->
    <div class="controls" style="text-align: center; margin-bottom: 20px;">
      <button class="btn refresh-btn" onclick="window.location.reload()">刷新所有状态</button>
    </div>
    
    <!-- 区域分组内容 -->
    ${regionSections}
    
    <div class="last-update">页面最后更新: ${lastUpdateTime}</div>
    
    <!-- 加载弹窗 -->
    <div id="loading" class="loading" style="display: none;">
      <div class="spinner"></div>
      <div id="loading-text">处理中...</div>
    </div>
    
    <footer>
      <p>SAP Cloud 多区域应用管理系统</p>
      <div class="footer-links">
        <a href="https://github.com/eooce/Auto-deploy-sap-and-keepalive" target="_blank" rel="noopener">GitHub</a>
        <a href="https://www.youtube.com/@eooce" target="_blank" rel="noopener">YouTube</a>
        <a href="https://t.me/eooceu" target="_blank" rel="noopener">Telegram</a>
      </div>
      <p>&copy; ${new Date().getFullYear()} Auto-SAP</p>
    </footer>
  </div>

  <script>
    // 区域重启逻辑（前端交互，与后端接口对应）
    async function restartRegion(regionCode) {
      const regionNames = { US: "美国区域", AP: "新加坡区域" };
      const regionName = regionNames[regionCode] || regionCode;
      
      // 确认操作（避免误触）
      if (!confirm("确定重启" + regionName + "所有应用吗？\\n重启期间应用可能短暂不可用！")) {
        return;
      }
      
      // 显示加载状态
      const loadingEl = document.getElementById("loading");
      const loadingTextEl = document.getElementById("loading-text");
      loadingTextEl.textContent = "正在重启" + regionName + "应用...";
      loadingEl.style.display = "flex";
      
      try {
        // 调用后端区域重启接口
        const res = await fetch("/restart-region", {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify({ region: regionCode })
        });
        const data = await res.json();
        
        if (data.ok) {
          alert(regionName + "重启任务已提交！\\n结果将通过Telegram通知");
        } else {
          alert("重启失败: " + (data.error || "未知错误"));
        }
      } catch (e) {
        alert("接口调用失败: " + e.message);
      } finally {
        // 隐藏加载状态
        loadingEl.style.display = "none";
      }
    }
  </script>
</body>
</html>
  `;
}

// ------------------------------ 入口函数（Cloudflare Worker核心） ------------------------------
export default {
  async fetch(request, env, ctx) {
    // 从环境变量加载配置（优先级：环境变量 > 硬编码）
    email = env.EMAIL || email;
    password = env.PASSWORD || password;
    CHAT_ID = env.CHAT_ID || CHAT_ID;
    BOT_TOKEN = env.BOT_TOKEN || BOT_TOKEN;

    const url = new URL(request.url);
    try {
      // 1. 首页：显示区域分组监控页面
      if (url.pathname === "/") {
        // 按区域分组应用，并获取每个应用的最后重启记录
        const appsByRegion = {};
        for (const app of MONITORED_APPS) {
          const regionCode = getAppRegion(app);
          const healthy = await checkAppHealth(app.url);
          const lastRestart = await getLastRestart(app.name, regionCode, env);
          
          if (!appsByRegion[regionCode]) appsByRegion[regionCode] = [];
          appsByRegion[regionCode].push({ ...app, healthy, lastRestart });
        }
        
        const html = generateStatusPage(appsByRegion, formatShanghaiTime(new Date()));
        return new Response(html, { headers: { "content-type": "text/html;charset=UTF-8" } });
      }

      // 2. 区域重启接口（供前端调用）
      if (url.pathname === "/restart-region" && request.method === "POST") {
        const body = await request.json();
        const regionCode = body.region;
        
        // 参数校验
        if (!regionCode) return json({ ok: false, error: "缺少region参数" }, 400);
        if (!REGIONS[regionCode]) return json({ 
          ok: false, 
          error: `无效区域，有效区域: ${Object.keys(REGIONS).join(",")}` 
        }, 400);
        
        // 异步执行重启（避免前端等待超时）
        ctx.waitUntil(
          restartRegionApps(regionCode, "user", env)
            .catch(e => console.error(`[Region-Restart-Error] ${e.message}`))
        );
        
        return json({ ok: true, msg: `区域${REGIONS[regionCode].name}重启任务已启动` });
      }

      // 3. 状态查询接口（API，供外部系统调用）
      if (url.pathname === "/api/status") {
        const status = await Promise.all(
          MONITORED_APPS.map(async app => ({
            name: app.name,
            url: app.url,
            region: getAppRegion(app),
            healthy: await checkAppHealth(app.url),
            lastRestart: await getLastRestart(app.name, getAppRegion(app), env)
          }))
        );
        return json({ ok: true, status, timestamp: formatShanghaiTime(new Date()) });
      }

      // 4. 健康检查接口（供监控系统使用）
      if (url.pathname === "/health") {
        return json({ ok: true, timestamp: Date.now() });
      }

      // 默认响应： Worker运行中
      return new Response("SAP Cloud 多区域监控系统运行中", { status: 200 });
    } catch (e) {
      console.error(`[Fetch-Error] ${e.message}`);
      return json({ ok: false, error: e.message }, 500);
    }
  },

  // 定时任务（Cloudflare Scheduled Worker，自动监控应用状态）
  async scheduled(event, env, ctx) {
    try {
      // 加载环境变量配置
      email = env.EMAIL || email;
      password = env.PASSWORD || password;
      CHAT_ID = env.CHAT_ID || CHAT_ID;
      BOT_TOKEN = env.BOT_TOKEN || BOT_TOKEN;
      
      // 异步执行监控逻辑（确保任务完成）
      ctx.waitUntil(monitorAllApps(env));
    } catch (e) {
      console.error(`[Scheduled-Error] ${e.message}`);
    }
  }
};
