<script setup lang="ts">
import { computed, onMounted, ref } from 'vue'
type Session = { access_token: string; refresh_token: string; expires_in: number }
type Store = { id: string; name: string; is_open: boolean }
type Product = { id: string; name: string; price: number; cost: number; stock: number; is_tracking: boolean }
type Item = { id: string; name: string; qty: number; price: number }
type Order = { id: string; order_code: string; customer_name: string; order_type: string; total_price: number; status: string; created_at: string; items: Item[] }

const apiUrl = import.meta.env.VITE_API_BASE_URL?.replace(/\/$/, '')
const username = ref(''), password = ref(''), store = ref<Store | null>(null)
const products = ref<Product[]>([]), orders = ref<Order[]>([]), tab = ref<'orders'|'menu'>('orders')
const name = ref(''), price = ref<number | null>(null), cost = ref<number>(0), stock = ref<number>(0), tracking = ref(false)
const editingId = ref<string | null>(null), busy = ref(false), error = ref(''), message = ref('')
const activeOrders = computed(() => orders.value.filter(o => o.status === 'pending' || o.status === 'cooking'))

function readSession(): Session | null { try { return JSON.parse(localStorage.getItem('canteen_merchant_session') || 'null') } catch { return null } }
function saveSession(value: Session | null) { value ? localStorage.setItem('canteen_merchant_session', JSON.stringify(value)) : localStorage.removeItem('canteen_merchant_session') }
async function api(path: string, options: RequestInit = {}, retry = true): Promise<any> {
  const current = readSession(), headers = new Headers(options.headers); headers.set('Content-Type', 'application/json')
  if (current?.access_token) headers.set('Authorization', `Bearer ${current.access_token}`)
  const response = await fetch(`${apiUrl}${path}`, { ...options, headers, cache: 'no-store' })
  if (response.status === 401 && retry && current?.refresh_token) {
    const refreshed = await fetch(`${apiUrl}/staff/refresh`, { method:'POST', headers:{'Content-Type':'application/json'}, body:JSON.stringify({refresh_token:current.refresh_token}) })
    if (refreshed.ok) { saveSession(await refreshed.json()); return api(path, options, false) }
  }
  const data = await response.json().catch(() => ({})); if (!response.ok) throw new Error(data.detail || 'ดำเนินการไม่สำเร็จ'); return data
}
async function load() { const data = await api('/merchant/dashboard'); store.value=data.store; products.value=data.products; orders.value=data.orders }
async function login() {
  busy.value=true; error.value=''
  try { saveSession(await api('/staff/login',{method:'POST',body:JSON.stringify({email:`${username.value.trim().toLowerCase()}@btadapp.com`,password:password.value})},false)); password.value=''; await load() }
  catch(e){ error.value=e instanceof Error?e.message:'เข้าสู่ระบบไม่สำเร็จ'; saveSession(null) } finally { busy.value=false }
}
function resetForm(){ editingId.value=null; name.value=''; price.value=null; cost.value=0; stock.value=0; tracking.value=false }
function edit(product: Product){ editingId.value=product.id; name.value=product.name; price.value=Number(product.price); cost.value=Number(product.cost); stock.value=product.stock; tracking.value=product.is_tracking; tab.value='menu' }
async function saveProduct(){ busy.value=true; error.value=''; message.value=''; try {
  const body=JSON.stringify({name:name.value,price:price.value,cost:cost.value,stock:stock.value,is_tracking:tracking.value})
  await api(editingId.value?`/merchant/products/${editingId.value}`:'/merchant/products',{method:editingId.value?'PUT':'POST',body})
  message.value=editingId.value?'แก้ไขเมนูแล้ว':'เพิ่มเมนูแล้ว'; resetForm(); await load()
 }catch(e){error.value=e instanceof Error?e.message:'บันทึกเมนูไม่สำเร็จ'}finally{busy.value=false}}
async function setStatus(order: Order, status: string){ busy.value=true; error.value=''; try{await api(`/merchant/orders/${order.id}`,{method:'PATCH',body:JSON.stringify({status})});await load()}catch(e){error.value=e instanceof Error?e.message:'เปลี่ยนสถานะไม่สำเร็จ'}finally{busy.value=false}}
function logout(){saveSession(null);store.value=null;products.value=[];orders.value=[];error.value='';message.value=''}
onMounted(async()=>{if(!readSession())return;busy.value=true;try{await load()}catch{logout()}finally{busy.value=false}})
</script>

