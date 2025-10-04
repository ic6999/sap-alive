export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    
    // 存储应用状态的内存数据结构
    if (!env.appStates) {
      env.appStates = {
        us: { 
          lastStartTime: null, 
          status: 'initializing',
          lastError: null,
          appUrl: 'https://us0.cfapps.us10-001.hana.ondemand.com/sub'
        },
        sg: { 
          lastStartTime: null, 
          status: 'initializing',
          lastError: null,
          appUrl: 'https://sggudgxx.cfapps.ap21.hana.ondemand.com/sub'
        }
      };
    }

    // 处理API请求
    if (url.pathname === '/api/restart-all') {
      const results = await batchRestartSAP(env);
      return new Response(JSON.stringify(results), {
        headers: { 'Content-Type': 'application/json' }
      });
    }
    
    if (url.pathname === '/api/restart-us') {
      const result = await restartSingleSAP(env, 0);
      return new Response(JSON.stringify(result), {
        headers: { 'Content-Type': 'application/json' }
      });
    }
    
    if (url.pathname === '/api/restart-sg') {
      const result = await restartSingleSAP(env, 1);
      return new Response(JSON.stringify(result), {
        headers: { 'Content-Type': 'application/json' }
      });
    }
    
    if (url.pathname === '/api/status') {
      return new Response(JSON.stringify(env.appStates), {
        headers: { 'Content-Type': 'application/json' }
      });
    }
    
    // 新增：强制刷新状态的API
    if (url.pathname === '/api/force-check') {
      await checkApplicationStatus(env);
      return new Response(JSON.stringify(env.appStates), {
        headers: { 'Content-Type': 'application/json' }
      });
    }

    // 确保首次检查状态
    if (!env.initialCheckDone) {
      env.initialCheckDone = true;
      // 使用超时机制确保不会无限期停留在初始化中
      Promise.race([
        checkApplicationStatus(env),
        new Promise(resolve => {
          setTimeout(() => {
            console.log("状态检查超时，强制更新状态为未知");
            // 超时后将状态从初始化中改为未知
            if (env.appStates) {
              env.appStates.us.status = 'unknown';
              env.appStates.sg.status = 'unknown';
            }
            resolve();
          }, 15000); // 15秒超时
        })
      ]);
    }

    // 提供主界面
    return new Response(generateHtml(), {
      status: 200,
      headers: { 'Content-Type': 'text/html; charset=UTF-8' }
    });
  },

  async scheduled(event, env, ctx) {
    ctx.waitUntil(
      (async () => {
        try {
          const results = await batchRestartSAP(env);
          console.log('定时批量重启结果：\n' + results.join('\n'));
        } catch (error) {
          console.error('定时执行出错: ' + error.message);
        }
      })()
    );
  }
};

