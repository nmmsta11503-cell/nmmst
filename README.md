<!DOCTYPE html>
<html lang="zh-TW">
<head>
<meta charset="UTF-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1.0"/>
<title>AI 提示詞共享平台 · 國立海洋科技博物館</title>
<meta name="description" content="國立海洋科技博物館 AI 提示詞及生成結果分享平台"/>
<link rel="icon" href="data:image/svg+xml,<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 100 100'><text y='.9em' font-size='90'>🌊</text></svg>"/>
<link rel="preconnect" href="https://fonts.googleapis.com"/>
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin/>
<link href="https://fonts.googleapis.com/css2?family=Noto+Sans+TC:wght@300;400;500;700&family=Space+Grotesk:wght@400;500;600;700&display=swap" rel="stylesheet"/>
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.0/css/all.min.css"/>
<style>
CSSEOF
cat /home/claude/nmst-ai-platform/css/main.css >> /home/claude/nmst-ai-platform/index.html
cat >> /home/claude/nmst-ai-platform/index.html << 'HTMLEOF'
</style>
</head>
<body>
<div id="app">
  <div class="splash">
    <div class="splash-icon"><i class="fa-solid fa-water"></i></div>
    <p>載入中…</p>
  </div>
</div>
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/dist/umd/supabase.min.js"></script>
<script>
// =============================================
// js/config.js
// ★ 把下面兩個值換成你的 Supabase 設定 ★
// Supabase → Settings → API
// =============================================

const SUPABASE_URL  = 'https://oqennxszrjjcwoulovse.supabase.co';
const SUPABASE_KEY  = 'sb_publishable_82JqqtVTZxofUT5xMPuPYw_NCsr9ozg';

// 敏感詞清單（可自行增減）
const SENSITIVE_KEYWORDS = [
  '色情','裸露','情色','成人','AV','暴力','殺人','自殺','自殘',
  '毒品','大麻','海洛因','賭博','詐騙','違法','仇恨','歧視','恐怖'
];

// 人氣排行榜顯示幾筆
const RANK_LIMIT = 5;

// 預設每頁幾篇
const PAGE_SIZE = 20;
// =============================================
// js/supabase.js  — 所有資料庫操作封裝在這裡
// =============================================

const { createClient } = supabase;
const db = createClient(SUPABASE_URL, SUPABASE_KEY);

/* ---- Auth ---- */
async function signUp(email, password, name) {
  return db.auth.signUp({ email, password, options: { data: { name } } });
}

async function signIn(email, password) {
  return db.auth.signInWithPassword({ email, password });
}

async function signOut() {
  return db.auth.signOut();
}

async function getSession() {
  const { data } = await db.auth.getSession();
  return data.session;
}

async function getProfile(userId) {
  const { data } = await db.from('profiles').select('*').eq('id', userId).single();
  return data;
}

/* ---- Posts ---- */
async function fetchPosts({ sort = 'view_count', tag = null, search = null, status = 'approved' } = {}) {
  let q = db.from('posts')
    .select(`*, profiles(name, role)`)
    .eq('status', status)
    .order(sort, { ascending: false })
    .limit(PAGE_SIZE);

  if (tag)    q = q.contains('tags', [tag]);
  if (search) q = q.or(`title.ilike.%${search}%`);

  const { data, error } = await q;
  return { data: data || [], error };
}

async function fetchMyPosts(userId) {
  const { data, error } = await db.from('posts')
    .select(`*, profiles(name, role)`)
    .eq('user_id', userId)
    .order('created_at', { ascending: false });
  return { data: data || [], error };
}

async function fetchPendingPosts() {
  const { data, error } = await db.from('posts')
    .select(`*, profiles(name, role)`)
    .eq('status', 'pending')
    .order('created_at', { ascending: true });
  return { data: data || [], error };
}

async function fetchPostById(id) {
  const { data, error } = await db.from('posts')
    .select(`*, profiles(name, role), attachments(*)`)
    .eq('id', id)
    .single();
  return { data, error };
}

async function createPost(payload) {
  const { data, error } = await db.from('posts').insert(payload).select().single();
  return { data, error };
}

async function updatePostStatus(id, status) {
  const { error } = await db.from('posts').update({ status }).eq('id', id);
  return { error };
}

async function incrementViewCount(id) {
  await db.rpc('increment_view', { post_id: id }).catch(() => {
    // fallback: read then write
    db.from('posts').select('view_count').eq('id', id).single()
      .then(({ data }) => {
        if (data) db.from('posts').update({ view_count: data.view_count + 1 }).eq('id', id);
      });
  });
}

async function fetchTopPosts(limit = RANK_LIMIT) {
  const { data } = await db.from('posts')
    .select('id, title, view_count, like_count')
    .eq('status', 'approved')
    .order('view_count', { ascending: false })
    .limit(limit);
  return data || [];
}

async function fetchStats() {
  const [approved, pending, rejected, profiles] = await Promise.all([
    db.from('posts').select('id', { count: 'exact' }).eq('status', 'approved'),
    db.from('posts').select('id', { count: 'exact' }).eq('status', 'pending'),
    db.from('posts').select('id', { count: 'exact' }).eq('status', 'rejected'),
    db.from('profiles').select('id', { count: 'exact' }),
  ]);
  const views = await db.from('posts').select('view_count').eq('status', 'approved');
  const totalViews = (views.data || []).reduce((a, p) => a + (p.view_count || 0), 0);
  return {
    approved: approved.count || 0,
    pending:  pending.count  || 0,
    rejected: rejected.count || 0,
    members:  profiles.count || 0,
    totalViews,
  };
}

/* ---- Likes ---- */
async function toggleLike(postId, userId) {
  const { data: existing } = await db.from('likes')
    .select('id').eq('post_id', postId).eq('user_id', userId).single();
  if (existing) {
    await db.from('likes').delete().eq('id', existing.id);
    return false;
  } else {
    await db.from('likes').insert({ post_id: postId, user_id: userId });
    return true;
  }
}

async function fetchUserLikes(userId) {
  const { data } = await db.from('likes').select('post_id').eq('user_id', userId);
  return new Set((data || []).map(l => l.post_id));
}

/* ---- Comments ---- */
async function fetchComments(postId) {
  const { data } = await db.from('comments')
    .select(`*, profiles(name)`)
    .eq('post_id', postId)
    .order('created_at', { ascending: true });
  return data || [];
}

async function addComment(postId, userId, content) {
  const { data, error } = await db.from('comments')
    .insert({ post_id: postId, user_id: userId, content })
    .select(`*, profiles(name)`).single();
  return { data, error };
}

