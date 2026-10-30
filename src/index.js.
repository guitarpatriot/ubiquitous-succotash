
// Coinbase Cloud Bot — simulation only, v4
// Never places real Coinbase orders or transfers.

const OLD='paper-trading-state-v1';
const PAPER='paper-trading-state-v4';
const REPORT='scanner-v4-report';
const CONTROL='scanner-v4-control';
const FEE=0.009;
const PAPER_BUY_MAX=4;

const PAIRS=[
  'BTC-USD','ETH-USD','SOL-USD','SUI-USD',
  'AVAX-USD','XRP-USD','DOGE-USD','ADA-USD',
  'LINK-USD','LTC-USD','BCH-USD','UNI-USD',
  'AAVE-USD','ICP-USD','NEAR-USD','ARB-USD',
  'OP-USD','INJ-USD','FIL-USD','ATOM-USD',
  'FET-USD','ONDO-USD','ENA-USD','HYPE-USD',
  'WLD-USD','TAO-USD','PEPE-USD','ZEC-USD',
  'HBAR-USD','SHIB-USD'
];

const headers={
  'cache-control':'no-store',
  'x-content-type-options':'nosniff',
  'referrer-policy':'no-referrer'
};

const val=x=>
  Number.isFinite(Number(x))?Number(x):0;

const fixed=(x,n=6)=>
  Number(val(x).toFixed(n));

const now=()=>new Date().toISOString();

const wait=ms=>
  new Promise(resolve=>setTimeout(resolve,ms));

const kvReady=env=>
  !!env.PAPER_KV &&
  typeof env.PAPER_KV.get==='function';

const j=(data,status=200)=>
  new Response(JSON.stringify(data,null,2),{
    status,
    headers:{
      ...headers,
      'content-type':
        'application/json; charset=utf-8'
    }
  });

async function read(env,key,fallback=null){
  const v=await env.PAPER_KV.get(key,'json');
  return v===null?fallback:v;
}

async function write(env,key,value){
  await env.PAPER_KV.put(
    key,JSON.stringify(value)
  );
}

function equal(a,b){
  const aa=new TextEncoder().encode(a);
  const bb=new TextEncoder().encode(b);
  let d=aa.length^bb.length;

  for(let i=0;i<Math.max(
    aa.length,bb.length
  );i++){
    d|=(aa[i]||0)^(bb[i]||0);
  }
  return d===0;
}

function authorized(req,env){
  const h=req.headers.get('authorization')||'';

  if(
    !env.BOT_PASSWORD ||
    !h.startsWith('Basic ')
  )return false;

  try{
    const text=atob(h.slice(6));
    const sep=text.indexOf(':');

    return sep>=0 &&
      equal(text.slice(0,sep),'owner') &&
      equal(
        text.slice(sep+1),
        env.BOT_PASSWORD
      );
  }catch{
    return false;
  }
}

function login(){
  return new Response(
    'Sign in as owner.',
    {
      status:401,
      headers:{
        ...headers,
        'www-authenticate':
          'Basic realm="Coinbase Paper Bot"'
      }
    }
  );
}

async function exchange(pair,endpoint){
  const url=
    'https://api.exchange.coinbase.com/products/'+
    encodeURIComponent(pair)+'/'+endpoint;

  const resp=await fetch(url,{
    headers:{accept:'application/json'}
  });

  if(resp.status===429){
    const e=new Error('Coinbase HTTP 429');
    e.rateLimit=true;
    throw e;
  }

  if(!resp.ok){
    throw new Error(
      'Coinbase '+pair+' HTTP '+resp.status
    );
  }

  return resp.json();
}