// 检查应用状态的函数 - 增强错误处理
async function checkApplicationStatus(env) {
  const accounts = [
    {
      uaaUrl: "https://uaa.cf.us10-001.hana.ondemand.com/oauth/token",
      apiUrl: "https://api.cf.us10-001.hana.ondemand.com",
      username: env.SAP_USERNAME,
      password: env.SAP_PASSWORD,
      appGuid: env.US_SAP_APP_GUID,
      regionKey: 'us',
      regionName: '美国'
    },
    {
      uaaUrl: "https://uaa.cf.ap21.hana.ondemand.com/oauth/token",
      apiUrl: "https://api.cf.ap21.hana.ondemand.com",
      username: env.SAP_USERNAME,
      password: env.SAP_PASSWORD,
      appGuid: env.SG_SAP_APP_GUID,
      regionKey: 'sg',
      regionName: '新加坡'
    }
  ];

  for (const account of accounts) {
    try {
      console.log(`开始检查${account.regionName}区域状态`);
      
      // 验证必要参数
      if (!account.username || !account.password || !account.appGuid) {
        throw new Error("缺少必要的认证参数");
      }

      // 获取Token - 增加超时
      const tokenResponse = await Promise.race([
        fetch(account.uaaUrl, {
          method: "POST",
          headers: {
            "Content-Type": "application/x-www-form-urlencoded",
            "Authorization": "Basic Y2Y6"
          },
          body: new URLSearchParams({
            grant_type: "password",
            username: account.username,
            password: account.password,
            scope: "cloud_controller.read"
          })
        }),
        new Promise((_, reject) => 
          setTimeout(() => reject(new Error("获取Token超时")), 8000)
        )
      ]);

      if (!tokenResponse.ok) {
        const errorText = await tokenResponse.text();
        console.error(`${account.regionName}区域Token获取失败: ${errorText}`);
        throw new Error(`Token获取失败 (HTTP ${tokenResponse.status})`);
      }

      const tokenData = await tokenResponse.json();
      const accessToken = tokenData.access_token;
      if (!accessToken) {
        throw new Error("未获取到有效Token");
      }

      // 获取应用状态 - 增加超时
      const appResponse = await Promise.race([
        fetch(
          account.apiUrl + "/v3/apps/" + account.appGuid,
          {
            method: "GET",
            headers: {
              Authorization: "Bearer " + accessToken,
              "Content-Type": "application/json"
            }
          }
        ),
        new Promise((_, reject) => 
          setTimeout(() => reject(new Error("获取应用状态超时")), 8000)
        )
      ]);

      if (!appResponse.ok) {
        const errorText = await appResponse.text();
        console.error(`${account.regionName}区域获取应用状态失败: ${errorText}`);
        throw new Error(`获取应用状态失败 (HTTP ${appResponse.status})`);
      }

      const appData = await appResponse.json();
      console.log(`${account.regionName}区域状态获取成功:`, appData.state);
      
      // 更新状态和最后启动时间
      if (env.appStates) {
        env.appStates[account.regionKey] = {
          ...env.appStates[account.regionKey],
          status: appData.state === 'STARTED' ? 'running' : 'error',
          lastStartTime: appData.lifecycle?.data?.last_started_at || new Date().toISOString(),
          lastError: null
        };
      }
    } catch (err) {
      console.error(`${account.regionName}区域状态检查失败: ${err.message}`);
      if (env.appStates) {
        env.appStates[account.regionKey] = {
          ...env.appStates[account.regionKey],
          status: 'error',
          lastError: err.message
        };
      }
    }
  }
}