/* ---- Attachments / Storage ---- */
async function uploadFile(file, postId) {
  const ext  = file.name.split('.').pop().toLowerCase();
  const isImg = ['jpg','jpeg','png','gif','webp','bmp'].includes(ext);
  const finalName = isImg
    ? file.name.replace(/\.(jpg|jpeg|png|gif|bmp)$/i, '.webp')
    : file.name;
  const path = `${postId}/${Date.now()}_${finalName}`;

  // 若為圖片，嘗試轉 WebP（現代瀏覽器支援 canvas）
  let uploadBlob = file;
  if (isImg && ext !== 'webp') {
    try {
      uploadBlob = await convertToWebP(file);
    } catch (_) { /* 轉換失敗就用原檔 */ }
  }

  const { data, error } = await db.storage.from('attachments').upload(path, uploadBlob, {
    contentType: isImg ? 'image/webp' : file.type,
    upsert: true,
  });
  if (error) return { error };

  const { data: pub } = db.storage.from('attachments').getPublicUrl(path);
  const record = await db.from('attachments').insert({
    post_id: postId,
    file_name: finalName,
    file_url: pub.publicUrl,
    file_type: isImg ? 'image/webp' : file.type,
    file_size: uploadBlob.size,
    is_webp: isImg,
  }).select().single();

  return { data: record.data, error: null };
}

async function convertToWebP(file) {
  return new Promise((resolve, reject) => {
    const img = new Image();
    const url = URL.createObjectURL(file);
    img.onload = () => {
      const canvas = document.createElement('canvas');
      canvas.width = img.width; canvas.height = img.height;
      canvas.getContext('2d').drawImage(img, 0, 0);
      canvas.toBlob(blob => {
        URL.revokeObjectURL(url);
        blob ? resolve(blob) : reject(new Error('toBlob failed'));
      }, 'image/webp', 0.88);
    };
    img.onerror = reject;
    img.src = url;
  });
}

/* ---- Sensitive check (client-side fast check) ---- */
function detectSensitive(text) {
  const lower = (text || '').toLowerCase();
  return SENSITIVE_KEYWORDS.filter(k => lower.includes(k));
}
// =============================================
// js/render.js  — 所有 HTML 產生函式
// =============================================