async function candles(pair){
  const data=await exchange(
    pair,'candles?granularity=900'
  );

  if(!Array.isArray(data)){
    throw new Error('Invalid candles');
  }

  const cutoff=
    Math.floor(Date.now()/900000)*900;

  return data
    .filter(x=>
      Array.isArray(x)&&x.length>=6
    )
    .map(x=>({
      t:val(x[0]),
      high:val(x[2]),
      low:val(x[1]),
      close:val(x[4]),
      volume:val(x[5])
    }))
    .filter(x=>
      x.t<cutoff &&
      x.t>0 &&
      x.close>0 &&
      x.volume>0
    )
    .sort((a,b)=>a.t-b.t)
    .slice(-45);
}

async function quote(pair){
  const x=await exchange(pair,'ticker');

  const bid=val(x.bid);
  const ask=val(x.ask);
  const t=Date.parse(x.time||'');

  if(!(
    bid>0 &&
    ask>=bid &&
    Number.isFinite(t) &&
    Math.abs(Date.now()-t)<180000
  )){
    throw new Error('Missing or stale quote');
  }

  return {
    bid,ask,
    at:new Date(t).toISOString(),
    spreadPct:fixed(
      100*(ask-bid)/ask,3
    ),
    approx24hUsd:fixed(
      val(x.volume)*val(x.price),2
    )
  };
}

function analyze(pair,c){
  const last=c.at(-1);

  const cutoff=
    Math.floor(Date.now()/900000)*900;

  if(
    c.length<26 ||
    !last ||
    cutoff-last.t>2700
  ){
    return {
      pair,
      signal:'HOLD',
      reason:
        'Insufficient or stale completed candles'
    };
  }

  const prev=c.slice(-21,-1);

  const resistance=Math.max(
    ...prev.map(x=>x.high)
  );

  const mean=
    prev.reduce(
      (s,x)=>s+x.volume,0
    )/prev.length;

  const v=last.volume/
    Math.max(mean,1e-12);

  const m15=100*(
    last.close/c.at(-2).close-1
  );

  const m1h=100*(
    last.close/c.at(-5).close-1
  );

  const m4h=100*(
    last.close/c.at(-17).close-1
  );

  const breakout=
    last.close>resistance*1.001;

  const closeHigh=
    (last.close-last.low)/
    Math.max(
      last.high-last.low,1e-12
    )>0.65;

  const candidate=
    breakout &&
    v>=1.8 &&
    m15>0 &&
    m1h>=1.5 &&
    m1h<=12 &&
    m4h>0 &&
    closeHigh;

  return {
    pair,
    signal:'HOLD',
    candidate,
    candleAt:new Date(
      last.t*1000
    ).toISOString(),
    candleClose:last.close,
    resistance:fixed(resistance,9),
    volumeRatio:fixed(v,2),
    momentum15mPct:fixed(m15,2),
    momentum1hPct:fixed(m1h,2),
    momentum4hPct:fixed(m4h,2),
    score:fixed(
      Math.min(Math.max(m1h,0),12)+
      Math.min(v,8)*2+
      (breakout?6:0),
      2
    ),
    reason:candidate
      ?'Checking quote and fees'
      :'Breakout conditions not met'
  };
}

function rankQuote(s,q){
  const entry=q.ask;
  const stop=entry*0.97;
  const target=entry*1.095;

  const netUp=100*(
    target*(1-FEE)/
    (entry*(1+FEE))-1
  );

  const netDown=100*(
    1-stop*(1-FEE)/
    (entry*(1+FEE))
  );

  const rr=netUp/netDown;

  const accepted=
    q.spreadPct<=0.3 &&
    q.approx24hUsd>=1000000 &&
    entry<=s.candleClose*1.02 &&
    rr>=1.5;

  const {candidate,...rest}=s;

  return {
    ...rest,
    quote:q,
    signal:accepted?'WATCH_BUY':'HOLD',
    entry:accepted?fixed(entry,9):null,
    stop:accepted?fixed(stop,9):null,
    target:accepted?fixed(target,9):null,
    netTargetPct:fixed(netUp,2),
    netStopLossPct:fixed(netDown,2),
    rewardRisk:fixed(rr,2),
    reason:accepted
      ?'Research signal; verify retail availability'
      :'Quote, spread or reward/risk failed'
  };
}

