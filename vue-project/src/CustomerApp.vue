<script setup lang="ts">
import { computed, onMounted, ref } from 'vue'
import liff from '@line/liff'

type Customer = { id: string; display_name: string; picture_url: string | null; role: 'customer' }
type Store = { id: string; name: string; is_open: boolean; queue_count: number }
type Product = { id: string; name: string; price: number; stock: number; is_tracking: boolean }

const customer = ref<Customer | null>(null)
const stores = ref<Store[]>([])
const selectedStore = ref<Store | null>(null)
const products = ref<Product[]>([])
const busy = ref(false)
const message = ref('')
const ready = ref(false)
const liffId = import.meta.env.VITE_LIFF_ID?.trim()
const apiUrl = import.meta.env.VITE_API_BASE_URL?.replace(/\/$/, '')
const pageTitle = computed(() => selectedStore.value ? selectedStore.value.name : 'เลือกร้านอาหาร')

async function api(path: string) {
  const token = liff.getIDToken()
  if (!token) throw new Error('กรุณาเปิดระบบผ่าน LINE อีกครั้ง')
  const response = await fetch(`${apiUrl}${path}`, {
    headers: { Authorization: `Bearer ${token}` }, cache: 'no-store', signal: AbortSignal.timeout(15000),
  })
  if (response.status === 401) throw new Error('การเข้าสู่ระบบหมดอายุ กรุณาเปิดระบบผ่าน LINE อีกครั้ง')
  if (!response.ok) throw new Error('โหลดข้อมูลไม่สำเร็จ กรุณาลองใหม่')
  return response.json()
}

async function loadAccount() {
  const data = await api('/auth/me')
  customer.value = data.customer
  const catalog = await api('/customer/stores')
  stores.value = catalog.stores
}

async function openStore(store: Store) {
  busy.value = true; message.value = ''
  try {
    const data = await api(`/customer/stores/${encodeURIComponent(store.id)}/products`)
    selectedStore.value = store
    products.value = data.products
  } catch (error) { message.value = error instanceof Error ? error.message : 'โหลดข้อมูลไม่สำเร็จ' }
  finally { busy.value = false }
}

function backToStores() { selectedStore.value = null; products.value = []; message.value = '' }

async function start() {
  busy.value = true; message.value = ''
  try {
    if (!liffId || !apiUrl) throw new Error('ระบบยังตั้งค่าไม่ครบ')
    await liff.init({ liffId }); ready.value = true
    if (liff.isLoggedIn()) await loadAccount()
  } catch (error) { message.value = error instanceof Error ? error.message : 'เปิดระบบไม่สำเร็จ' }
  finally { busy.value = false }
}

function login() { if (ready.value) liff.login({ redirectUri: window.location.origin + '/' }) }
onMounted(start)
</script>

<template>
  <main class="page">
    <header class="topbar">
      <button v-if="selectedStore" class="icon-button" aria-label="ย้อนกลับ" @click="backToStores">←</button>
      <div><strong>CANTEEN</strong><span>สำหรับลูกค้า</span></div>
      <img v-if="customer?.picture_url" :src="customer.picture_url" alt="รูปโปรไฟล์ LINE" class="avatar" />
    </header>

    <section v-if="!customer" class="login-card" :aria-busy="busy">
      <span class="eyebrow">มื้ออร่อย เริ่มที่นี่</span>
      <h1>โรงอาหาร<br />ในมือคุณ</h1>
      <p class="intro">เชื่อมต่อบัญชี LINE เพื่อเลือกร้านและดูเมนูอาหาร</p>
      <p v-if="busy" class="notice">กำลังเชื่อมต่อบัญชี…</p>
      <p v-else-if="message" role="alert" class="error">{{ message }}</p>
      <button v-if="ready && !liff.isLoggedIn()" class="primary" @click="login">เข้าสู่ระบบด้วย LINE</button>
      <button v-else-if="!busy && message" class="primary" @click="start">ลองอีกครั้ง</button>
    </section>

    <section v-else class="content">
      <div class="greeting"><span>สวัสดี</span><strong>{{ customer.display_name }}</strong></div>
      <h1 class="section-title">{{ pageTitle }}</h1>
      <p v-if="message" role="alert" class="error">{{ message }}</p>
      <div v-if="busy" class="notice">กำลังโหลด…</div>

      <div v-else-if="!selectedStore" class="store-list">
        <button v-for="store in stores" :key="store.id" class="store-card" @click="openStore(store)">
          <span class="store-icon">⌂</span>
          <span class="store-info"><strong>{{ store.name }}</strong><small>รออยู่ {{ store.queue_count }} คิว</small></span>
          <span class="arrow">›</span>
        </button>
        <div v-if="stores.length === 0" class="empty">ยังไม่มีร้านค้าเปิดให้บริการ</div>
      </div>

      <div v-else class="product-list">
        <article v-for="product in products" :key="product.id" class="product-card">
          <div><h2>{{ product.name }}</h2><p>฿{{ Number(product.price).toFixed(0) }}</p></div>
          <span v-if="product.is_tracking" :class="['stock', { out: product.stock <= 0 }]">
            {{ product.stock > 0 ? `เหลือ ${product.stock}` : 'หมด' }}
          </span>
        </article>
        <div v-if="products.length === 0" class="empty">ร้านนี้ยังไม่มีเมนูอาหาร</div>
      </div>
    </section>
    <footer>ระบบใหม่ · Vue + FastAPI + Supabase</footer>
  </main>
</template>