/* ---- Helpers ---- */
function fmtNum(n) {
  if (!n) return '0';
  return n >= 1000 ? (n / 1000).toFixed(1) + 'k' : String(n);
}
function fmtDate(s) {
  if (!s) return '';
  return new Date(s).toLocaleDateString('zh-TW', { year: 'numeric', month: '2-digit', day: '2-digit' });
}
function initials(name) { return name ? name.slice(0, 1) : '?'; }
function avatarColor(str) {
  const colors = ['#0e4d8a','#007a6e','#8a5000','#6a1a1a','#4a1a8a'];
  let h = 0; for (let c of (str||'')) h = (h * 31 + c.charCodeAt(0)) & 0xffffffff;
  return colors[Math.abs(h) % colors.length];
}
function esc(s) { return String(s||'').replace(/</g,'&lt;').replace(/>/g,'&gt;').replace(/"/g,'&quot;'); }

/* ---- Header ---- */
function renderHeader(state) {
  if (!state.user) return '';
  const pending = (state.pendingCount || 0);
  return `
<header>
  <div class="header-inner">
    <a class="logo-area" href="#">
      ${state.logoSrc ? `<img class="logo-img" src="${state.logoSrc}" alt="NMST Logo">` :
        `<span style="font-size:24px;color:var(--ocean-bright)"><i class="fa-solid fa-water"></i></span>`}
      <div class="logo-divider"></div>
      <div class="logo-text">
        <div class="name">國立海洋科技博物館</div>
        <div class="sub">AI PROMPT PLATFORM</div>
      </div>
    </a>

    <nav class="nav">
      <button class="nav-btn ${state.page==='feed'?'active':''}" onclick="App.goto('feed')">
        <i class="fa-solid fa-compass"></i>探索
      </button>
      <button class="nav-btn ${state.page==='my'?'active':''}" onclick="App.goto('my')">
        <i class="fa-regular fa-bookmark"></i>我的分享
      </button>
      ${state.profile?.role === 'admin' ? `
      <button class="nav-btn ${state.page==='admin'?'active':''}" onclick="App.goto('admin')">
        <i class="fa-solid fa-shield-halved"></i>審核管理
        ${pending > 0 ? `<span class="badge-count">${pending}</span>` : ''}
      </button>` : ''}
    </nav>

    <div class="header-actions">
      <button class="btn btn-primary" onclick="App.openUpload()">
        <i class="fa-solid fa-plus"></i>分享提示詞
      </button>
      <div class="avatar" style="background:${avatarColor(state.profile?.name)}" onclick="App.openProfile()" title="${esc(state.profile?.name)}">
        ${initials(state.profile?.name)}
      </div>
    </div>
  </div>
</header>`;
}

/* ---- Login Page ---- */
function renderLogin(tab = 'login') {
  return `
<div class="login-page">
  <div class="login-card">
    <div class="login-logo">
      <div class="icon"><i class="fa-solid fa-water"></i></div>
      <h1>AI 提示詞共享平台</h1>
      <p>國立海洋科技博物館 · NMST</p>
    </div>
    <div class="login-tabs">
      <button class="login-tab ${tab==='login'?'active':''}" onclick="App.setLoginTab('login')">登入</button>
      <button class="login-tab ${tab==='register'?'active':''}" onclick="App.setLoginTab('register')">註冊</button>
    </div>
    ${tab === 'login' ? `
    <div class="form-group">
      <label class="form-label">電子郵件</label>
      <input id="li-email" type="email" class="form-input" placeholder="your@nmst.gov.tw" autocomplete="email">
    </div>
    <div class="form-group">
      <label class="form-label">密碼</label>
      <input id="li-pw" type="password" class="form-input" placeholder="••••••••" onkeydown="if(event.key==='Enter')App.doLogin()">
    </div>
    <button class="btn btn-primary" style="width:100%;justify-content:center;padding:10px" onclick="App.doLogin()">
      <i class="fa-solid fa-right-to-bracket"></i>登入
    </button>
    ` : `
    <div class="form-group">
      <label class="form-label">姓名</label>
      <input id="reg-name" type="text" class="form-input" placeholder="王小明">
    </div>
    <div class="form-group">
      <label class="form-label">電子郵件</label>
      <input id="reg-email" type="email" class="form-input" placeholder="your@nmst.gov.tw">
    </div>
    <div class="form-group">
      <label class="form-label">密碼（至少 6 位）</label>
      <input id="reg-pw" type="password" class="form-input" placeholder="••••••••">
    </div>
    <button class="btn btn-primary" style="width:100%;justify-content:center;padding:10px" onclick="App.doRegister()">
      <i class="fa-solid fa-user-plus"></i>建立帳號
    </button>
    `}
  </div>
</div>`;
}

/* ---- Post Card ---- */
function renderPostCard(post, opts = {}) {
  const name  = post.profiles?.name || '未知用戶';
  const pct   = opts.maxViews ? Math.round((post.view_count / opts.maxViews) * 100) : 0;
  const liked = opts.likedSet?.has(post.id);
  const thumb = (post.attachments || []).find(a => a.is_webp);
  const showStatus = opts.showStatus;

  return `
<div class="post-card" onclick="App.openDetail('${post.id}')">
  <div class="post-card-head">
    <div class="avatar" style="width:36px;height:36px;font-size:13px;background:${avatarColor(name)};border:2px solid var(--border-subtle)">
      ${initials(name)}
    </div>
    <div class="post-author-info">
      <div class="post-author-name">${esc(name)}</div>
      <div class="post-author-time"><i class="fa-regular fa-clock" style="font-size:10px"></i> ${fmtDate(post.created_at)}</div>
    </div>
    ${showStatus ? `
    <span class="badge badge-${post.status}" style="margin-left:auto">
      ${post.status==='approved'?'已上架':post.status==='pending'?'待審核':'已拒絕'}
    </span>` : ''}
    <span class="badge ${post.content_type==='image'?'badge-image':'badge-text'}" ${showStatus?'':'style="margin-left:auto"'}>
      ${post.content_type==='image'?'<i class="fa-regular fa-image"></i> 圖片':'<i class="fa-solid fa-align-left"></i> 文字'}
    </span>
  </div>

  <div class="post-title">${esc(post.title)}</div>

  <div class="post-tags">
    ${(post.tags||[]).map((t,i)=>`<span class="tag ${i%2===1?'tag-teal':''}"># ${esc(t)}</span>`).join('')}
  </div>

  <div class="post-prompt-preview">
    <span style="color:var(--text-muted);font-size:11px;margin-right:5px">提示詞：</span>${esc(post.prompt)}
  </div>

  ${thumb ? `
  <div class="post-img-thumb">
    <img src="${esc(thumb.file_url)}" alt="生成圖片" loading="lazy">
  </div>` : ''}

  <div class="post-footer">
    <button class="post-action ${liked?'liked':''}" onclick="event.stopPropagation();App.toggleLike('${post.id}')">
      <i class="${liked?'fa-solid':'fa-regular'} fa-heart"></i> ${fmtNum(post.like_count)}
    </button>
    <button class="post-action">
      <i class="fa-regular fa-comment"></i> ${fmtNum(post.comment_count)}
    </button>
    <button class="post-action">
      <i class="fa-regular fa-eye"></i> ${fmtNum(post.view_count)}
    </button>
    <div class="popularity-bar">
      <i class="fa-solid fa-fire fire" style="color:var(--coral)"></i>
      <div class="bar-track"><div class="bar-fill" style="width:${pct}%"></div></div>
      <span class="bar-pct">${pct}%</span>
    </div>
  </div>
</div>`;
}

/* ---- Feed Page ---- */
function renderFeed(state) {
  const { posts = [], sortBy = 'view_count', filterTag, searchQ = '', stats = {}, topPosts = [], likedSet, profile } = state;
  const maxViews = Math.max(...posts.map(p => p.view_count || 0), 1);

  const allTags = [...new Set(posts.flatMap(p => p.tags || []))];

  const typeFilters = [
    { key: null,    label: '全部', icon: 'fa-border-all' },
    { key: 'image', label: '圖像', icon: 'fa-image' },
    { key: 'text',  label: '文字', icon: 'fa-align-left' },
  ];

  const toolFilters = ['ChatGPT','Midjourney','DALL-E','Claude','Stable Diffusion','Gemini'];

  return `
<div class="page">
  <div class="feed-layout">

    <!-- Left sidebar -->
    <aside class="sidebar left-col">
      <div class="sidebar-section-title">瀏覽分類</div>
      ${typeFilters.map(f=>`
      <div class="sidebar-item ${state.filterType===f.key?'active':''}" onclick="App.setFilter('type','${f.key}')">
        <i class="fa-solid ${f.icon}"></i>${f.label}
        ${f.key===null?`<span class="sc">${stats.approved||0}</span>`:''}
      </div>`).join('')}
      <div class="sidebar-sep"></div>
      <div class="sidebar-section-title">AI 工具</div>
      ${toolFilters.map(t=>`
      <div class="sidebar-item ${filterTag===t?'active':''}" onclick="App.setFilter('tag','${t}')">
        <i class="fa-solid fa-robot"></i>${t}
      </div>`).join('')}
      ${allTags.filter(t=>!toolFilters.includes(t)).slice(0,6).map(t=>`
      <div class="sidebar-item ${filterTag===t?'active':''}" onclick="App.setFilter('tag','${t}')">
        <i class="fa-solid fa-hashtag"></i>${esc(t)}
      </div>`).join('')}
    </aside>

    <!-- Main feed -->
    <div>
      <div class="feed-topbar">
        <h2>${filterTag ? `# ${esc(filterTag)}` : '探索提示詞'}</h2>
        <div class="sort-pills">
          <button class="sort-pill ${sortBy==='view_count'?'active':''}" onclick="App.setSort('view_count')"><i class="fa-solid fa-fire"></i>人氣</button>
          <button class="sort-pill ${sortBy==='created_at'?'active':''}" onclick="App.setSort('created_at')"><i class="fa-regular fa-clock"></i>最新</button>
          <button class="sort-pill ${sortBy==='like_count'?'active':''}" onclick="App.setSort('like_count')"><i class="fa-regular fa-heart"></i>讚數</button>
        </div>
      </div>

      <div class="search-bar">
        <i class="fa-solid fa-magnifying-glass"></i>
        <input type="text" placeholder="搜尋標題或標籤…" value="${esc(searchQ)}"
          oninput="App.setSearch(this.value)" />
        ${searchQ ? `<button onclick="App.setSearch('')" class="btn-icon" style="width:22px;height:22px;font-size:12px;border:none"><i class="fa-solid fa-xmark"></i></button>` : ''}
      </div>

      ${state.loading
        ? `<div class="loader"><i class="fa-solid fa-spinner fa-spin"></i> 載入中…</div>`
        : posts.length === 0
          ? `<div class="empty-state"><i class="fa-solid fa-water"></i><p>尚無符合的內容</p><small>試試其他關鍵字或標籤</small></div>`
          : posts.map(p => renderPostCard(p, { maxViews, likedSet, showStatus: false })).join('')
      }
    </div>

    <!-- Right sidebar -->
    <aside class="right-col">
      <div class="widget-card">
        <div class="widget-title"><i class="fa-solid fa-fire"></i>人氣排行榜</div>
        ${topPosts.length === 0 ? '<div class="text-muted text-sm">暫無資料</div>' :
          topPosts.map((p,i)=>`
          <div class="rank-item" onclick="App.openDetail('${p.id}')">
            <div class="rank-no ${i<3?'top':''}">${i+1}</div>
            <span class="rank-title" title="${esc(p.title)}">${esc(p.title)}</span>
            <span class="rank-views"><i class="fa-regular fa-eye" style="font-size:10px"></i> ${fmtNum(p.view_count)}</span>
          </div>`).join('')}
      </div>

      <div class="widget-card">
        <div class="widget-title"><i class="fa-solid fa-chart-bar"></i>平台統計</div>
        <div class="stat-mini-grid">
          <div class="stat-mini"><div class="val val-teal">${stats.approved||0}</div><div class="lbl">已上架</div></div>
          <div class="stat-mini"><div class="val val-blue">${stats.members||0}</div><div class="lbl">會員</div></div>
          <div class="stat-mini"><div class="val val-gold">${stats.pending||0}</div><div class="lbl">待審核</div></div>
          <div class="stat-mini"><div class="val val-coral">${fmtNum(stats.totalViews||0)}</div><div class="lbl">總瀏覽</div></div>
        </div>
      </div>

      <div class="widget-card">
        <div class="widget-title"><i class="fa-solid fa-lightbulb"></i>提示詞技巧</div>
        <div style="font-size:13px;color:var(--text-secondary);line-height:1.9">
          <div><span style="color:var(--ocean-teal)">01</span>　明確指定風格與媒材</div>
          <div><span style="color:var(--ocean-teal)">02</span>　加入情境與氛圍描述</div>
          <div><span style="color:var(--ocean-teal)">03</span>　指定比例或解析度</div>
          <div><span style="color:var(--ocean-teal)">04</span>　英文提示通常效果更好</div>
          <div><span style="color:var(--ocean-teal)">05</span>　善用負向提示詞排除</div>
        </div>
      </div>
    </aside>
  </div>
</div>`;
}

/* ---- My Posts Page ---- */
function renderMyPage(state) {
  const { myPosts = [], likedSet } = state;
  const maxViews = Math.max(...myPosts.map(p => p.view_count || 0), 1);
  return `
<div class="page">
  <div class="feed-topbar" style="margin-bottom:1.25rem">
    <div>
      <h2>我的分享</h2>
      <p class="text-muted text-sm" style="margin-top:4px">共 ${myPosts.length} 篇</p>
    </div>
    <button class="btn btn-primary" onclick="App.openUpload()">
      <i class="fa-solid fa-plus"></i>新增分享
    </button>
  </div>
  ${state.loading
    ? `<div class="loader"><i class="fa-solid fa-spinner fa-spin"></i> 載入中…</div>`
    : myPosts.length === 0
      ? `<div class="empty-state"><i class="fa-solid fa-pen-to-square"></i><p>尚未分享任何提示詞</p><small>點擊「分享提示詞」開始</small></div>`
      : myPosts.map(p => renderPostCard(p, { maxViews, likedSet, showStatus: true })).join('')
  }
</div>`;
}

/* ---- Admin Page ---- */
function renderAdmin(state) {
  const { adminPosts = [], adminTab = 'pending', adminStats = {} } = state;
  return `
<div class="page">
  <h2 style="margin-bottom:1.25rem;font-size:20px">審核管理後台</h2>
  <div class="admin-stats">
    ${[
      { label:'待審核', val: adminStats.pending||0, cls:'val-gold' },
      { label:'已上架', val: adminStats.approved||0, cls:'val-teal' },
      { label:'已拒絕', val: adminStats.rejected||0, cls:'val-coral' },
      { label:'會員人數', val: adminStats.members||0, cls:'val-blue' },
    ].map(s=>`
    <div class="admin-stat-card">
      <div class="asc-label">${s.label}</div>
      <div class="asc-val ${s.cls}">${s.val}</div>
    </div>`).join('')}
  </div>

  <div class="sort-pills" style="margin-bottom:1.25rem">
    <button class="sort-pill ${adminTab==='pending'?'active':''}" onclick="App.setAdminTab('pending')">
      <i class="fa-regular fa-clock"></i>待審核 ${adminStats.pending>0?`(${adminStats.pending})`:''}
    </button>
    <button class="sort-pill ${adminTab==='approved'?'active':''}" onclick="App.setAdminTab('approved')">
      <i class="fa-solid fa-check"></i>已上架
    </button>
    <button class="sort-pill ${adminTab==='rejected'?'active':''}" onclick="App.setAdminTab('rejected')">
      <i class="fa-solid fa-ban"></i>已拒絕
    </button>
  </div>

  ${state.loading
    ? `<div class="loader"><i class="fa-solid fa-spinner fa-spin"></i> 載入中…</div>`
    : adminPosts.length === 0
      ? `<div class="empty-state"><i class="fa-solid fa-inbox"></i><p>此分類目前沒有內容</p></div>`
      : adminPosts.map(p => {
          const name = p.profiles?.name || '未知';
          const flags = p.sensitive_flags || [];
          return `
<div class="review-card">
  <div class="review-main">
    <div class="flex-center gap-8" style="margin-bottom:5px">
      <div class="review-title">${esc(p.title)}</div>
      <span class="badge badge-${p.status}" style="margin-left:auto">
        ${p.status==='approved'?'已上架':p.status==='pending'?'待審核':'已拒絕'}
      </span>
    </div>
    <div class="review-meta">提交者：${esc(name)} · ${fmtDate(p.created_at)} · ${p.content_type==='image'?'圖片':'文字'}</div>
    ${flags.length > 0 ? `<div class="sensitive-alert"><i class="fa-solid fa-triangle-exclamation"></i>偵測到敏感詞：${flags.map(esc).join('、')}</div>` : ''}
    <div class="review-body"><span style="font-size:11px;color:var(--text-muted)">提示詞：</span>${esc(p.prompt)}</div>
    <div class="post-tags">${(p.tags||[]).map(t=>`<span class="tag"># ${esc(t)}</span>`).join('')}</div>
  </div>
  ${p.status === 'pending' ? `
  <div class="review-actions">
    <button class="btn btn-teal" onclick="App.approvePost('${p.id}')"><i class="fa-solid fa-check"></i>通過</button>
    <button class="btn btn-danger" onclick="App.rejectPost('${p.id}')"><i class="fa-solid fa-ban"></i>拒絕</button>
  </div>` : ''}
</div>`;
        }).join('')
  }
</div>`;
}

/* ---- Upload Modal ---- */
function renderUploadModal() {
  return `
<div class="modal-overlay" onclick="if(event.target===this)App.closeModal()">
  <div class="modal">
    <div class="modal-header">
      <h3><i class="fa-solid fa-wand-magic-sparkles" style="color:var(--ocean-bright);margin-right:7px"></i>分享 AI 提示詞</h3>
      <button class="close-btn" onclick="App.closeModal()"><i class="fa-solid fa-xmark"></i></button>
    </div>
    <div class="modal-body">
      <div class="form-group">
        <label class="form-label">標題 <span class="req">*</span></label>
        <input id="up-title" type="text" class="form-input" placeholder="簡短描述這個提示詞的用途…">
      </div>
      <div class="form-row">
        <div class="form-group">
          <label class="form-label">內容類型</label>
          <select id="up-type" class="form-input">
            <option value="text">文字生成</option>
            <option value="image">圖像生成</option>
            <option value="file">檔案/其他</option>
          </select>
        </div>
        <div class="form-group">
          <label class="form-label">AI 工具</label>
          <select id="up-tool" class="form-input">
            <option>ChatGPT</option>
            <option>Claude</option>
            <option>Midjourney</option>
            <option>DALL-E</option>
            <option>Stable Diffusion</option>
            <option>Gemini</option>
            <option>其他</option>
          </select>
        </div>
      </div>

      <div class="form-group">
        <label class="form-label">提示詞內容 <span class="req">*</span></label>
        <div class="rich-editor">
          <div class="rich-toolbar">
            <button class="tb-btn" onclick="rc('bold')" title="粗體"><i class="fa-solid fa-bold"></i></button>
            <button class="tb-btn" onclick="rc('italic')" title="斜體"><i class="fa-solid fa-italic"></i></button>
            <button class="tb-btn" onclick="rc('underline')" title="底線"><i class="fa-solid fa-underline"></i></button>
            <div class="tb-sep"></div>
            <button class="tb-btn" onclick="rc('insertUnorderedList')" title="無序清單"><i class="fa-solid fa-list-ul"></i></button>
            <button class="tb-btn" onclick="rc('insertOrderedList')" title="有序清單"><i class="fa-solid fa-list-ol"></i></button>
            <div class="tb-sep"></div>
            <button class="tb-btn" onclick="rc('justifyLeft')"><i class="fa-solid fa-align-left"></i></button>
            <button class="tb-btn" onclick="rc('justifyCenter')"><i class="fa-solid fa-align-center"></i></button>
            <div class="tb-sep"></div>
            <button class="tb-btn" onclick="rc('undo')"><i class="fa-solid fa-rotate-left"></i></button>
            <button class="tb-btn" onclick="rc('redo')"><i class="fa-solid fa-rotate-right"></i></button>
          </div>
          <div id="up-prompt" class="rich-area" contenteditable="true"
            data-ph="輸入你的提示詞…&#10;例：水彩風格的深海章魚，bioluminescent，藍紫色調…"></div>
        </div>
      </div>

      <div class="form-group">
        <label class="form-label">AI 生成結果（文字）</label>
        <div class="rich-editor">
          <div class="rich-toolbar">
            <button class="tb-btn" onclick="rr('bold')"><i class="fa-solid fa-bold"></i></button>
            <button class="tb-btn" onclick="rr('italic')"><i class="fa-solid fa-italic"></i></button>
            <div class="tb-sep"></div>
            <button class="tb-btn" onclick="rr('undo')"><i class="fa-solid fa-rotate-left"></i></button>
          </div>
          <div id="up-result" class="rich-area" contenteditable="true"
            data-ph="貼上 AI 生成的文字結果…（圖片請在下方上傳）"></div>
        </div>
      </div>

      <div class="form-group">
        <label class="form-label">上傳附件
          <span class="text-muted text-sm">（圖片自動轉 WebP · 上限 10MB）</span>
        </label>
        <div class="file-drop" id="file-drop"
          ondragover="event.preventDefault();this.classList.add('over')"
          ondragleave="this.classList.remove('over')"
          ondrop="App.handleDrop(event)"
          onclick="document.getElementById('file-input').click()">
          <div class="icon"><i class="fa-solid fa-cloud-arrow-up"></i></div>
          <p>拖曳或點擊上傳</p>
          <span>JPG PNG GIF → 自動轉 WebP &ensp;·&ensp; PDF ZIP DOC 自動壓縮</span>
        </div>
        <input type="file" id="file-input" style="display:none" multiple
          accept="image/*,.pdf,.zip,.doc,.docx"
          onchange="App.handleFiles(this.files)">
        <div id="file-list" class="file-list"></div>
      </div>

      <div class="form-group">
        <label class="form-label">標籤（逗號分隔）</label>
        <input id="up-tags" type="text" class="form-input" placeholder="例：圖像生成, Midjourney, 海洋生物">
      </div>

      <div class="info-box">
        <i class="fa-solid fa-circle-info"></i>
        送出後由管理員審核；含敏感詞彙的內容將自動拒絕。請確保內容符合博物館使用規範。
      </div>
    </div>
    <div class="modal-footer">
      <button class="btn btn-ghost" onclick="App.closeModal()">取消</button>
      <button class="btn btn-primary" onclick="App.doSubmit()">
        <i class="fa-solid fa-paper-plane"></i>送出審核
      </button>
    </div>
  </div>
</div>`;
}

/* ---- Detail Modal ---- */
function renderDetailModal(state) {
  const post = state.detailPost;
  if (!post) return `<div class="modal-overlay" onclick="App.closeModal()"><div class="modal"><div class="loader"><i class="fa-solid fa-spinner fa-spin"></i></div></div></div>`;
  const name = post.profiles?.name || '未知';
  const liked = state.likedSet?.has(post.id);
  const imgs = (post.attachments || []).filter(a => a.is_webp);
  const files = (post.attachments || []).filter(a => !a.is_webp);

  return `
<div class="modal-overlay" onclick="if(event.target===this)App.closeModal()">
  <div class="modal modal-lg">
    <div class="modal-header">
      <div>
        <div style="margin-bottom:5px">${(post.tags||[]).map(t=>`<span class="tag" style="margin-right:4px"># ${esc(t)}</span>`).join('')}</div>
        <h3>${esc(post.title)}</h3>
      </div>
      <button class="close-btn" onclick="App.closeModal()"><i class="fa-solid fa-xmark"></i></button>
    </div>
    <div class="modal-body">
      <div class="flex-center gap-12" style="margin-bottom:1.1rem">
        <div class="avatar" style="width:36px;height:36px;font-size:13px;background:${avatarColor(name)}">
          ${initials(name)}
        </div>
        <div>
          <div style="font-size:14px;font-weight:500">${esc(name)}</div>
          <div class="text-muted text-sm">${fmtDate(post.created_at)}</div>
        </div>
        <span class="badge badge-approved" style="margin-left:auto">已上架</span>
        <span class="badge ${post.content_type==='image'?'badge-image':'badge-text'}">
          ${post.content_type==='image'?'圖片':'文字'}
        </span>
      </div>

      <div class="section-label">提示詞</div>
      <div class="prompt-block" style="margin-bottom:1.2rem">
        ${esc(post.prompt)}
        <button class="copy-btn" onclick="App.copyText('${esc(post.prompt).replace(/'/g,'&#39;')}')">
          <i class="fa-regular fa-copy"></i>複製
        </button>
      </div>

      ${post.result_text ? `
      <div class="section-label">生成結果（文字）</div>
      <div style="background:rgba(10,22,40,.4);border:1px solid var(--border-subtle);border-radius:var(--r-sm);padding:13px 15px;font-size:14px;line-height:1.8;color:var(--text-secondary);margin-bottom:1.2rem">
        ${esc(post.result_text)}
      </div>` : ''}

      ${imgs.length > 0 ? `
      <div class="section-label">生成圖片</div>
      <div style="display:grid;grid-template-columns:repeat(auto-fill,minmax(200px,1fr));gap:10px;margin-bottom:1.2rem">
        ${imgs.map(a=>`
        <div style="border-radius:var(--r-md);overflow:hidden;border:1px solid var(--border-subtle)">
          <img src="${esc(a.file_url)}" alt="${esc(a.file_name)}" style="width:100%;display:block">
        </div>`).join('')}
      </div>` : imgs.length === 0 && post.content_type === 'image' ? `
      <div class="detail-img-wrap" style="margin-bottom:1.2rem">
        <div class="detail-img-placeholder">
          <i class="fa-regular fa-image"></i>
          <p>圖片載入中或尚未上傳</p>
        </div>
      </div>` : ''}

      ${files.length > 0 ? `
      <div class="section-label">附件檔案</div>
      <div class="file-list" style="margin-bottom:1.2rem">
        ${files.map(a=>`
        <div class="file-item">
          <i class="fa-regular fa-file fi-icon"></i>
          <span class="fi-name">${esc(a.file_name)}</span>
          <span class="fi-size">${a.file_size ? Math.round(a.file_size/1024)+'KB' : ''}</span>
          <a href="${esc(a.file_url)}" target="_blank" class="btn btn-ghost" style="padding:4px 10px;font-size:12px">
            <i class="fa-solid fa-download"></i>下載
          </a>
        </div>`).join('')}
      </div>` : ''}

      <div class="action-bar">
        <button class="post-action ${liked?'liked':''}" onclick="App.toggleLike('${post.id}')">
          <i class="${liked?'fa-solid':'fa-regular'} fa-heart"></i>&ensp;${fmtNum(post.like_count)} 個讚
        </button>
        <button class="post-action"><i class="fa-regular fa-eye"></i>&ensp;${fmtNum(post.view_count)}</button>
        <button class="post-action" onclick="App.copyText(window.location.href)">
          <i class="fa-solid fa-share-nodes"></i>&ensp;分享
        </button>
        <button class="post-action"><i class="fa-regular fa-bookmark"></i>&ensp;收藏</button>
      </div>

      <div class="section-label">討論（${(state.comments||[]).length}）</div>
      <div id="comment-list">
        ${(state.comments||[]).length === 0
          ? `<div class="text-muted text-sm" style="margin-bottom:10px">尚無留言，搶先留言吧！</div>`
          : (state.comments||[]).map(c=>`
        <div class="comment-item">
          <div class="comment-author">${esc(c.profiles?.name||'未知')}</div>
          <div class="comment-text">${esc(c.content)}</div>
          <div class="comment-time">${fmtDate(c.created_at)}</div>
        </div>`).join('')}
      </div>
      <div class="comment-input-row">
        <input id="cmt-input" type="text" class="form-input" placeholder="留下你的想法…"
          onkeydown="if(event.key==='Enter')App.submitComment('${post.id}')">
        <button class="btn btn-primary" onclick="App.submitComment('${post.id}')">
          <i class="fa-solid fa-paper-plane"></i>
        </button>
      </div>
    </div>
  </div>
</div>`;
}

/* ---- Profile Modal ---- */
function renderProfileModal(state) {
  const { profile, myPosts = [] } = state;
  const totalLikes = myPosts.reduce((a, p) => a + (p.like_count || 0), 0);
  const totalViews = myPosts.reduce((a, p) => a + (p.view_count || 0), 0);
  return `
<div class="modal-overlay" onclick="if(event.target===this)App.closeModal()">
  <div class="modal modal-sm">
    <div class="modal-header">
      <h3>個人資料</h3>
      <button class="close-btn" onclick="App.closeModal()"><i class="fa-solid fa-xmark"></i></button>
    </div>
    <div class="modal-body" style="text-align:center">
      <div class="avatar" style="width:72px;height:72px;font-size:28px;margin:0 auto 1rem;background:${avatarColor(profile?.name)};border-width:3px">
        ${initials(profile?.name)}
      </div>
      <div style="font-size:20px;font-weight:700;margin-bottom:4px">${esc(profile?.name)}</div>
      ${profile?.role === 'admin' ? `<span class="badge badge-admin" style="margin-bottom:12px"><i class="fa-solid fa-shield-halved"></i> 管理員</span>` : ''}
      <div class="profile-stats">
        <div class="profile-stat"><div class="pval val-blue">${myPosts.length}</div><div class="plbl">分享篇數</div></div>
        <div class="profile-stat"><div class="pval val-coral">${fmtNum(totalLikes)}</div><div class="plbl">獲讚數</div></div>
        <div class="profile-stat"><div class="pval val-teal">${fmtNum(totalViews)}</div><div class="plbl">總瀏覽</div></div>
      </div>
    </div>
    <div class="modal-footer">
      <button class="btn btn-ghost" style="flex:1" onclick="App.closeModal()">關閉</button>
      <button class="btn btn-danger" style="flex:1" onclick="App.doSignOut()">
        <i class="fa-solid fa-right-from-bracket"></i>登出
      </button>
    </div>
  </div>
</div>`;
}

/* ---- Toast ---- */
function renderToast(toast) {
  if (!toast) return '';
  return `<div class="toast ${toast.type}">
    <i class="fa-solid ${toast.type==='success'?'fa-circle-check':toast.type==='error'?'fa-circle-xmark':'fa-circle-info'}"></i>
    ${esc(toast.msg)}
  </div>`;
}

/* ---- Rich editor helpers (global) ---- */
function rc(cmd) { document.getElementById('up-prompt')?.focus(); document.execCommand(cmd, false, null); }
function rr(cmd) { document.getElementById('up-result')?.focus(); document.execCommand(cmd, false, null); }
// =============================================
// js/app.js  — 主控制器，管理狀態與事件
// =============================================

const App = (() => {

  /* ---------- State ---------- */
  let S = {
    user:        null,
    profile:     null,
    page:        'login',
    loginTab:    'login',
    loading:     false,
    posts:       [],
    myPosts:     [],
    adminPosts:  [],
    topPosts:    [],
    stats:       {},
    adminStats:  {},
    adminTab:    'pending',
    comments:    [],
    likedSet:    new Set(),
    sortBy:      'view_count',
    filterTag:   null,
    filterType:  null,
    searchQ:     '',
    modal:       null,   // 'upload' | 'detail' | 'profile'
    detailPost:  null,
    toast:       null,
    pendingCount: 0,
    uploadFiles:  [],
    logoSrc:     'img/logo.png',  // 博物館 LOGO 路徑
  };

  let toastTimer = null;

  /* ---------- Render ---------- */
  function render() {
    const root = document.getElementById('app');
    let html = '';

    if (!S.user) {
      html = renderLogin(S.loginTab);
    } else {
      html  = renderHeader(S);
      if (S.page === 'feed')  html += renderFeed(S);
      if (S.page === 'my')    html += renderMyPage(S);
      if (S.page === 'admin') html += renderAdmin(S);
    }

    if (S.modal === 'upload')  html += renderUploadModal();
    if (S.modal === 'detail')  html += renderDetailModal(S);
    if (S.modal === 'profile') html += renderProfileModal(S);

    html += renderToast(S.toast);
    root.innerHTML = html;
  }

  function set(updates) { Object.assign(S, updates); render(); }

  function toast(msg, type = 'info') {
    S.toast = { msg, type };
    render();
    clearTimeout(toastTimer);
    toastTimer = setTimeout(() => { S.toast = null; render(); }, 3500);
  }

  /* ---------- Auth ---------- */
  async function init() {
    const session = await getSession();
    if (session?.user) {
      const profile = await getProfile(session.user.id);
      const likedSet = await fetchUserLikes(session.user.id);
      S.user = session.user; S.profile = profile; S.likedSet = likedSet;
      S.page = 'feed';
      render();
      loadFeed();
      loadStats();
      loadTopPosts();
    } else {
      render();
    }

    // Listen for auth changes
    db.auth.onAuthStateChange(async (event, session) => {
      if (event === 'SIGNED_IN' && session) {
        const profile = await getProfile(session.user.id);
        const likedSet = await fetchUserLikes(session.user.id);
        set({ user: session.user, profile, likedSet, page: 'feed' });
        loadFeed(); loadStats(); loadTopPosts();
      }
      if (event === 'SIGNED_OUT') {
        set({ user: null, profile: null, page: 'login', likedSet: new Set(), posts: [], myPosts: [] });
      }
    });
  }

  async function doLogin() {
    const email = document.getElementById('li-email')?.value?.trim();
    const pw    = document.getElementById('li-pw')?.value;
    if (!email || !pw) { toast('請填寫帳號密碼', 'error'); return; }
    const { error } = await signIn(email, pw);
    if (error) toast(error.message || '登入失敗', 'error');
    else toast(`歡迎回來！`, 'success');
  }

  async function doRegister() {
    const name  = document.getElementById('reg-name')?.value?.trim();
    const email = document.getElementById('reg-email')?.value?.trim();
    const pw    = document.getElementById('reg-pw')?.value;
    if (!name || !email || !pw) { toast('請填寫所有欄位', 'error'); return; }
    if (pw.length < 6) { toast('密碼至少 6 位', 'error'); return; }
    const { error } = await signUp(email, pw, name);
    if (error) toast(error.message || '註冊失敗', 'error');
    else toast('註冊成功！請確認您的電子郵件 ✓', 'success');
  }

  async function doSignOut() {
    await signOut();
    toast('已登出', 'info');
  }

  /* ---------- Navigation ---------- */
  function goto(page) {
    set({ page, modal: null });
    if (page === 'feed')  loadFeed();
    if (page === 'my')    loadMyPosts();
    if (page === 'admin') loadAdminPosts();
  }

  function setLoginTab(tab) { set({ loginTab: tab }); }

  /* ---------- Feed / Filters ---------- */
  async function loadFeed() {
    set({ loading: true });
    const { data } = await fetchPosts({ sort: S.sortBy, tag: S.filterTag, search: S.searchQ });
    set({ posts: data, loading: false });
  }

  async function loadMyPosts() {
    if (!S.user) return;
    set({ loading: true });
    const { data } = await fetchMyPosts(S.user.id);
    set({ myPosts: data, loading: false });
  }

  async function loadAdminPosts() {
    set({ loading: true });
    const { data } = await (
      S.adminTab === 'pending'  ? fetchPendingPosts() :
      S.adminTab === 'approved' ? fetchPosts({ status: 'approved', sort: 'created_at' }) :
      fetchPosts({ status: 'rejected', sort: 'created_at' })
    );
    const adminStats = await fetchStats();
    set({ adminPosts: data, adminStats, pendingCount: adminStats.pending, loading: false });
  }

  async function loadStats() {
    const stats = await fetchStats();
    set({ stats, pendingCount: stats.pending });
  }

  async function loadTopPosts() {
    const topPosts = await fetchTopPosts(RANK_LIMIT);
    set({ topPosts });
  }

  function setSort(sortBy) { set({ sortBy }); loadFeed(); }

  function setFilter(type, val) {
    if (type === 'tag')  set({ filterTag: S.filterTag === val ? null : val, filterType: null });
    if (type === 'type') set({ filterType: S.filterType === val ? null : val, filterTag: null });
    loadFeed();
  }

  let searchTimer;
  function setSearch(q) {
    S.searchQ = q;
    clearTimeout(searchTimer);
    searchTimer = setTimeout(loadFeed, 350);
    render();
  }

  function setAdminTab(tab) { set({ adminTab: tab }); loadAdminPosts(); }

  /* ---------- Post Detail ---------- */
  async function openDetail(id) {
    set({ modal: 'detail', detailPost: null, comments: [] });
    incrementViewCount(id);
    const { data } = await fetchPostById(id);
    const comments = await fetchComments(id);
    set({ detailPost: data, comments });
  }

  async function submitComment(postId) {
    const input = document.getElementById('cmt-input');
    const content = input?.value?.trim();
    if (!content) return;
    const { data, error } = await addComment(postId, S.user.id, content);
    if (error) { toast('留言失敗', 'error'); return; }
    const comments = [...S.comments, data];
    // update comment count in local state
    const detailPost = S.detailPost ? { ...S.detailPost, comment_count: (S.detailPost.comment_count || 0) + 1 } : S.detailPost;
    set({ comments, detailPost });
    toast('已留言', 'success');
  }

  /* ---------- Likes ---------- */
  async function toggleLikeAction(postId) {
    if (!S.user) { toast('請先登入', 'error'); return; }
    const liked = await toggleLike(postId, S.user.id);
    const likedSet = new Set(S.likedSet);
    if (liked) likedSet.add(postId); else likedSet.delete(postId);

    const delta = liked ? 1 : -1;
    const updateList = list => list.map(p =>
      p.id === postId ? { ...p, like_count: Math.max((p.like_count || 0) + delta, 0) } : p
    );
    const detailPost = S.detailPost?.id === postId
      ? { ...S.detailPost, like_count: Math.max((S.detailPost.like_count || 0) + delta, 0) }
      : S.detailPost;

    set({ likedSet, posts: updateList(S.posts), myPosts: updateList(S.myPosts), detailPost });
  }

  /* ---------- Upload ---------- */
  function openUpload() {
    S.uploadFiles = [];
    set({ modal: 'upload' });
  }

  function handleFiles(files) {
    for (const f of files) {
      if (f.size > 10 * 1024 * 1024) { toast(`${f.name} 超過 10MB`, 'error'); continue; }
      S.uploadFiles.push(f);
    }
    refreshFileList();
  }

  function handleDrop(e) {
    e.preventDefault();
    document.getElementById('file-drop')?.classList.remove('over');
    handleFiles(e.dataTransfer.files);
  }

  function removeFile(idx) {
    S.uploadFiles.splice(idx, 1);
    refreshFileList();
  }

  function refreshFileList() {
    const el = document.getElementById('file-list');
    if (!el) return;
    if (!S.uploadFiles.length) { el.innerHTML = ''; return; }
    el.innerHTML = S.uploadFiles.map((f, i) => {
      const isImg = f.type.startsWith('image/');
      const dispName = isImg ? f.name.replace(/\.(jpg|jpeg|png|gif|bmp)$/i, '.webp') : f.name;
      return `
      <div class="file-item">
        <i class="fa-regular ${isImg ? 'fa-image' : 'fa-file'} fi-icon"></i>
        <span class="fi-name">${esc(dispName)}</span>
        ${isImg ? '<span class="webp-tag">WebP</span>' : ''}
        <span class="fi-size">${Math.round(f.size / 1024)}KB</span>
        <button class="fi-remove" onclick="App.removeFile(${i})"><i class="fa-solid fa-xmark"></i></button>
      </div>`;
    }).join('');
  }

  async function doSubmit() {
    const title  = document.getElementById('up-title')?.value?.trim();
    const type   = document.getElementById('up-type')?.value || 'text';
    const tool   = document.getElementById('up-tool')?.value || '';
    const prompt = document.getElementById('up-prompt')?.innerText?.trim();
    const result = document.getElementById('up-result')?.innerText?.trim() || null;
    const tagsRaw = document.getElementById('up-tags')?.value || '';
    const tags = [...new Set([
      ...tagsRaw.split(',').map(t => t.trim()).filter(Boolean),
      ...(tool && tool !== '其他' ? [tool] : []),
    ])];

    if (!title)  { toast('請填寫標題', 'error'); return; }
    if (!prompt) { toast('請填寫提示詞內容', 'error'); return; }

    const sensitiveFlags = detectSensitive(title + ' ' + prompt + ' ' + result);
    const status = sensitiveFlags.length > 0 ? 'rejected' : 'pending';

    const payload = {
      user_id: S.user.id,
      title, content_type: type, prompt,
      result_text: result, tags, status,
      sensitive_flags: sensitiveFlags,
    };

    const { data: post, error } = await createPost(payload);
    if (error) { toast('送出失敗：' + error.message, 'error'); return; }

    // Upload attachments
    for (const file of S.uploadFiles) {
      await uploadFile(file, post.id);
    }

    set({ modal: null });
    S.uploadFiles = [];

    if (sensitiveFlags.length > 0) {
      toast(`含敏感詞「${sensitiveFlags[0]}」，已自動拒絕`, 'error');
    } else {
      toast('已送出，等待管理員審核 ✓', 'success');
    }
    loadFeed();
    loadStats();
  }

  /* ---------- Admin ---------- */
  async function approvePost(id) {
    const { error } = await updatePostStatus(id, 'approved');
    if (error) { toast('操作失敗', 'error'); return; }
    toast('已通過審核 ✓', 'success');
    loadAdminPosts(); loadStats(); loadTopPosts();
  }

  async function rejectPost(id) {
    const { error } = await updatePostStatus(id, 'rejected');
    if (error) { toast('操作失敗', 'error'); return; }
    toast('已拒絕刊登', 'error');
    loadAdminPosts(); loadStats();
  }

  /* ---------- Misc ---------- */
  function openProfile() { loadMyPosts(); set({ modal: 'profile' }); }
  function closeModal()   { set({ modal: null }); }

  function copyText(text) {
    navigator.clipboard.writeText(text).then(() => toast('已複製到剪貼簿 ✓', 'success'));
  }

  /* ---------- Expose ---------- */
  return {
    init,
    doLogin, doRegister, doSignOut,
    goto, setLoginTab,
    setSort, setFilter, setSearch, setAdminTab,
    openDetail, submitComment,
    toggleLike: toggleLikeAction,
    openUpload, handleFiles, handleDrop, removeFile, doSubmit,
    approvePost, rejectPost,
    openProfile, closeModal, copyText,
  };
})();

/* ---- Bootstrap ---- */
window.addEventListener('DOMContentLoaded', App.init);
</script>
</body>
</html>