// 生成主界面HTML
function generateHtml() {
  return `
  <!DOCTYPE html>
  <html lang="zh-CN">
  <head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>SAP多区域应用监控</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link href="https://cdn.jsdelivr.net/npm/font-awesome@4.7.0/css/font-awesome.min.css" rel="stylesheet">
    <script>
      tailwind.config = {
        theme: {
          extend: {
            colors: {
              primary: '#165DFF',
              success: '#36D399',
              warning: '#FBBD23',
              error: '#F87272',
              neutral: '#3D4451',
            },
            fontFamily: {
              inter: ['Inter', 'system-ui', 'sans-serif'],
            },
          },
        }
      }
    </script>
    <style type="text/tailwindcss">
      @layer utilities {
        .card-shadow {
          box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1), 0 4px 6px -2px rgba(0, 0, 0, 0.05);
        }
        .btn-hover {
          @apply transition-all duration-300 hover:shadow-lg transform hover:-translate-y-0.5;
        }
        .status-pulse {
          animation: pulse 2s cubic-bezier(0.4, 0, 0.6, 1) infinite;
        }
        @keyframes pulse {
          0%, 100% { opacity: 1; }
          50% { opacity: 0.5; }
        }
        .url-link {
          @apply font-medium text-primary hover:underline break-all cursor-pointer;
        }
      }
    </style>
  </head>
  <body class="bg-gray-50 font-inter text-neutral min-h-screen">
    <!-- 顶部导航 -->
    <header class="bg-white shadow-md fixed w-full top-0 z-10">
      <div class="container mx-auto px-4 py-4 flex flex-wrap items-center justify-between">
        <div class="flex items-center space-x-2">
          <i class="fa fa-cloud text-primary text-2xl"></i>
          <h1 class="text-xl md:text-2xl font-bold text-neutral">SAP多区域应用监控</h1>
        </div>
        
        <div class="flex space-x-3 mt-3 sm:mt-0">
          <button id="refresh-all" class="flex items-center space-x-2 bg-primary text-white px-4 py-2 rounded-lg btn-hover">
            <i class="fa fa-refresh"></i>
            <span>全局刷新</span>
          </button>
          <button id="force-refresh" class="flex items-center space-x-2 bg-primary/80 text-white px-4 py-2 rounded-lg btn-hover">
            <i class="fa fa-refresh fa-spin"></i>
            <span>强制刷新</span>
          </button>
          <button id="restart-all" class="flex items-center space-x-2 bg-warning text-white px-4 py-2 rounded-lg btn-hover">
            <i class="fa fa-repeat"></i>
            <span>全局重启</span>
          </button>
        </div>
      </div>
    </header>

    <!-- 主内容区 -->
    <main class="container mx-auto px-4 pt-24 pb-16">
      <!-- 状态卡片区域 -->
      <div class="grid grid-cols-1 md:grid-cols-2 gap-8">
        <!-- 美国区域卡片 -->
        <div id="us-card" class="bg-white rounded-xl p-6 card-shadow transition-all duration-300 hover:shadow-xl">
          <div class="flex justify-between items-start mb-6">
            <div class="flex items-center space-x-3">
              <div class="bg-blue-100 p-3 rounded-full">
                <i class="fa fa-flag text-primary text-xl"></i>
              </div>
              <h2 class="text-xl font-bold">美国区域</h2>
            </div>
            <span id="us-status-badge" class="px-3 py-1 rounded-full text-sm font-medium bg-primary/20 text-primary">
              <i class="fa fa-circle-o-notch fa-spin mr-1"></i>初始化中
            </span>
          </div>
          
          <div class="space-y-4 mb-6">
            <div class="flex flex-wrap justify-between items-center">
              <span class="text-gray-500">应用URL</span>
              <div class="ml-2 mt-1 sm:mt-0">
                <a id="us-app-url" href="https://us0.cfapps.us10-001.hana.ondemand.com/sub" target="_blank" class="url-link">
                  https://us0.cfapps.us10-001.hana.ondemand.com/sub
                </a>
              </div>
            </div>
            <div class="flex justify-between items-center">
              <span class="text-gray-500">最后启动时间</span>
              <span id="us-last-start" class="font-medium">--</span>
            </div>
            <div class="h-px bg-gray-100"></div>
            <div id="us-error" class="text-error hidden">
              <i class="fa fa-exclamation-circle mr-1"></i>
              <span></span>
            </div>
          </div>
          
          <div class="flex space-x-3">
            <button id="refresh-us" class="flex-1 bg-primary/10 text-primary px-4 py-2 rounded-lg btn-hover">
              <i class="fa fa-refresh mr-1"></i> 刷新状态
            </button>
            <button id="restart-us" class="flex-1 bg-warning/10 text-warning px-4 py-2 rounded-lg btn-hover">
              <i class="fa fa-repeat mr-1"></i> 重启应用
            </button>
          </div>
        </div>
        
        <!-- 新加坡区域卡片 -->
        <div id="sg-card" class="bg-white rounded-xl p-6 card-shadow transition-all duration-300 hover:shadow-xl">
          <div class="flex justify-between items-start mb-6">
            <div class="flex items-center space-x-3">
              <div class="bg-green-100 p-3 rounded-full">
                <i class="fa fa-flag text-green-600 text-xl"></i>
              </div>
              <h2 class="text-xl font-bold">新加坡区域</h2>
            </div>
            <span id="sg-status-badge" class="px-3 py-1 rounded-full text-sm font-medium bg-primary/20 text-primary">
              <i class="fa fa-circle-o-notch fa-spin mr-1"></i>初始化中
            </span>
          </div>
          
          <div class="space-y-4 mb-6">
            <div class="flex flex-wrap justify-between items-center">
              <span class="text-gray-500">应用URL</span>
              <div class="ml-2 mt-1 sm:mt-0">
                <a id="sg-app-url" href="https://sggudgxx.cfapps.ap21.hana.ondemand.com/sub" target="_blank" class="url-link">
                  https://sggudgxx.cfapps.ap21.hana.ondemand.com/sub
                </a>
              </div>
            </div>
            <div class="flex justify-between items-center">
              <span class="text-gray-500">最后启动时间</span>
              <span id="sg-last-start" class="font-medium">--</span>
            </div>
            <div class="h-px bg-gray-100"></div>
            <div id="sg-error" class="text-error hidden">
              <i class="fa fa-exclamation-circle mr-1"></i>
              <span></span>
            </div>
          </div>
          
          <div class="flex space-x-3">
            <button id="refresh-sg" class="flex-1 bg-primary/10 text-primary px-4 py-2 rounded-lg btn-hover">
              <i class="fa fa-refresh mr-1"></i> 刷新状态
            </button>
            <button id="restart-sg" class="flex-1 bg-warning/10 text-warning px-4 py-2 rounded-lg btn-hover">
              <i class="fa fa-repeat mr-1"></i> 重启应用
            </button>
          </div>
        </div>
      </div>
    </main>

    <!-- 确认对话框 -->
    <div id="confirm-modal" class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50 hidden">
      <div class="bg-white rounded-xl p-6 max-w-md w-full mx-4 transform transition-all">
        <h3 id="confirm-title" class="text-xl font-bold mb-2">确认操作</h3>
        <p id="confirm-message" class="text-gray-600 mb-6">请确认是否执行此操作</p>
        <div class="flex space-x-3">
          <button id="confirm-cancel" class="flex-1 bg-gray-200 text-gray-800 px-4 py-2 rounded-lg btn-hover">
            取消
          </button>
          <button id="confirm-ok" class="flex-1 bg-error text-white px-4 py-2 rounded-lg btn-hover">
            确认
          </button>
        </div>
      </div>
    </div>

    <script>
      // 格式化时间
      function formatDate(dateString) {
        if (!dateString) return '--';
        const date = new Date(dateString);
        return date.toLocaleString('zh-CN', {
          year: 'numeric',
          month: '2-digit',
          day: '2-digit',
          hour: '2-digit',
          minute: '2-digit',
          second: '2-digit'
        });
      }
      
      // 更新状态显示
      function updateStatusDisplay(data) {
        updateRegionStatus('us', data.us);
        updateRegionStatus('sg', data.sg);
      }
      
      // 更新单个区域状态
      function updateRegionStatus(region, status) {
        const badge = document.getElementById(region + '-status-badge');
        const lastStart = document.getElementById(region + '-last-start');
        const errorDiv = document.getElementById(region + '-error');
        const errorText = errorDiv.querySelector('span');
        const card = document.getElementById(region + '-card');
        
        // 更新状态徽章
        badge.className = '';
        badge.classList.add('px-3', 'py-1', 'rounded-full', 'text-sm', 'font-medium');
        
        switch(status.status) {
          case 'running':
            badge.classList.add('bg-success/20', 'text-success');
            badge.innerHTML = '<i class="fa fa-check-circle mr-1"></i>运行中';
            card.classList.remove('border-error', 'border-2');
            break;
          case 'error':
            badge.classList.add('bg-error/20', 'text-error');
            badge.innerHTML = '<i class="fa fa-exclamation-circle mr-1 status-pulse"></i>异常';
            card.classList.add('border-error', 'border-2');
            break;
          case 'restarting':
            badge.classList.add('bg-warning/20', 'text-warning');
            badge.innerHTML = '<i class="fa fa-spinner fa-spin mr-1"></i>重启中';
            break;
          case 'initializing':
            badge.classList.add('bg-primary/20', 'text-primary');
            badge.innerHTML = '<i class="fa fa-circle-o-notch fa-spin mr-1"></i>初始化中';
            break;
          default:
            badge.classList.add('bg-gray-100', 'text-gray-800');
            badge.innerHTML = '<i class="fa fa-question-circle mr-1"></i>未知';
        }
        
        // 更新最后启动时间
        lastStart.textContent = formatDate(status.lastStartTime);
        
        // 更新错误信息
        if (status.lastError) {
          errorText.textContent = status.lastError;
          errorDiv.classList.remove('hidden');
        } else {
          errorDiv.classList.add('hidden');
        }
      }
      
      // 加载状态数据
      async function loadStatus() {
        try {
          const response = await fetch('/api/status');
          if (!response.ok) throw new Error('获取状态失败 (HTTP ' + response.status + ')');
          
          const data = await response.json();
          updateStatusDisplay(data);
          return data;
        } catch (error) {
          console.error('获取状态出错: ' + error.message);
          // 更新UI显示错误
          const regions = ['us', 'sg'];
          regions.forEach(region => {
            const badge = document.getElementById(region + '-status-badge');
            badge.className = 'px-3 py-1 rounded-full text-sm font-medium bg-error/20 text-error';
            badge.innerHTML = '<i class="fa fa-exclamation-circle mr-1"></i>加载失败';
            
            const errorDiv = document.getElementById(region + '-error');
            const errorText = errorDiv.querySelector('span');
            errorText.textContent = '无法获取状态: ' + error.message;
            errorDiv.classList.remove('hidden');
          });
        }
      }
      
      // 强制刷新状态
      async function forceRefreshStatus() {
        try {
          // 更新UI显示正在刷新
          const regions = ['us', 'sg'];
          regions.forEach(region => {
            const badge = document.getElementById(region + '-status-badge');
            badge.className = 'px-3 py-1 rounded-full text-sm font-medium bg-primary/20 text-primary';
            badge.innerHTML = '<i class="fa fa-circle-o-notch fa-spin mr-1"></i>刷新中';
          });
          
          const response = await fetch('/api/force-check');
          if (!response.ok) throw new Error('强制刷新失败 (HTTP ' + response.status + ')');
          
          const data = await response.json();
          updateStatusDisplay(data);
        } catch (error) {
          console.error('强制刷新出错: ' + error.message);
          loadStatus(); // 尝试普通刷新
        }
      }
      
      // 重启所有区域
      async function restartAll() {
        try {
          // 更新UI显示重启中状态
          document.getElementById('us-status-badge').className = 'px-3 py-1 rounded-full text-sm font-medium bg-warning/20 text-warning';
          document.getElementById('us-status-badge').innerHTML = '<i class="fa fa-spinner fa-spin mr-1"></i>重启中';
          
          document.getElementById('sg-status-badge').className = 'px-3 py-1 rounded-full text-sm font-medium bg-warning/20 text-warning';
          document.getElementById('sg-status-badge').innerHTML = '<i class="fa fa-spinner fa-spin mr-1"></i>重启中';
          
          const response = await fetch('/api/restart-all');
          if (!response.ok) throw new Error('重启请求失败 (HTTP ' + response.status + ')');
          
          await response.json();
          // 等待更长时间再刷新，给应用重启留出时间
          setTimeout(loadStatus, 5000);
        } catch (error) {
          console.error('全局重启出错: ' + error.message);
          setTimeout(loadStatus, 1000);
        }
      }
      
      // 重启单个区域
      async function restartRegion(region, name) {
        try {
          // 更新UI显示重启中状态
          const badge = document.getElementById(region + '-status-badge');
          badge.className = 'px-3 py-1 rounded-full text-sm font-medium bg-warning/20 text-warning';
          badge.innerHTML = '<i class="fa fa-spinner fa-spin mr-1"></i>重启中';
          
          const response = await fetch('/api/restart-' + region);
          if (!response.ok) throw new Error('重启请求失败 (HTTP ' + response.status + ')');
          
          await response.json();
          // 等待更长时间再刷新
          setTimeout(loadStatus, 5000);
        } catch (error) {
          console.error(name + '区域重启出错: ' + error.message);
          setTimeout(loadStatus, 1000);
        }
      }
      
      // 初始化事件监听
      function initEventListeners() {
        // 全局刷新
        document.getElementById('refresh-all').addEventListener('click', loadStatus);
        
        // 强制刷新
        document.getElementById('force-refresh').addEventListener('click', forceRefreshStatus);
        
        // 美国区域刷新
        document.getElementById('refresh-us').addEventListener('click', loadStatus);
        
        // 新加坡区域刷新
        document.getElementById('refresh-sg').addEventListener('click', loadStatus);
        
        // 确认对话框
        const modal = document.getElementById('confirm-modal');
        const confirmOk = document.getElementById('confirm-ok');
        const confirmCancel = document.getElementById('confirm-cancel');
        const confirmTitle = document.getElementById('confirm-title');
        const confirmMessage = document.getElementById('confirm-message');
        
        let confirmAction = null;
        
        confirmOk.addEventListener('click', () => {
          modal.classList.add('hidden');
          if (confirmAction) confirmAction();
          confirmAction = null;
        });
        
        confirmCancel.addEventListener('click', () => {
          modal.classList.add('hidden');
          confirmAction = null;
        });
        
        // 全局重启
        document.getElementById('restart-all').addEventListener('click', () => {
          confirmTitle.textContent = '确认全局重启';
          confirmMessage.textContent = '确定要重启所有区域的SAP应用吗？这可能会导致短暂的服务中断。';
          confirmAction = restartAll;
          modal.classList.remove('hidden');
        });
        
        // 美国区域重启
        document.getElementById('restart-us').addEventListener('click', () => {
          confirmTitle.textContent = '确认重启美国区域';
          confirmMessage.textContent = '确定要重启美国区域的SAP应用吗？这可能会导致短暂的服务中断。';
          confirmAction = () => restartRegion('us', '美国');
          modal.classList.remove('hidden');
        });
        
        // 新加坡区域重启
        document.getElementById('restart-sg').addEventListener('click', () => {
          confirmTitle.textContent = '确认重启新加坡区域';
          confirmMessage.textContent = '确定要重启新加坡区域的SAP应用吗？这可能会导致短暂的服务中断。';
          confirmAction = () => restartRegion('sg', '新加坡');
          modal.classList.remove('hidden');
        });
      }
      
      // 页面加载完成后初始化
      document.addEventListener('DOMContentLoaded', () => {
        initEventListeners();
        // 立即加载状态
        loadStatus();
        
        // 每10秒自动刷新一次状态
        setInterval(loadStatus, 10000);
        
        // 初始化时如果15秒后仍为初始化中，自动尝试强制刷新
        setTimeout(() => {
          const usStatus = document.getElementById('us-status-badge').innerHTML;
          const sgStatus = document.getElementById('sg-status-badge').innerHTML;
          
          if (usStatus.includes('初始化中') || sgStatus.includes('初始化中')) {
            console.log('检测到长时间初始化中，尝试强制刷新');
            forceRefreshStatus();
          }
        }, 15000);
      });
    </script>
  </body>
  </html>
  `;
  }