<template>
<main class="merchant-page">
  <header class="merchant-header"><div><strong>CANTEEN</strong><span>ระบบร้านค้า</span></div><button v-if="store" class="secondary" @click="logout">ออกจากระบบ</button></header>
  <section v-if="!store" class="admin-login card"><span class="eyebrow">สำหรับเจ้าของร้าน</span><h1>เข้าสู่ระบบร้านค้า</h1><form @submit.prevent="login">
    <label>ชื่อผู้ใช้<input v-model.trim="username" autocomplete="username" required /></label><label>รหัสผ่าน<input v-model="password" type="password" minlength="8" autocomplete="current-password" required /></label>
    <p v-if="error" class="error">{{error}}</p><button class="primary" :disabled="busy">{{busy?'กำลังเข้าสู่ระบบ…':'เข้าสู่ระบบ'}}</button>
  </form></section>
  <template v-else>
    <section class="merchant-title"><div><span class="eyebrow">หน้าร้าน</span><h1>{{store.name}}</h1></div><span :class="['open-badge',{closed:!store.is_open}]">{{store.is_open?'เปิดรับออเดอร์':'ร้านปิด'}}</span></section>
    <nav class="merchant-tabs"><button :class="{active:tab==='orders'}" @click="tab='orders'">ออเดอร์ <span>{{activeOrders.length}}</span></button><button :class="{active:tab==='menu'}" @click="tab='menu'">เมนูอาหาร <span>{{products.length}}</span></button></nav>
    <p v-if="message" class="success">{{message}}</p><p v-if="error" class="error">{{error}}</p>
    <section v-if="tab==='orders'" class="order-grid">
      <div v-if="activeOrders.length===0" class="empty card">ยังไม่มีออเดอร์ใหม่</div>
      <article v-for="order in activeOrders" :key="order.id" class="order-card card"><header><div><strong>{{order.order_code}}</strong><small>{{order.customer_name||'ลูกค้า'}}</small></div><b>฿{{Number(order.total_price).toFixed(0)}}</b></header>
        <ul><li v-for="item in order.items" :key="item.id"><span>{{item.qty}} × {{item.name}}</span><span>฿{{Number(item.price*item.qty).toFixed(0)}}</span></li></ul>
        <div class="order-actions"><button v-if="order.status==='pending'" class="cook" :disabled="busy" @click="setStatus(order,'cooking')">รับออเดอร์</button><button v-else class="done" :disabled="busy" @click="setStatus(order,'completed')">ทำเสร็จแล้ว</button><button class="cancel" :disabled="busy" @click="setStatus(order,'cancelled')">ยกเลิก</button></div>
      </article>
    </section>
    <section v-else class="menu-grid"><form class="card product-form" @submit.prevent="saveProduct"><h2>{{editingId?'แก้ไขเมนู':'เพิ่มเมนูอาหาร'}}</h2>
      <label>ชื่อเมนู<input v-model.trim="name" maxlength="100" required /></label><div class="two-fields"><label>ราคาขาย<input v-model.number="price" type="number" min="0" step="0.01" required /></label><label>ต้นทุน<input v-model.number="cost" type="number" min="0" step="0.01" required /></label></div>
      <label class="check"><input v-model="tracking" type="checkbox" /> ติดตามจำนวนคงเหลือ</label><label v-if="tracking">จำนวนคงเหลือ<input v-model.number="stock" type="number" min="0" required /></label>
      <button class="primary" :disabled="busy">{{busy?'กำลังบันทึก…':editingId?'บันทึกการแก้ไข':'เพิ่มเมนู'}}</button><button v-if="editingId" type="button" class="secondary" @click="resetForm">ยกเลิกการแก้ไข</button>
    </form><div class="card product-admin-list"><h2>เมนูทั้งหมด</h2><div v-if="products.length===0" class="empty">ยังไม่มีเมนูอาหาร</div><button v-for="product in products" :key="product.id" class="product-admin-row" @click="edit(product)"><div><strong>{{product.name}}</strong><small>{{product.is_tracking?`เหลือ ${product.stock}`:'ไม่ติดตามสต็อก'}}</small></div><b>฿{{Number(product.price).toFixed(0)}}</b></button></div></section>
  </template>
</main>
</template>