// Read the OLD paper portfolio without modifying it.
function convert(old){
  const x=old&&typeof old==='object'
    ?old:{};

  const open=
    x.positions &&
    typeof x.positions==='object'
      ?Object.keys(x.positions).length
      :0;

  const history=
    Array.isArray(x.trades)
      ?x.trades:[];

  const cash=x.cashUsd==null
    ?34.07:val(x.cashUsd);

  return {
    version:4,
    mode:'PAPER_TRADING_ONLY',
    tradingEnabled:false,
    paused:open>0,
    warning:open
      ?'Legacy open positions need manual migration'
      :null,
    startingCashUsd:
      val(x.startingCashUsd)||34.07,
    cashUsd:cash,
    positions:[],
    estimatedEquityUsd:
      open?null:cash,
    realizedProfitUsd:
      val(x.realizedProfitUsd),
    unrealizedProfitUsd:0,
    feesPaidUsd:val(x.feesPaidUsd),
    tradeCount:history.length,
    dailyBuys:{
      date:now().slice(0,10),
      count:0
    },
    lastSignal:'',
    recentTrades:
      history.slice(-40).reverse()
        .map(t=>({
          time:t.time,
          side:t.side,
          symbol:t.coin||t.symbol,
          price:t.price,
          quantity:t.amount,
          totalUsd:
            t.budgetUsd??t.proceedsUsd,
          feeUsd:t.feeUsd,
          legacy:true,
          virtual:true
        }))
  };
}

async function paper(env){
  const saved=await read(env,PAPER);

  return saved??convert(
    await read(env,OLD)
  );
}

async function paperTick(env,scan){
  const state=await paper(env);

  if(state.paused)return;

  const held=
    Array.isArray(state.positions)
      ?state.positions:[];

  const remaining=[];
  const made=[];

  let cash=val(state.cashUsd);
  let quotesCurrent=true;

  for(const p of held){
    let q;

    try{
      q=await quote(p.pair);
    }catch(e){
      if(e.rateLimit)throw e;

      quotesCurrent=false;
      remaining.push(p);
      continue;
    }

    p.lastBid=q.bid;
    p.lastQuoteAt=q.at;

    if(
      q.bid<=p.stop ||
      q.bid>=p.target
    ){
      const fee=p.qty*q.bid*FEE;
      const proceeds=p.qty*q.bid-fee;

      cash+=proceeds;

      state.realizedProfitUsd=fixed(
        val(state.realizedProfitUsd)+
        proceeds-p.costUsd,
        8
      );

      state.feesPaidUsd=fixed(
        val(state.feesPaidUsd)+fee,
        8
      );

      made.push({
        time:now(),
        side:'PAPER_SELL',
        symbol:p.pair,
        price:q.bid,
        quantity:p.qty,
        totalUsd:fixed(proceeds,8),
        feeUsd:fixed(fee,8),
        virtual:true
      });
    }else{
      remaining.push(p);
    }

    await wait(350);
  }

  const equity=
    cash+remaining.reduce(
      (s,p)=>s+p.qty*val(p.lastBid),
      0
    );

  const today=now().slice(0,10);

  if(state.dailyBuys?.date!==today){
    state.dailyBuys={
      date:today,
      count:0
    };
  }

  const best=scan.ranking.find(s=>
    s.signal==='WATCH_BUY' &&
    !remaining.some(p=>p.pair===s.pair)
  );

  if(
    best &&
    quotesCurrent &&
    remaining.length<3 &&
    state.dailyBuys.count<2 &&
    state.lastSignal!==
      best.pair+best.candleAt
  ){
    const spend=Math.floor(
      Math.min(
        PAPER_BUY_MAX,
        equity*0.1,
        cash
      )*100
    )/100;

    if(spend>=1){
      const qty=spend/(
        best.entry*(1+FEE)
      );

      const fee=qty*best.entry*FEE;

      cash-=spend;

      remaining.push({
        pair:best.pair,
        qty,
        costUsd:spend,
        entry:best.entry,
        stop:best.stop,
        target:best.target,
        lastBid:best.quote.bid,
        lastQuoteAt:best.quote.at
      });

      state.feesPaidUsd=fixed(
        val(state.feesPaidUsd)+fee,
        8
      );

      state.dailyBuys.count++;

      state.lastSignal=
        best.pair+best.candleAt;

      made.push({
        time:now(),
        side:'PAPER_BUY',
        symbol:best.pair,
        price:best.entry,
        quantity:qty,
        totalUsd:spend,
        feeUsd:fixed(fee,8),
        virtual:true
      });
    }
  }

  state.cashUsd=fixed(cash,8);
  state.positions=remaining;

  const validMarks=
    quotesCurrent &&
    remaining.every(p=>
      p.lastBid>0 &&
      Date.now()-Date.parse(
        p.lastQuoteAt||''
      )<1800000
    );

  state.estimatedEquityUsd=
    validMarks
      ?fixed(
        cash+remaining.reduce(
          (s,p)=>s+p.qty*p.lastBid,
          0
        ),8
      )
      :null;

  state.unrealizedProfitUsd=
    validMarks
      ?fixed(
        remaining.reduce(
          (s,p)=>
            s+p.qty*p.lastBid*(1-FEE)-
            p.costUsd,
          0
        ),8
      )
      :null;

  state.tradeCount=
    val(state.tradeCount)+made.length;

  state.recentTrades=[
    ...made,
    ...(state.recentTrades||[])
  ].slice(0,100);

  // Separate key; legacy records stay intact.
  await write(env,PAPER,state);
}

