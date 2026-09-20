# 前端模板（Vue3 + Vite + TypeScript + Element Plus）

写入 `web/` 目录下对应路径。占位符 `{{PROJECT_NAME}}`、`{{PROJECT_TITLE}}`、`{{SERVER_PORT}}` 替换后写入。
只创建模板中出现的目录；`web/public/` 放 `.gitkeep` 占位，其余目录（assets/components/composables/constants/directives/enums/lang/types 等）按需创建，不预置空目录。

## web/package.json

```json
{
  "name": "{{PROJECT_NAME}}-web",
  "description": "{{PROJECT_TITLE}} 前端",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vue-tsc --noEmit && vite build",
    "build-only": "vite build",
    "preview": "vite preview",
    "type-check": "vue-tsc --noEmit"
  },
  "dependencies": {
    "@element-plus/icons-vue": "^2.3.2",
    "axios": "^1.12.2",
    "element-plus": "^2.11.4",
    "pinia": "^3.0.3",
    "vue": "^3.5.22",
    "vue-router": "^4.5.1"
  },
  "devDependencies": {
    "@types/node": "^24.7.1",
    "@vitejs/plugin-vue": "^6.0.1",
    "sass": "^1.93.2",
    "typescript": "^6.0.3",
    "vite": "^7.1.9",
    "vue-tsc": "^3.3.11"
  },
  "engines": {
    "node": "^20.19.0 || >=22.12.0"
  }
}
```

## web/vite.config.ts

```ts
import vue from "@vitejs/plugin-vue";
import { type ConfigEnv, type UserConfig, loadEnv, defineConfig } from "vite";
import { resolve } from "path";

// Vite配置  https://cn.vitejs.dev/config
export default defineConfig(({ mode }: ConfigEnv): UserConfig => {
  const env = loadEnv(mode, process.cwd());

  return {
    resolve: {
      alias: {
        "@": resolve(__dirname, "src"),
      },
    },
    server: {
      host: "0.0.0.0",
      port: +(env.VITE_APP_PORT || 3000),
      open: true,
      proxy: {
        // 代理 /api 的请求到后端
        "/api": {
          changeOrigin: true,
          target: env.VITE_APP_API_URL,
        },
      },
    },
    plugins: [vue()],
    build: {
      outDir: "dist",
      emptyOutDir: true,
      chunkSizeWarningLimit: 2000,
    },
  };
});
```

## web/tsconfig.json

```json
{
  "compilerOptions": {
    "target": "esnext",
    "module": "esnext",
    "moduleResolution": "bundler",
    "lib": ["esnext", "dom"],
    "paths": {
      "@/*": ["./src/*"]
    },

    // 严格性和类型检查相关配置
    "strict": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,

    // 模块和兼容性相关配置
    "allowSyntheticDefaultImports": true,
    "esModuleInterop": true,
    "resolveJsonModule": true,

    // 调试和兼容性相关配置
    "sourceMap": true,
    "useDefineForClassFields": true,
    "allowJs": true,

    // 类型声明相关配置
    "types": ["node", "vite/client", "element-plus/global"]
  },

  "include": ["src/**/*.ts", "src/**/*.vue", "vite.config.ts"],
  "exclude": ["node_modules", "dist"]
}
```

## web/index.html

```html
<!doctype html>
<html lang="zh-CN">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <link rel="icon" href="/favicon.ico" />
    <title>{{PROJECT_TITLE}}</title>
  </head>
  <body>
    <div id="app"></div>
    <script type="module" src="/src/main.ts"></script>
  </body>
</html>
```

## web/.env.development

```
# 应用端口
VITE_APP_PORT=3000
# 项目名称
VITE_APP_TITLE={{PROJECT_TITLE}}
# 后端接口地址
VITE_APP_API_URL=http://127.0.0.1:{{SERVER_PORT}}
```

## web/.env.production

```
# 项目名称
VITE_APP_TITLE={{PROJECT_TITLE}}
# 后端接口地址（生产环境为空时走 nginx /api 反代）
VITE_APP_API_URL=
```

## web/src/main.ts

```ts
import { createApp } from "vue";
import App from "./App.vue";
import { setupPlugins } from "@/plugins";
import "@/styles/index.scss";

const app = createApp(App);
// 注册插件
setupPlugins(app);
app.mount("#app");
```

## web/src/App.vue

```vue
<template>
  <el-config-provider>
    <router-view />
  </el-config-provider>
</template>

<script setup lang="ts"></script>
```

## web/src/settings.ts

```ts
// 平台默认配置项
export const defaultSettings = {
  // 项目名称
  title: (import.meta.env.VITE_APP_TITLE as string) || "{{PROJECT_TITLE}}",
};
```

## web/src/plugins/index.ts

