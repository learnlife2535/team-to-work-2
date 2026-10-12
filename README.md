<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" />
  <title>TEAM OS V4.5</title>
  <script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
  <script src="https://cdn.tailwindcss.com"></script>
  <link href="https://fonts.googleapis.com/css2?family=Kanit:wght@400;500;600;700;900&display=swap" rel="stylesheet" />
  <script>
    tailwind.config = {
      theme: {
        extend: {
          colors: {
            cream: '#FFFBF0',
            ink: '#18181B',
            yellow: '#FFD84D',
            line: '#E8E2D6',
            muted: '#71717A',
          },
          fontFamily: { sans: ['Kanit', 'ui-sans-serif', 'system-ui', 'sans-serif'] },
        },
      },
    };
  </script>
  <style>
    * { font-family: Kanit, ui-sans-serif, system-ui, sans-serif; }
    body { background: #fffbf0; color: #18181b; margin: 0; }
  </style>
</head>
<body>
  <div id="root"></div>

  <script type="module">
    // ═══════════════════════════════════════════════════════════
    // ใส่ค่าจาก Supabase Dashboard → Project Settings → API
    // ═══════════════════════════════════════════════════════════
    const SUPABASE_URL = localStorage.getItem('team_os_supabase_url') || '';
    const SUPABASE_ANON_KEY = localStorage.getItem('team_os_supabase_key') || '';

    let supabase = null;
    if (SUPABASE_URL && SUPABASE_ANON_KEY) {
      supabase = window.supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);
    }

    const root = document.getElementById('root');

    const state = {
      view: 'boot', // boot | config | auth | createOrg | app
      mode: 'login', // login | signup
      user: null,
      org: null,
      role: null,
      email: '',
      password: '',
      displayName: '',
      orgName: '',
      orgSlug: '',
      jobs: [],
      drafts: [],
      msg: '',
      err: '',
      loading: false,
      cfgUrl: SUPABASE_URL,
      cfgKey: SUPABASE_ANON_KEY,
    };

    function slugify(t) {
      return String(t || '')
        .trim()
        .toLowerCase()
        .replace(/\s+/g, '-')
        .replace(/[^\w\-]+/g, '')
        .replace(/\-\-+/g, '-')
        .replace(/^-+|-+$/g, '');
    }

    function setState(patch) {
      Object.assign(state, patch);
      render();
    }

    async function boot() {
      if (!supabase) {
        setState({ view: 'config' });
        return;
      }
      const { data: { session } } = await supabase.auth.getSession();
      if (!session) {
        setState({ view: 'auth' });
        return;
      }
      await afterAuth(session.user);
    }

    async function afterAuth(authUser) {
      setState({ loading: true, err: '' });
      // ensure profile exists
      await supabase.from('profiles').upsert({
        id: authUser.id,
        email: authUser.email,
        display_name: authUser.user_metadata?.display_name || authUser.email?.split('@')[0] || 'User',
      }, { onConflict: 'id' });

      const { data: member } = await supabase
        .from('organization_members')
        .select('role, organization_id, organizations(id, name, slug)')
        .eq('user_id', authUser.id)
        .eq('status', 'active')
        .limit(1)
        .maybeSingle();

      if (!member) {
        setState({
          view: 'createOrg',
          user: { id: authUser.id, email: authUser.email, name: authUser.email },
          loading: false,
        });
        return;
      }

      const org = member.organizations;
      setState({
        user: { id: authUser.id, email: authUser.email, name: authUser.email },
        org,
        role: member.role,
        view: 'app',
        loading: false,
      });
      await loadData(org.id);
    }

    async function loadData(orgId) {
      const [jobsRes, draftsRes] = await Promise.all([
        supabase.from('jobs').select('*').eq('organization_id', orgId).order('created_at', { ascending: false }),
        supabase.from('content_drafts').select('*').eq('organization_id', orgId).order('created_at', { ascending: false }),
      ]);
      setState({
        jobs: jobsRes.data || [],
        drafts: draftsRes.data || [],
      });
    }

    async function doSignup() {
      setState({ loading: true, err: '' });
      const { data, error } = await supabase.auth.signUp({
        email: state.email.trim(),
        password: state.password,
        options: { data: { display_name: state.displayName.trim() || state.email.split('@')[0] } },
      });
      if (error) {
        setState({ err: error.message, loading: false });
        return;
      }
      if (data.session) {
        await afterAuth(data.user);
      } else {
        setState({ msg: 'สมัครสำเร็จ — ตรวจอีเมลยืนยัน (ถ้าเปิดไว้) หรือลองเข้าสู่ระบบ', loading: false, mode: 'login' });
      }
    }

    async function doLogin() {
      setState({ loading: true, err: '' });
      const { data, error } = await supabase.auth.signInWithPassword({
        email: state.email.trim(),
        password: state.password,
      });
      if (error) {
        setState({ err: error.message, loading: false });
        return;
      }
      await afterAuth(data.user);
    }

    async function createOrg() {
      if (!state.orgName.trim()) {
        setState({ err: 'กรุณาใส่ชื่อทีม' });
        return;
      }
      const slug = state.orgSlug.trim() || slugify(state.orgName);
      setState({ loading: true, err: '' });
      try {
        const { data: org, error: orgErr } = await supabase
          .from('organizations')
          .insert({ name: state.orgName.trim(), slug, timezone: 'Asia/Bangkok' })
          .select()
          .single();
        if (orgErr) throw orgErr;

        const { error: memErr } = await supabase.from('organization_members').insert({
          organization_id: org.id,
          user_id: state.user.id,
          role: 'owner',
          status: 'active',
          joined_at: new Date().toISOString(),
        });
        if (memErr) throw memErr;

        setState({ org, role: 'owner', view: 'app', loading: false, jobs: [], drafts: [] });
      } catch (e) {
        setState({
          err: e.code === '23505' ? 'Slug นี้ถูกใช้แล้ว ลองเปลี่ยนชื่อ' : (e.message || 'สร้างไม่สำเร็จ'),
          loading: false,
        });
      }
    }

    async function addJob() {
      const title = prompt('ชื่องาน');
      if (!title) return;
      const { data, error } = await supabase.from('jobs').insert({
        organization_id: state.org.id,
        title,
        client_name: 'ลูกค้าใหม่',
        amount: 0,
        status: 'pending',
        created_by: state.user.id,
      }).select().single();
      if (error) {
        alert(error.message);
        return;
      }
      setState({ jobs: [data, ...state.jobs] });
    }

    async function approveJob(id) {
      const { error } = await supabase.from('jobs').update({ status: 'approved' }).eq('id', id);
      if (error) return alert(error.message);
      setState({ jobs: state.jobs.map((j) => (j.id === id ? { ...j, status: 'approved' } : j)) });
    }

    async function logout() {
      await supabase.auth.signOut();
      setState({ user: null, org: null, role: null, view: 'auth', jobs: [], drafts: [] });
    }

    function saveConfig() {
      localStorage.setItem('team_os_supabase_url', state.cfgUrl.trim());
      localStorage.setItem('team_os_supabase_key', state.cfgKey.trim());
      location.reload();
    }

    function card(inner) {
      return `<div class="bg-white rounded-[20px] border border-line p-6 shadow-sm">${inner}</div>`;
    }

    function render() {
      if (state.view === 'config') {
        root.innerHTML = `
          <div class="min-h-screen flex items-center justify-center p-4">
            <div class="w-full max-w-md space-y-4">
              <div class="text-center">
                <div class="inline-flex w-14 h-14 rounded-2xl bg-ink text-yellow font-black text-xl items-center justify-center mb-3">T</div>
                <h1 class="font-bold text-xl">ตั้งค่า Supabase</h1>
                <p class="text-sm text-muted mt-1">ใส่ Project URL และ anon key จาก Dashboard</p>
              </div>
              ${card(`
                <label class="text-xs font-medium">Project URL</label>
                <input id="cfgUrl" value="${state.cfgUrl}" placeholder="https://xxxx.supabase.co"
                  class="mt-1 w-full h-11 px-3 rounded-xl border border-line bg-cream text-sm outline-none" />
                <label class="text-xs font-medium mt-3 block">anon / publishable key</label>
                <input id="cfgKey" value="${state.cfgKey}" placeholder="eyJhbGciOi..."
                  class="mt-1 w-full h-11 px-3 rounded-xl border border-line bg-cream text-sm outline-none" />
                <button id="btnSaveCfg" class="mt-4 w-full h-12 rounded-xl bg-ink text-yellow font-bold">บันทึกและเริ่มใช้งาน</button>
              `)}
            </div>
          </div>`;
        document.getElementById('cfgUrl').oninput = (e) => (state.cfgUrl = e.target.value);
        document.getElementById('cfgKey').oninput = (e) => (state.cfgKey = e.target.value);
        document.getElementById('btnSaveCfg').onclick = saveConfig;
        return;
      }

      if (state.view === 'auth') {
        root.innerHTML = `
          <div class="min-h-screen flex items-center justify-center p-4">
            <div class="w-full max-w-md space-y-4">
              <div class="text-center">
                <div class="inline-flex w-14 h-14 rounded-2xl bg-ink text-yellow font-black text-xl items-center justify-center mb-3">T</div>
                <h1 class="font-bold text-xl">TEAM OS V4.5</h1>
                <p class="text-sm text-muted mt-1">${state.mode === 'login' ? 'เข้าสู่ระบบ' : 'สมัครสมาชิก'}</p>
              </div>
              ${card(`
                ${state.mode === 'signup' ? `
                  <label class="text-xs font-medium">ชื่อที่แสดง</label>
                  <input id="displayName" value="${state.displayName}" class="mt-1 w-full h-11 px-3 rounded-xl border border-line bg-cream text-sm outline-none mb-3" />
                ` : ''}
                <label class="text-xs font-medium">อีเมล</label>
                <input id="email" type="email" value="${state.email}" class="mt-1 w-full h-11 px-3 rounded-xl border border-line bg-cream text-sm outline-none" />
                <label class="text-xs font-medium mt-3 block">รหัสผ่าน</label>
                <input id="password" type="password" value="${state.password}" class="mt-1 w-full h-11 px-3 rounded-xl border border-line bg-cream text-sm outline-none" />
                ${state.err ? `<div class="mt-3 text-xs text-red-600 bg-red-50 border border-red-200 rounded-xl px-3 py-2">${state.err}</div>` : ''}
                ${state.msg ? `<div class="mt-3 text-xs text-green-700 bg-green-50 border border-green-200 rounded-xl px-3 py-2">${state.msg}</div>` : ''}
                <button id="btnAuth" class="mt-4 w-full h-12 rounded-xl bg-ink text-yellow font-bold" ${state.loading ? 'disabled' : ''}>
                  ${state.loading ? 'กำลังทำงาน...' : (state.mode === 'login' ? 'เข้าสู่ระบบ' : 'สมัครสมาชิก')}
                </button>
                <button id="btnToggle" class="mt-2 w-full text-sm text-muted underline">
                  ${state.mode === 'login' ? 'ยังไม่มีบัญชี? สมัครสมาชิก' : 'มีบัญชีแล้ว? เข้าสู่ระบบ'}
                </button>
              `)}
              <button id="btnReconfig" class="w-full text-xs text-muted">ตั้งค่า Supabase ใหม่</button>
            </div>
          </div>`;
        ['email', 'password', 'displayName'].forEach((id) => {
          const el = document.getElementById(id);
          if (el) el.oninput = (e) => (state[id] = e.target.value);
        });
        document.getElementById('btnAuth').onclick = () => (state.mode === 'login' ? doLogin() : doSignup());
        document.getElementById('btnToggle').onclick = () => setState({ mode: state.mode === 'login' ? 'signup' : 'login', err: '', msg: '' });
        document.getElementById('btnReconfig').onclick = () => setState({ view: 'config' });
        return;
      }

      if (state.view === 'createOrg') {
        root.innerHTML = `
          <div class="min-h-screen flex items-center justify-center p-4">
            <div class="w-full max-w-md space-y-4">
              <div class="text-center">
                <h1 class="font-bold text-xl">สร้างทีม / องค์กร</h1>
                <p class="text-sm text-muted mt-1">คุณจะเป็น Owner ของทีมนี้</p>
              </div>
              ${card(`
                <label class="text-xs font-medium">ชื่อทีม</label>
                <input id="orgName" value="${state.orgName}" placeholder="ทีมบ้านรีโนเวท"
                  class="mt-1 w-full h-11 px-3 rounded-xl border border-line bg-cream text-sm outline-none" />
                <label class="text-xs font-medium mt-3 block">Slug</label>
                <div class="mt-1 flex items-center gap-2">
                  <span class="text-xs text-muted">team/</span>
                  <input id="orgSlug" value="${state.orgSlug}" placeholder="team-chiangmai"
                    class="flex-1 h-11 px-3 rounded-xl border border-line bg-cream text-sm outline-none" />
                </div>
                ${state.err ? `<div class="mt-3 text-xs text-red-600 bg-red-50 border border-red-200 rounded-xl px-3 py-2">${state.err}</div>` : ''}
                <button id="btnCreateOrg" class="mt-4 w-full h-12 rounded-xl bg-ink text-yellow font-bold" ${state.loading ? 'disabled' : ''}>
                  ${state.loading ? 'กำลังสร้าง...' : 'สร้างทีมและเข้าใช้งาน'}
                </button>
              `)}
              <button id="btnLogout" class="w-full text-sm text-muted">ออกจากระบบ</button>
            </div>
          </div>`;
        document.getElementById('orgName').oninput = (e) => {
          state.orgName = e.target.value;
          if (!state.orgSlug || state.orgSlug === slugify(state.orgName.slice(0, -1))) {
            state.orgSlug = slugify(e.target.value);
            document.getElementById('orgSlug').value = state.orgSlug;
          }
        };
        document.getElementById('orgSlug').oninput = (e) => (state.orgSlug = slugify(e.target.value));
        document.getElementById('btnCreateOrg').onclick = createOrg;
        document.getElementById('btnLogout').onclick = logout;
        return;
      }

      // App dashboard
      root.innerHTML = `
        <div class="min-h-screen">
          <header class="bg-white border-b border-line px-4 py-3 flex items-center justify-between">
            <div class="flex items-center gap-3">
              <div class="w-9 h-9 rounded-xl bg-ink text-yellow font-black flex items-center justify-center">T</div>
              <div>
                <div class="font-bold text-sm">${state.org?.name || 'TEAM OS'}</div>
                <div class="text-[11px] text-muted">${state.role} · ${state.user?.email || ''}</div>
              </div>
            </div>
            <button id="btnLogout" class="text-xs px-3 h-9 rounded-lg border border-line">ออกจากระบบ</button>
          </header>
          <main class="max-w-3xl mx-auto p-4 space-y-4">
            <div class="grid grid-cols-2 gap-3">
              <div class="bg-white rounded-2xl border border-line p-4 text-center">
                <div class="text-2xl font-bold">${state.jobs.length}</div>
                <div class="text-xs text-muted">งาน</div>
              </div>
              <div class="bg-white rounded-2xl border border-line p-4 text-center">
                <div class="text-2xl font-bold">${state.drafts.length}</div>
                <div class="text-xs text-muted">ดราฟต์</div>
              </div>
            </div>

            <div class="bg-white rounded-2xl border border-line p-4">
              <div class="flex items-center justify-between mb-3">
                <div class="font-bold text-sm">งานล่าสุด</div>
                <button id="btnAddJob" class="h-9 px-3 rounded-xl bg-ink text-yellow text-xs font-bold">+ เพิ่มงาน</button>
              </div>
              ${state.jobs.length === 0 ? '<div class="text-xs text-muted py-4 text-center">ยังไม่มีงาน</div>' : ''}
              <div class="space-y-2">
                ${state.jobs.map((j) => `
                  <div class="flex items-center justify-between border border-line rounded-xl px-3 py-2 bg-cream">
                    <div>
                      <div class="text-sm font-medium">${j.title}</div>
                      <div class="text-[11px] text-muted">${j.status} · ${j.client_name || ''}</div>
                    </div>
                    ${j.status === 'pending' && (state.role === 'owner' || state.role === 'admin') ? `
                      <button data-approve="${j.id}" class="text-[11px] px-2 py-1 rounded-lg bg-yellow font-bold">อนุมัติ</button>
                    ` : ''}
                  </div>
                `).join('')}
              </div>
            </div>

            <div class="bg-white rounded-2xl border border-line p-4">
              <div class="font-bold text-sm mb-3">ดราฟต์คอนเทนต์</div>
              ${state.drafts.length === 0 ? '<div class="text-xs text-muted py-4 text-center">ยังไม่มีดราฟต์</div>' : ''}
              <div class="space-y-2">
                ${state.drafts.map((d) => `
                  <div class="border border-line rounded-xl px-3 py-2 bg-cream">
                    <div class="text-sm font-medium">${d.title || 'ไม่มีชื่อ'}</div>
                    <div class="text-[11px] text-muted mt-0.5">${d.status}</div>
                    <div class="text-[11px] mt-1 line-clamp-2 text-muted">${(d.message_short || '').slice(0, 120)}</div>
                  </div>
                `).join('')}
              </div>
            </div>
          </main>
        </div>`;
      document.getElementById('btnLogout').onclick = logout;
      document.getElementById('btnAddJob').onclick = addJob;
      document.querySelectorAll('[data-approve]').forEach((btn) => {
        btn.onclick = () => approveJob(btn.getAttribute('data-approve'));
      });
    }

    boot();
  </script>
</body>
</html>