async function scan(env){
  const control=await read(
    env,CONTROL,{}
  );

  if(
    val(control.cooldownUntil)>
    Date.now()
  ){
    return {
      ...(await read(
        env,REPORT,{ranking:[]}
      )),
      cached:true,
      status:'COOLDOWN',
      nextTry:new Date(
        control.cooldownUntil
      ).toISOString()
    };
  }

  if(
    Date.now()-val(control.at)<
    14*60000
  ){
    return {
      ...(await read(
        env,REPORT,{ranking:[]}
      )),
      cached:true
    };
  }

  const batch=(
    val(control.nextBatch)%5+5
  )%5;

  const selected=PAIRS.slice(
    batch*6,
    batch*6+6
  );

  const ranking=[];
  const errors=[];
  let limited=false;

  for(const pair of selected){
    try{
      let s=analyze(
        pair,
        await candles(pair)
      );

      if(s.candidate){
        s=rankQuote(
          s,
          await quote(pair)
        );
      }else{
        delete s.candidate;
      }

      ranking.push(s);
    }catch(e){
      errors.push(
        pair+': '+String(
          e.message||e
        )
      );

      if(e.rateLimit){
        limited=true;
        break;
      }
    }

    await wait(400);
  }

  ranking.sort((a,b)=>
    Number(b.signal==='WATCH_BUY')-
    Number(a.signal==='WATCH_BUY') ||
    val(b.score)-val(a.score)
  );

  const result={
    mode:'SIMULATION_ONLY',
    tradingEnabled:false,
    scannedAt:now(),
    requested:selected.length,
    completed:ranking.length,
    universeSize:PAIRS.length,
    ranking,
    errors,
    note:
      'Public Exchange data; retail trading eligibility not guaranteed'
  };

  await write(env,REPORT,result);

  const next={
    at:Date.now(),
    nextBatch:limited
      ?batch
      :(batch+1)%5,
    cooldownUntil:limited
      ?Date.now()+30*60000
      :0
  };

  await write(env,CONTROL,next);

  if(!limited){
    try{
      await paperTick(env,result);
    }catch(e){
      result.paperError=
        String(e.message||e);

      if(e.rateLimit){
        next.cooldownUntil=
          Date.now()+30*60000;

        await write(
          env,CONTROL,next
        );
      }

      await write(
        env,REPORT,result
      );
    }
  }

  return result;
}