```ts
import type { App } from "vue";
import { createPinia } from "pinia";
import router from "@/router";
import ElementPlus from "element-plus";
import zhCn from "element-plus/es/locale/lang/zh-cn";
import "element-plus/dist/index.css";

export function setupPlugins(app: App) {
  // 状态管理
  app.use(createPinia());
  // 路由
  app.use(router);
  // Element Plus
  app.use(ElementPlus, { locale: zhCn });
}
```

## web/src/router/index.ts

```ts
import { createRouter, createWebHashHistory, type RouteRecordRaw } from "vue-router";

const Layout = () => import("@/layouts/index.vue");

// 静态路由
export const constantRoutes: RouteRecordRaw[] = [
  {
    path: "/login",
    component: () => import("@/views/login/index.vue"),
    meta: { hidden: true },
  },
  {
    path: "/",
    name: "/",
    component: Layout,
    redirect: "/dashboard",
    children: [
      {
        path: "dashboard",
        component: () => import("@/views/dashboard/index.vue"),
        name: "Dashboard",
        meta: { title: "首页" },
      },
    ],
  },
];

const router = createRouter({
  history: createWebHashHistory(),
  routes: constantRoutes,
});

export default router;
```

## web/src/layouts/index.vue

```vue
<template>
  <el-container class="layout">
    <el-aside width="200px">
      <div class="logo">{{ title }}</div>
      <el-menu router :default-active="route.path">
        <el-menu-item index="/dashboard">
          <span>首页</span>
        </el-menu-item>
      </el-menu>
    </el-aside>
    <el-container>
      <el-header>{{ title }}</el-header>
      <el-main>
        <router-view />
      </el-main>
    </el-container>
  </el-container>
</template>

<script setup lang="ts">
import { useRoute } from "vue-router";
import { defaultSettings } from "@/settings";

const route = useRoute();
const title = defaultSettings.title;
</script>

<style scoped lang="scss">
.layout {
  height: 100vh;
}

.logo {
  height: 60px;
  line-height: 60px;
  text-align: center;
  font-weight: bold;
}
</style>
```

## web/src/store/index.ts

```ts
// pinia store 模块统一放在 modules/ 目录，例如 modules/app.ts
// export const useAppStore = defineStore("app", () => { ... });
```

## web/src/utils/request.ts

```ts
import axios, { type AxiosInstance } from "axios";
import { ElMessage } from "element-plus";

// axios 实例
const service: AxiosInstance = axios.create({
  baseURL: import.meta.env.VITE_APP_API_URL as string,
  timeout: 10000,
});

// 响应拦截器（后端统一返回 { code, data, msg, now }，code 为 0 表示成功）
service.interceptors.response.use(
  (response) => {
    const res = response.data;
    if (res.code !== 0) {
      ElMessage.error(res.msg || "请求失败");
      return Promise.reject(new Error(res.msg || "Error"));
    }
    return res.data;
  },
  (error) => {
    ElMessage.error(error.message || "网络错误");
    return Promise.reject(error);
  },
);

export default service;
```

## web/src/api/demo-api.ts

```ts
import request from "@/utils/request";

export interface DemoHelloOut {
  message: string;
}

// 调用后端示例接口 GET /api/v1/demo/hello
export function getDemoHello(params: { name?: string }) {
  return request.get<any, DemoHelloOut>("/api/v1/demo/hello", { params });
}
```

## web/src/views/dashboard/index.vue

```vue
<template>
  <el-card>
    <template #header>首页</template>
    <p>{{ message }}</p>
    <el-button type="primary" @click="fetchHello">调用后端示例接口</el-button>
  </el-card>
</template>

<script setup lang="ts">
import { ref } from "vue";
import { getDemoHello } from "@/api/demo-api";

const message = ref("欢迎使用 {{PROJECT_TITLE}}");

async function fetchHello() {
  try {
    const data = await getDemoHello({ name: "{{PROJECT_NAME}}" });
    message.value = data.message;
  } catch (e) {
    message.value = "接口调用失败，请确认后端已启动";
  }
}
</script>
```

## web/src/views/login/index.vue

```vue
<template>
  <div class="login">
    <el-card class="login-card">
      <template #header>{{ title }}</template>
      <el-form>
        <el-form-item>
          <el-input v-model="username" placeholder="用户名" />
        </el-form-item>
        <el-form-item>
          <el-input v-model="password" type="password" placeholder="密码" show-password />
        </el-form-item>
        <el-button type="primary" class="w-full" @click="handleLogin">登 录</el-button>
      </el-form>
    </el-card>
  </div>
</template>

<script setup lang="ts">
import { ref } from "vue";
import { ElMessage } from "element-plus";
import { defaultSettings } from "@/settings";

const title = defaultSettings.title;
const username = ref("");
const password = ref("");

function handleLogin() {
  // TODO: 对接登录接口后跳转
  ElMessage.info("登录功能待实现");
}
</script>

<style scoped lang="scss">
.login {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 100vh;
}

.login-card {
  width: 380px;
}
</style>
```

## web/src/styles/index.scss

```scss
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

html,
body,
#app {
  height: 100%;
}
```
