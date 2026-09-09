<script setup lang="ts">
import { onMounted, ref } from 'vue'
import liff from '@line/liff'

type Customer = { id: string; display_name: string; picture_url: string | null; role: 'customer' }
const customer = ref<Customer | null>(null)
const busy = ref(false)
const message = ref('')
const liffId = import.meta.env.VITE_LIFF_ID?.trim()
const apiUrl = import.meta.env.VITE_API_BASE_URL?.replace(/\/$/, '')
const ready = ref(false)
const expired = ref(false)

async function loadAccount() {
  busy.value = true
  message.value = ''
  expired.value = false
  customer.value = null
  try {
    const token = liff.getIDToken()
    if (!token) throw new Error('กรุณาปิดหน้านี้และเปิดจาก LINE อีกครั้งเพื่อเข้าสู่ระบบ')
    const response = await fetch(`${apiUrl}/auth/me`, {
      headers: { Authorization: `Bearer ${token}` },
      signal: AbortSignal.timeout(15000),
      cache: 'no-store',
    })
    if (response.status === 401) {
      expired.value = true
      throw new Error('การเข้าสู่ระบบหมดอายุ กรุณาปิดหน้านี้และเปิดจาก LINE อีกครั้ง')
    }
    if (!response.ok) throw new Error('ยังเชื่อมต่อบัญชีไม่ได้ กรุณาลองใหม่อีกครั้ง')
    const data = await response.json()
    customer.value = data.customer
  } catch (error) {
    message.value = error instanceof Error ? error.message : 'เชื่อมต่อไม่สำเร็จ'
  } finally {
    busy.value = false
  }
}

async function start() {
  busy.value = true
  message.value = ''
  try {
    if (!liffId || !apiUrl) throw new Error('ระบบกำลังเตรียมเปิดให้บริการ กรุณาลองใหม่ภายหลัง')
    await liff.init({ liffId })
    ready.value = true
    if (liff.isLoggedIn()) await loadAccount()
  } catch (error) {
    message.value = error instanceof Error ? error.message : 'เปิดระบบไม่สำเร็จ'
  } finally {
    busy.value = false
  }
}

function login() {
  if (ready.value) liff.login()
}
onMounted(start)
</script>

<template>
  <main class="page">
    <div class="brand">CANTEEN <span>• สำหรับลูกค้า</span></div>
    <section class="card" :aria-busy="busy">
      <span class="eyebrow">มื้ออร่อย เริ่มที่นี่</span>
      <h1>โรงอาหาร<br />ในมือคุณ</h1>
      <p class="intro">เชื่อมต่อบัญชี LINE ของคุณ<br />เพื่อเตรียมพร้อมสำหรับการสั่งอาหาร</p>
      <div v-if="busy" role="status" class="notice">กำลังเชื่อมต่อบัญชี…</div>
      <div v-else-if="customer" class="account">
        <span class="check">✓</span>
        <h2>สวัสดี {{ customer.display_name }}</h2>
        <p>เชื่อมต่อบัญชี LINE สำเร็จแล้ว</p>
        <p class="muted">ระบบเลือกร้านและสั่งอาหารกำลังอยู่ระหว่างพัฒนา</p>
        <button class="secondary" @click="loadAccount">ตรวจสอบบัญชีอีกครั้ง</button>
      </div>
      <template v-else>
        <p v-if="message" role="alert" class="error">{{ message }}</p>
        <button v-if="ready && !liff.isLoggedIn()" @click="login">เข้าสู่ระบบด้วย LINE</button>
        <button v-else-if="message && !expired" @click="start">ลองอีกครั้ง</button>
        <p class="muted">ใช้บัญชี LINE ของคุณ ไม่ต้องตั้งรหัสผ่านใหม่</p>
      </template>
    </section>
    <footer>พื้นที่สำหรับลูกค้า · ร้านค้าใช้เว็บไซต์จัดการร้านแยกต่างหาก</footer>
  </main>
</template>