// 核心批量重启SAP函数
async function batchRestartSAP(env) {
  const accounts = [
    {
      uaaUrl: "https://uaa.cf.us10-001.hana.ondemand.com/oauth/token",
      apiUrl: "https://api.cf.us10-001.hana.ondemand.com",
      username: env.SAP_USERNAME,
      password: env.SAP_PASSWORD,
      appGuid: env.US_SAP_APP_GUID,
      regionKey: 'us',
      regionName: '美国',
      appUrl: 'https://us0.cfapps.us10-001.hana.ondemand.com/sub'
    },
    {
      uaaUrl: "https://uaa.cf.ap21.hana.ondemand.com/oauth/token",
      apiUrl: "https://api.cf.ap21.hana.ondemand.com",
      username: env.SAP_USERNAME,
      password: env.SAP_PASSWORD,
      appGuid: env.SG_SAP_APP_GUID,
      regionKey: 'sg',
      regionName: '新加坡',
      appUrl: 'https://sggudgxx.cfapps.ap21.hana.ondemand.com/sub'
    }
  ];

  if (env.appStates) {
    env.appStates.us.appUrl = accounts[0].appUrl;
    env.appStates.sg.appUrl = accounts[1].appUrl;
  }

  return await Promise.all(
    accounts.map(async (account) => {
      try {
        if (env.appStates) {
          env.appStates[account.regionKey] = {
            ...env.appStates[account.regionKey],
            status: 'restarting',
            appUrl: account.appUrl
          };
        }

        const tokenResponse = await fetch(account.uaaUrl, {
          method: "POST",
          headers: {
            "Content-Type": "application/x-www-form-urlencoded",
            "Authorization": "Basic Y2Y6"
          },
          body: new URLSearchParams({
            grant_type: "password",
            username: account.username,
            password: account.password,
            scope: "cloud_controller.write"
          })
        });

        if (!tokenResponse.ok) {
          const error = new Error("Token获取失败: " + await tokenResponse.text());
          if (env.appStates) {
            env.appStates[account.regionKey] = {
              ...env.appStates[account.regionKey],
              status: 'error',
              lastError: error.message,
              appUrl: account.appUrl
            };
          }
          throw error;
        }

        const tokenData = await tokenResponse.json();
        const accessToken = tokenData.access_token;
        if (!accessToken) {
          const error = new Error("未获取到有效Token");
          if (env.appStates) {
            env.appStates[account.regionKey] = {
              ...env.appStates[account.regionKey],
              status: 'error',
              lastError: error.message,
              appUrl: account.appUrl
            };
          }
          throw error;
        }

        const restartResponse = await fetch(
          account.apiUrl + "/v3/apps/" + account.appGuid + "/actions/restart",
          {
            method: "POST",
            headers: {
              Authorization: "Bearer " + accessToken,
              "Content-Type": "application/json"
            }
          }
        );

        const now = new Date().toISOString();
        if (restartResponse.ok) {
          if (env.appStates) {
            env.appStates[account.regionKey] = {
              lastStartTime: now,
              status: 'running',
              lastError: null,
              appUrl: account.appUrl
            };
          }
          return account.regionName + "区域：应用重启成功";
        } else {
          const errorDetails = await restartResponse.text();
          const error = new Error(account.regionName + "区域：重启失败 - " + errorDetails);
          if (env.appStates) {
            env.appStates[account.regionKey] = {
              lastStartTime: env.appStates[account.regionKey]?.lastStartTime || now,
              status: 'error',
              lastError: errorDetails,
              appUrl: account.appUrl
            };
          }
          throw error;
        }
      } catch (err) {
        return account.regionName + "区域：执行出错 - " + err.message;
      }
    })
  );
}