function dashboard(){
  return `<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport"
 content="width=device-width,initial-scale=1">
<title>Coinbase Paper Bot</title>
<style>
body{
  font:15px system-ui;
  background:#0d1526;
  color:#eef3ff;
  margin:0;
  padding:16px
}
main{max-width:900px;margin:auto}
.card{
  background:#1b2942;
  border-radius:12px;
  padding:15px;
  margin:12px 0
}
button{
  padding:12px;
  background:#94bdff;
  color:#10233d;
  border:0;
  border-radius:8px;
  font-weight:700
}
table{
  border-collapse:collapse;
  width:100%;
  font-size:13px
}
td,th{
  padding:8px;
  border-bottom:1px solid #34465f;
  text-align:left
}
.scroller{overflow:auto}
small{color:#b4c5d9}
</style>
</head>
<body>
<main>
<h1>Coinbase Cloud Bot</h1>
<p>
SIMULATION ONLY — No real trades or transfers
</p>

<div class="card" id="status">
Loading...
</div>

<div class="card" id="money"></div>

<div class="card">
<button id="scan">Scan Now</button>
<button id="refresh">Refresh</button>
<p id="message"></p>
</div>

<div class="card scroller">
<h2>Momentum signals</h2>
<div id="rows"></div>
</div>

<div class="card">
<h2>Virtual trades</h2>
<div id="trades"></div>
</div>

<small>
Rotating six of 30 public Coinbase Exchange
USD pairs per scan. Completed 15-minute
candles; assumed spot fees 0.9% each side.
Public quotes do not guarantee retail fills.
</small>
</main>

<script>
const escapeHTML=x=>
  String(x??'').replace(
    /[&<>"']/g,
    c=>({
      '&':'&amp;',
      '<':'&lt;',
      '>':'&gt;',
      '"':'&quot;',
      "'":'&#39;'
    }[c])
  );

async function get(path){
  const r=await fetch(path,{
    cache:'no-store'
  });

  if(!r.ok){
    throw Error(
      path+': HTTP '+r.status
    );
  }

  return r.json();
}

async function refresh(){
  try{
    const [h,p,s]=await Promise.all([
      get('/health'),
      get('/paper'),
      get('/signals')
    ]);

    document.querySelector(
      '#status'
    ).textContent=
      'KV: '+
      (h.kvReady?'Connected':'MISSING')+
      ' | Mode: '+h.mode+
      ' | Last scan: '+
      (s.scannedAt||'No scan yet')+
      (s.status==='COOLDOWN'
        ?' | Cooling down':''
      );

    document.querySelector(
      '#money'
    ).textContent=
      'Virtual cash: $'+p.cashUsd+
      ' | Equity: '+
      (p.estimatedEquityUsd===null
        ?'Unavailable'
        :'$'+p.estimatedEquityUsd
      )+
      ' | Virtual trades: '+
      p.tradeCount+
      (p.paused
        ?' | PAUSED: '+p.warning
        :''
      );

    document.querySelector(
      '#rows'
    ).innerHTML=
      '<table><tr>'+
      '<th>Pair</th>'+
      '<th>Signal</th>'+
      '<th>1h%</th>'+
      '<th>Vol×</th>'+
      '<th>Quote</th></tr>'+
      (s.ranking||[]).map(x=>
        '<tr><td>'+
        escapeHTML(x.pair)+
        '</td><td>'+
        escapeHTML(x.signal)+
        '</td><td>'+
        escapeHTML(x.momentum1hPct)+
        '</td><td>'+
        escapeHTML(x.volumeRatio)+
        '</td><td>'+
        escapeHTML(x.quote?.ask||'—')+
        '</td></tr>'
      ).join('')+
      '</table><small>'+
      escapeHTML(
        (s.errors||[]).join('; ')
      )+
      '</small>';

    document.querySelector(
      '#trades'
    ).innerHTML=
      (p.recentTrades||[])
        .slice(0,12)
        .map(t=>
          '<p>'+
          escapeHTML(t.time)+
          ' | '+
          escapeHTML(t.side)+
          ' '+
          escapeHTML(t.symbol)+
          ' | $'+
          escapeHTML(t.totalUsd)+
          '</p>'
        ).join('')||
      'No virtual trades';

  }catch(e){
    document.querySelector(
      '#status'
    ).textContent=
      'Error: '+e.message;
  }
}

document.querySelector(
  '#refresh'
).onclick=refresh;

document.querySelector(
  '#scan'
).onclick=async()=>{
  const b=document.querySelector(
    '#scan'
  );

  b.disabled=true;

  document.querySelector(
    '#message'
  ).textContent='Requesting scan...';

  try{
    const r=await fetch(
      '/scan-now',{
        method:'POST',
        headers:{
          'x-requested-with':
            'CoinbasePaperDashboard'
        }
      }
    );

    const x=await r.json();

    document.querySelector(
      '#message'
    ).textContent=x.cached
      ?'Recent scan reused.'
      :'Scan request completed.';

    await refresh();
  }catch(e){
    document.querySelector(
      '#message'
    ).textContent=e.message;
  }finally{
    b.disabled=false;
  }
};

refresh();
</script>
</body>
</html>`;
}

