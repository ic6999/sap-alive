const pad=n=>String(n).padStart(2,"0");
const sleep=ms=>new Promise(r=>setTimeout(r,ms));
const json=(o,c=200)=>new Response(JSON.stringify(o),{status:c,headers:{"content-type":"application/json"}});

async function cfGET(u,t){
  const r=await fetch(u,{headers:{authorization:`Bearer ${t}`}});
  const x=await r.text();
  if(!r.ok)throw new Error(`CF GET ${r.status} ${u}: ${x.slice(0,200)}`);
  return x?JSON.parse(x):{}
}
async function cfPOST(u,t,p){
  const r=await fetch(u,{method:"POST",headers:{authorization:`Bearer ${t}`,"content-type":"application/json"},body:p?JSON.stringify(p):null});
  const x=await r.text();
  if(!r.ok)throw new Error(`CF POST ${r.status} ${u}: ${x.slice(0,200)}`);
  return x?JSON.parse(x):{}
}
async function getUAAToken(env){
  const u=env.UAA_URL.replace(/\/+$/,"");
  const a="Basic "+btoa("cf:");
  const b=new URLSearchParams();
  b.set("grant_type","password");
  b.set("username",env.CF_USERNAME);
  b.set("password",env.CF_PASSWORD);
  b.set("response_type","token");
  const r=await fetch(`${u}/oauth/token`,{method:"POST",headers:{authorization:a,"content-type":"application/x-www-form-urlencoded"},body:b});
  const x=await r.text();
  if(!r.ok)throw new Error(`UAA token error: ${r.status} ${x}`);
  return JSON.parse(x).access_token
}
async function getAppState(api,tok,gid){
  const r=await cfGET(`${api}/v3/apps/${gid}`,tok);
  return r?.state||"UNKNOWN"
}
async function getWebProcessGuid(api,tok,gid){
  const r=await cfGET(`${api}/v3/apps/${gid}/processes`,tok);
  const w=r?.resources?.find(p=>p?.type==="web")||r?.resources?.[0];
  if(!w)throw new Error("No process found on app");
  return w.guid
}
async function getProcessStats(api,tok,pid){return cfGET(`${api}/v3/processes/${pid}/stats`,tok)}
async function resolveAppGuid(env,tok,api){
  if(env.APP_GUID)return env.APP_GUID;
  const org=await cfGET(`${api}/v3/organizations?names=${encodeURIComponent(env.ORG_NAME)}`,tok);
  if(!org?.resources?.length)throw new Error("ORG_NAME not found");
  const og=org.resources[0].guid;
  const sp=await cfGET(`${api}/v3/spaces?names=${encodeURIComponent(env.SPACE_NAME)}&organization_guids=${og}`,tok);
  if(!sp?.resources?.length)throw new Error("SPACE_NAME not found");
  const sg=sp.resources[0].guid;
  const apps=await cfGET(`${api}/v3/apps?names=${encodeURIComponent(env.APP_NAME)}&space_guids=${sg}`,tok);
  if(!apps?.resources?.length)throw new Error("APP_NAME not found");
  return apps.resources[0].guid
}
async function waitAppStarted(api,tok,gid){
  let d=2000,s="";
  for(let i=0;i<8;i++){
    await sleep(d);
    s=await getAppState(api,tok,gid);
    console.log("[app-state-check]",i,s);
    if(s==="STARTED")break;
    d=Math.min(d*1.6,15000)
  }
  if(s!=="STARTED")throw new Error(`App not STARTED in time, state=${s}`)
}
async function waitProcessInstancesRunning(api,tok,pid){
  let d=2000;
  for(let i=0;i<10;i++){
    const st=await getProcessStats(api,tok,pid);
    const ins=st?.resources||[];
    const states=ins.map(it=>it?.state);
    console.log("[proc-stats]",states.join(",")||"no-instances");
    if(states.some(s=>s==="RUNNING"))return;
    await sleep(d);
    d=Math.min(d*1.6,15000)
  }
  throw new Error("Process instances not RUNNING in time")
}

