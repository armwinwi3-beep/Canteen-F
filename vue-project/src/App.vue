<script setup lang="ts">
import AdminApp from './AdminApp.vue'
import CustomerApp from './CustomerApp.vue'
const isAdmin = window.location.pathname.startsWith('/admin')
</script>

<template><AdminApp v-if="isAdmin" /><CustomerApp v-else /></template>