export default {
  async fetch(request,env){
    const path=new URL(
      request.url
    ).pathname;

    if(path==='/health'){
      return j({
        status:'online',
        mode:'SIMULATION_ONLY',
        tradingEnabled:false,
        kvReady:kvReady(env),
        passwordConfigured:
          !!env.BOT_PASSWORD
      });
    }

    if(!authorized(request,env)){
      return login();
    }

    if(!kvReady(env)){
      return j({
        error:'PAPER_KV binding missing'
      },503);
    }

    if(
      path==='/' ||
      path==='/dashboard'
    ){
      return new Response(
        dashboard(),{
          headers:{
            ...headers,
            'content-type':
              'text/html; charset=utf-8',
            'content-security-policy':
              "default-src 'none';"+
              "script-src 'unsafe-inline';"+
              "style-src 'unsafe-inline';"+
              "connect-src 'self';"+
              "frame-ancestors 'none'"
          }
        }
      );
    }

    if(path==='/paper'){
      return j(await paper(env));
    }

    if(path==='/signals'){
      return j(
        await read(
          env,REPORT,{
            mode:'SIMULATION_ONLY',
            ranking:[],
            note:'No scan yet'
          }
        )
      );
    }

    if(
      path==='/scan-now' &&
      request.method==='POST'
    ){
      const origin=
        request.headers.get('origin');

      if(
        (
          origin &&
          origin!==
            new URL(request.url).origin
        ) ||
        request.headers.get(
          'x-requested-with'
        )!=='CoinbasePaperDashboard'
      ){
        return j({
          error:'Request blocked'
        },403);
      }

      return j(await scan(env));
    }

    if(path==='/coinbase-test'){
      return j({
        connected:false,
        mode:'SIMULATION_ONLY',
        tradingEnabled:false,
        note:
          'Public data only; no Coinbase account credentials used'
      });
    }

    if(path==='/analyze'){
      return j({
        mode:'SIMULATION_ONLY',
        note:
          'See /signals and /paper; no live account access'
      });
    }

    return j({
      error:'Not found'
    },404);
  },

  async scheduled(event,env,ctx){
    ctx.waitUntil(
      scan(env)
        .then(r=>console.log(
          'Scheduled scan',
          r.scannedAt||
          r.status||
          r.cached
        ))
        .catch(e=>console.log(
          'Scan error',
          String(e)
        ))
    );
  }
};