async function ensureRunning(env,{reason="unknown",force=!1}={}){
  console.log("[trigger]",reason,new Date().toISOString());
  const ymd=new Date().toISOString().slice(0,10);
  const lockKey=`start-success-lock:${ymd}`;
  if(!force){
    const ex=await env.START_LOCK.get(lockKey).catch(()=>null);
    if(ex){console.log("[lock] success-lock exists, skip",lockKey);return}
  }else{console.log("[lock] force=1, ignore success-lock")}
  const api=env.CF_API.replace(/\/+$/,"");
  const tok=await getUAAToken(env);
  const gid=await resolveAppGuid(env,tok,api);
  const pid=await getWebProcessGuid(api,tok,gid);
  const pre=await getProcessStats(api,tok,pid);
  const st=(pre?.resources||[]).map(it=>it?.state);
  console.log("[proc-before]",st.join(",")||"no-instances");
  if(st.some(s=>s==="RUNNING")){console.log("[decision] already RUNNING → nothing to do");return}
  let appState=await getAppState(api,tok,gid);
  console.log("[app-state-before]",appState);
  if(appState!=="STARTED"){
    await cfPOST(`${api}/v3/apps/${gid}/actions/start`,tok);
    console.log("[action] app start requested")
  }
  await waitAppStarted(api,tok,gid);
  await waitProcessInstancesRunning(api,tok,pid);
  if(env.APP_PING_URL){
    try{await fetch(env.APP_PING_URL,{method:"GET"});console.log("[ping] ok")}
    catch(e){console.log("[ping] fail",e?.message||e)}
  }
  await env.START_LOCK.put(lockKey,"1",{expirationTtl:3600});
  console.log("[lock] set",lockKey)
}

async function runIfInSchedule(env){
  const now=new Date();
  const utcH=now.getUTCHours();
  const utcM=now.getUTCMinutes();
  const allowedUtcHour=0;
  if(utcH===allowedUtcHour && utcM%2===0){
    console.log(`[cron] hit ${pad(utcH)}:${pad(utcM)} UTC (Beijing 08:${pad(utcM)}) → ensureRunning`);
    await ensureRunning(env,{reason:"cron"})
  }else{
    console.log(`[cron] skip at ${pad(utcH)}:${pad(utcM)} UTC`)
  }
}

export default{
  async scheduled(e,env,ctx){ctx.waitUntil(runIfInSchedule(env))},
  async fetch(req,env,ctx){
    const u=new URL(req.url);
    try{
      if(u.pathname==="/start"){
        const f=u.searchParams.get("force")==="1";
        ctx.waitUntil(ensureRunning(env,{reason:"manual",force:f}));
        return json({ok:!0,msg:"start requested",force:f})
      }
      if(u.pathname==="/stop"){
        const t=await getUAAToken(env);
        const a=env.CF_API.replace(/\/+$/,"");
        const g=await resolveAppGuid(env,t,a);
        await cfPOST(`${a}/v3/apps/${g}/actions/stop`,t);
        return json({ok:!0,msg:"stop requested"})
      }
      if(u.pathname==="/state"){
        const t=await getUAAToken(env);
        const a=env.CF_API.replace(/\/+$/,"");
        const g=await resolveAppGuid(env,t,a);
        const s=await getAppState(a,t,g);
        const p=await getWebProcessGuid(a,t,g).catch(()=>null);
        const st=p?await getProcessStats(a,t,p):null;
        return json({ok:!0,appGuid:g,appState:s,instances:(st?.resources||[]).map(it=>({index:it?.index,state:it?.state}))})
      }
      if(u.pathname==="/diag"){
        const t=await getUAAToken(env);
        const a=env.CF_API.replace(/\/+$/,"");
        return json({ok:!0,token_len:t?.length||0,api:a})
      }
      if(u.pathname==="/unlock"){
        const y=new Date().toISOString().slice(0,10);
        const k=`start-success-lock:${y}`;
        await env.START_LOCK.delete(k);
        return json({ok:!0,deleted:k})
      }
      return new Response("Worker up")
    }catch(err){
      console.error("[diag-error]",err?.message||err);
      return json({ok:!1,error:String(err)},500)
    }
  }
};
