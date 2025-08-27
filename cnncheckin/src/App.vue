<script setup lang="ts">
import { RouterLink, RouterView } from 'vue-router'
import HelloWorld from './components/HelloWorld.vue'
</script>

<template>
  <header>
    <img alt="Vue logo" class="logo" src="@/assets/logo.svg" width="125" height="125" />

    <div class="wrapper">
      <HelloWorld msg="You did it!" />

      <nav>
        <RouterLink to="/">Home</RouterLink>
        <RouterLink to="/camera" class="nav-link">📸 Câmera</RouterLink>
        <RouterLink to="/about">About</RouterLink>
      </nav>
    </div>
  </header>

  <main class="main-content">
    <RouterView />
  </main>
</template>


<style scoped>
#app {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

.navbar {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  box-shadow: 0 2px 10px rgba(0,0,0,0.1);
  position: sticky;
  top: 0;
  z-index: 100;
}

.nav-container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 20px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  height: 60px;
}

.nav-logo a {
  font-size: 1.5rem;
  font-weight: bold;
  color: white;
  text-decoration: none;
  transition: color 0.3s;
}

.nav-logo a:hover {
  color: #f0f0f0;
}

.nav-menu {
  display: flex;
  gap: 30px;
  align-items: center;
}

.nav-link {
  color: white;
  text-decoration: none;
  font-weight: 500;
  padding: 8px 16px;
  border-radius: 20px;
  transition: all 0.3s;
  position: relative;
}

.nav-link:hover {
  background: rgba(255,255,255,0.2);
  transform: translateY(-1px);
}

.nav-link.router-link-active {
  background: rgba(255,255,255,0.25);
  backdrop-filter: blur(10px);
}

.main-content {
  flex: 1;
  background: #f8f9fa;
  min-height: calc(100vh - 60px);
}

/* Responsividade */
@media (max-width: 768px) {
  .nav-container {
    padding: 0 15px;
    height: 50px;
  }
  
  .nav-logo a {
    font-size: 1.3rem;
  }
  
  .nav-menu {
    gap: 15px;
  }
  
  .nav-link {
    padding: 6px 12px;
    font-size: 0.9rem;
  }
  
  .main-content {
    min-height: calc(100vh - 50px);
  }
}

@media (max-width: 480px) {
  .nav-container {
    flex-direction: column;
    height: auto;
    padding: 10px 15px;
  }
  
  .nav-menu {
    margin-top: 10px;
    gap: 10px;
  }
  
  .nav-link {
    padding: 5px 10px;
    font-size: 0.8rem;
  }
  
  .main-content {
    min-height: calc(100vh - 80px);
  }
}
</style>
