<script setup lang="ts">
import { onMounted, ref } from 'vue'
type Staff = { email: string; role: string }
type Store = { id: string; name: string; is_open: boolean; email: string | null }
type Session = { access_token: string; refresh_token: string; expires_in: number }

const apiUrl = import.meta.env.VITE_API_BASE_URL?.replace(/\/$/, '')
const email = ref('admin@btadapp.com'), password = ref('')
const staff = ref<Staff | null>(null), stores = ref<Store[]>([])
const storeName = ref(''), username = ref(''), merchantPassword = ref('')
const busy = ref(false), message = ref(''), error = ref('')

function session(): Session | null {
  try { return JSON.parse(localStorage.getItem('canteen_staff_session') || 'null') } catch { return null }
}
function save(value: Session | null) {
  if (value) localStorage.setItem('canteen_staff_session', JSON.stringify(value))
  else localStorage.removeItem('canteen_staff_session')
}
async function api(path: string, options: RequestInit = {}, retry = true): Promise<any> {
  const current = session(), headers = new Headers(options.headers)
  headers.set('Content-Type', 'application/json')
  if (current?.access_token) headers.set('Authorization', `Bearer ${current.access_token}`)
  let response = await fetch(`${apiUrl}${path}`, { ...options, headers, cache: 'no-store' })
  if (response.status === 401 && retry && current?.refresh_token) {
    const refreshed = await fetch(`${apiUrl}/staff/refresh`, {
      method: 'POST', headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ refresh_token: current.refresh_token }),
    })
    if (refreshed.ok) { save(await refreshed.json()); return api(path, options, false) }
  }
  const data = await response.json().catch(() => ({}))
  if (!response.ok) throw new Error(data.detail || 'ดำเนินการไม่สำเร็จ')
  return data
}
async function load() {
  const [account, list] = await Promise.all([api('/staff/me'), api('/staff/stores')])
  staff.value = account.staff; stores.value = list.stores
}
async function login() {
  busy.value = true; error.value = ''
  try {
    save(await api('/staff/login', { method: 'POST', body: JSON.stringify({ email: email.value, password: password.value }) }, false))
    password.value = ''; await load()
  } catch (e) { error.value = e instanceof Error ? e.message : 'เข้าสู่ระบบไม่สำเร็จ' }
  finally { busy.value = false }
}
async function createStore() {
  busy.value = true; error.value = ''; message.value = ''
  try {
    await api('/staff/stores', { method: 'POST', body: JSON.stringify({
      store_name: storeName.value, username: username.value, password: merchantPassword.value,
    }) })
    storeName.value = ''; username.value = ''; merchantPassword.value = ''
    message.value = 'สร้างบัญชีร้านค้าเรียบร้อยแล้ว'; await load()
  } catch (e) { error.value = e instanceof Error ? e.message : 'สร้างร้านค้าไม่สำเร็จ' }
  finally { busy.value = false }
}
async function toggle(store: Store) {
  busy.value = true; error.value = ''
  try {
    await api(`/staff/stores/${encodeURIComponent(store.id)}`, {
      method: 'PATCH', body: JSON.stringify({ is_open: !store.is_open }),
    }); await load()
  } catch (e) { error.value = e instanceof Error ? e.message : 'เปลี่ยนสถานะร้านไม่สำเร็จ' }
  finally { busy.value = false }
}
function logout() { save(null); staff.value = null; stores.value = []; error.value = ''; message.value = '' }
onMounted(async () => {
  if (!apiUrl) { error.value = 'ระบบยังตั้งค่าไม่ครบ'; return }
  if (!session()) return
  busy.value = true; try { await load() } catch { logout() } finally { busy.value = false }
})
</script>

<template>
  <main class="admin-page">
    <header class="admin-header">
      <div><strong>CANTEEN</strong><span>ระบบผู้ดูแล</span></div>
      <button v-if="staff" class="secondary" @click="logout">ออกจากระบบ</button>
    </header>
    <section v-if="!staff" class="admin-login card">
      <span class="eyebrow">สำหรับผู้ดูแลระบบ</span><h1>เข้าสู่ระบบแอดมิน</h1>
      <form @submit.prevent="login">
        <label>อีเมล<input v-model.trim="email" type="email" autocomplete="username" required /></label>
        <label>รหัสผ่าน<input v-model="password" type="password" autocomplete="current-password" minlength="8" required /></label>
        <p v-if="error" class="error" role="alert">{{ error }}</p>
        <button class="primary" :disabled="busy">{{ busy ? 'กำลังเข้าสู่ระบบ…' : 'เข้าสู่ระบบ' }}</button>
      </form>
    </section>
    <template v-else>
      <section class="admin-title">
        <div><span class="eyebrow">เข้าสู่ระบบด้วย {{ staff.email }}</span><h1>จัดการร้านค้า</h1></div>
        <span class="count">{{ stores.length }} ร้าน</span>
      </section>
      <p v-if="message" class="success">{{ message }}</p>
      <p v-if="error" class="error" role="alert">{{ error }}</p>
      <div class="admin-grid">
        <section class="card create-card">
          <h2>เพิ่มบัญชีร้านค้า</h2><p>กำหนดชื่อผู้ใช้และรหัสผ่านเริ่มต้นให้เจ้าของร้าน</p>
          <form @submit.prevent="createStore">
            <label>ชื่อร้านค้า<input v-model.trim="storeName" maxlength="100" required placeholder="เช่น ร้านข้าวแกงป้าสม" /></label>
            <label>ชื่อผู้ใช้<input v-model.trim="username" minlength="3" maxlength="32" pattern="[a-z0-9][a-z0-9._-]{2,31}" required placeholder="เช่น somshop" /><small>อังกฤษตัวเล็ก ตัวเลข จุด ขีดกลาง หรือขีดล่าง</small></label>
            <label>รหัสผ่านเริ่มต้น<input v-model="merchantPassword" type="password" minlength="8" required autocomplete="new-password" /></label>
            <button class="primary" :disabled="busy">{{ busy ? 'กำลังบันทึก…' : 'สร้างร้านค้า' }}</button>
          </form>
        </section>
        <section class="card store-admin-list">
          <h2>ร้านค้าทั้งหมด</h2>
          <div v-if="stores.length === 0" class="empty">ยังไม่มีร้านค้า</div>
          <article v-for="store in stores" :key="store.id" class="admin-store-row">
            <div><strong>{{ store.name }}</strong><small>{{ store.email }}</small></div>
            <button :class="['status-button', { closed: !store.is_open }]" :disabled="busy" @click="toggle(store)">
              {{ store.is_open ? 'เปิดร้าน' : 'ปิดร้าน' }}
            </button>
          </article>
        </section>
      </div>
    </template>
  </main>
</template>