// 重启单个SAP应用
async function restartSingleSAP(env, index) {
  const accounts = [
    {
      uaaUrl: "https://uaa.cf.us10-001.hana.ondemand.com/oauth/token",
      apiUrl: "https://api.cf.us10-001.hana.ondemand.com",
      username: env.SAP_USERNAME,
      password: env.SAP_PASSWORD,
      appGuid: env.US_SAP_APP_GUID,
      regionKey: 'us',
      regionName: '美国',
      appUrl: 'https://us0.cfapps.us10-001.hana.ondemand.com/sub'
    },
    {
      uaaUrl: "https://uaa.cf.ap21.hana.ondemand.com/oauth/token",
      apiUrl: "https://api.cf.ap21.hana.ondemand.com",
      username: env.SAP_USERNAME,
      password: env.SAP_PASSWORD,
      appGuid: env.SG_SAP_APP_GUID,
      regionKey: 'sg',
      regionName: '新加坡',
      appUrl: 'https://sggudgxx.cfapps.ap21.hana.ondemand.com/sub'
    }
  ];

  const account = accounts[index];
  
  if (env.appStates) {
    env.appStates[account.regionKey].appUrl = account.appUrl;
  }
  
  try {
    if (env.appStates) {
      env.appStates[account.regionKey] = {
        ...env.appStates[account.regionKey],
        status: 'restarting',
        appUrl: account.appUrl
      };
    }

    const tokenResponse = await fetch(account.uaaUrl, {
      method: "POST",
      headers: {
        "Content-Type": "application/x-www-form-urlencoded",
        "Authorization": "Basic Y2Y6"
      },
      body: new URLSearchParams({
        grant_type: "password",
        username: account.username,
        password: account.password,
        scope: "cloud_controller.write"
      })
    });

    if (!tokenResponse.ok) {
      const errorMsg = "Token获取失败: " + await tokenResponse.text();
      if (env.appStates) {
        env.appStates[account.regionKey] = {
          ...env.appStates[account.regionKey],
          status: 'error',
          lastError: errorMsg,
          appUrl: account.appUrl
        };
      }
      return { success: false, error: errorMsg };
    }

    const tokenData = await tokenResponse.json();
    const accessToken = tokenData.access_token;
    if (!accessToken) {
      const errorMsg = "未获取到有效Token";
      if (env.appStates) {
        env.appStates[account.regionKey] = {
          ...env.appStates[account.regionKey],
          status: 'error',
          lastError: errorMsg,
          appUrl: account.appUrl
        };
      }
      return { success: false, error: errorMsg };
    }

    const restartResponse = await fetch(
      account.apiUrl + "/v3/apps/" + account.appGuid + "/actions/restart",
      {
        method: "POST",
        headers: {
          Authorization: "Bearer " + accessToken,
          "Content-Type": "application/json"
        }
      }
    );

    const now = new Date().toISOString();
    if (restartResponse.ok) {
      if (env.appStates) {
        env.appStates[account.regionKey] = {
          lastStartTime: now,
          status: 'running',
          lastError: null,
          appUrl: account.appUrl
        };
      }
      return { success: true, message: account.regionName + "区域：应用重启成功" };
    } else {
      const errorDetails = await restartResponse.text();
      if (env.appStates) {
        env.appStates[account.regionKey] = {
          lastStartTime: env.appStates[account.regionKey]?.lastStartTime || now,
          status: 'error',
          lastError: errorDetails,
          appUrl: account.appUrl
        };
      }
      return { success: false, error: account.regionName + "区域：重启失败 - " + errorDetails };
    }
  } catch (err) {
    return { success: false, error: account.regionName + "区域：执行出错 - " + err.message };
  }
}
