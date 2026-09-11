<script setup lang="ts">
import AdminApp from './AdminApp.vue'
import CustomerApp from './CustomerApp.vue'
import MerchantApp from './MerchantApp.vue'
const isAdmin = window.location.pathname.startsWith('/admin')
const isMerchant = window.location.pathname.startsWith('/merchant')
</script>

<template><AdminApp v-if="isAdmin" /><MerchantApp v-else-if="isMerchant" /><CustomerApp v-else /></template>
