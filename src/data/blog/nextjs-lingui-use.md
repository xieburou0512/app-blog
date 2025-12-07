---
title: Next.js 16+ Turbopack 环境下集成 Lingui 的正确方式
author: xieburou
pubDatetime: 2025-12-07T12:00:00Z
slug: nextjs-lingui-turbopack-setup
featured: true
draft: false
tags:
  - Next.js
  - Lingui
  - Turbopack
  - i18n
description: Next.js 16+ 默认使用 Turbopack 替代 Webpack，导致 Lingui 官方示例失效。本文提供完整的适配方案。
---

Next.js 15+ 版本默认采用 Turbopack 作为构建工具，Lingui 官方文档中的 Webpack 配置示例已不再适用。本文将介绍如何在新版 Next.js 中正确集成 Lingui 国际化方案。

## 安装依赖
```bash
# 安装运行时依赖
bun add @lingui/react

# 安装开发依赖
bun add -d @lingui/cli @lingui/format-po @lingui/loader @lingui/swc-plugin
```

本文使用的版本：
```json
{
  "dependencies": {
    "@lingui/react": "^5.6.1"
  },
  "devDependencies": {
    "@lingui/cli": "^5.6.1",
    "@lingui/format-po": "^5.6.1",
    "@lingui/loader": "^5.6.1",
    "@lingui/swc-plugin": "^5.6.1"
  }
}
```

## 配置文件

### 1. Lingui 配置

在项目根目录创建 `lingui.config.ts`：
```typescript
import { formatter } from "@lingui/format-po";

/** @type {import('@lingui/conf').LinguiConfig} */
export default {
  locales: ["en", "zh-Hans"],
  sourceLocale: "en",
  fallbackLocales: {
    default: "en",
  },
  catalogs: [
    {
      path: "src/locales/{locale}",
      include: ["src/"],
    },
  ],
  format: formatter({
    lineNumbers: false,
    origins: false,
    printLinguiId: true,
  }),
};
```

### 2. Next.js 配置

修改 `next.config.ts`，添加 Turbopack 的 loader 规则和 SWC 插件：
```typescript
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  turbopack: {
    rules: {
      "*.po": {
        loaders: ["@lingui/loader"],
        as: "*.js",
      },
    },
  },
  experimental: {
    swcPlugins: [["@lingui/swc-plugin", {}]],
  },
};

export default nextConfig;
```

## 核心模块

在 `src` 目录下创建 `lingui` 文件夹和 `locales` 文件夹，用于存放国际化相关代码和翻译文件。

### 1. 中间件代理 (proxy.ts)

处理路由的语言前缀和自动重定向：
```typescript
import { type NextRequest, NextResponse } from "next/server";
import linguiConfig from "../lingui.config";

const { locales, sourceLocale } = linguiConfig;

export function proxy(request: NextRequest) {
  const { pathname } = request.nextUrl;

  // 检查路径是否已包含语言前缀
  const pathnameHasLocale = locales.some(
    (locale) => pathname.startsWith(`/${locale}/`) || pathname === `/${locale}`
  );

  if (pathnameHasLocale) {
    const locale = pathname.split("/")[1];
    const response = NextResponse.next();
    response.cookies.set("NEXT_LOCALE", locale, { path: "/" });
    return response;
  }

  // 自动添加语言前缀并重定向
  const locale = getRequestLocale(request);
  const url = request.nextUrl.clone();
  url.pathname = `/${locale}${pathname}`;
  return NextResponse.redirect(url);
}

function getRequestLocale(request: NextRequest): string {
  // 优先读取 cookie
  const cookieLocale = request.cookies.get("NEXT_LOCALE")?.value;
  if (cookieLocale && locales.includes(cookieLocale)) {
    return cookieLocale;
  }

  // 解析 Accept-Language 请求头
  const acceptLanguage = request.headers.get("accept-language");
  if (acceptLanguage) {
    const preferredLocale = acceptLanguage
      .split(",")
      .map((lang) => {
        const [locale, qValue] = lang.trim().split(";");
        const quality = qValue ? parseFloat(qValue.split("=")[1]) : 1;
        return { locale: locale.split("-")[0], quality };
      })
      .sort((a, b) => b.quality - a.quality)
      .find(({ locale }) => locales.includes(locale));

    if (preferredLocale) {
      return preferredLocale.locale;
    }
  }

  // 返回默认语言
  return sourceLocale as string;
}

export const config = {
  matcher: ["/((?!api|_next|_vercel|.*\\.).*)"],
};
```

### 2. 服务端 i18n 实例 (appRouterI18n.ts)

预加载所有语言的翻译目录：
```typescript
import "server-only";

import { I18n, Messages, setupI18n } from "@lingui/core";
import linguiConfig from "../../lingui.config";

const { locales } = linguiConfig;

type SupportedLocales = string;

async function loadCatalog(
  locale: SupportedLocales
): Promise<{ [k: string]: Messages }> {
  const { messages } = await import(`../locales/${locale}.po`);
  return { [locale]: messages };
}

const catalogs = await Promise.all(locales.map(loadCatalog));

export const allMessages = catalogs.reduce((acc, oneCatalog) => {
  return { ...acc, ...oneCatalog };
}, {});

type AllI18nInstances = { [K in SupportedLocales]: I18n };

export const allI18nInstances: AllI18nInstances = locales.reduce(
  (acc, locale) => {
    const messages = allMessages[locale] ?? {};
    const i18n = setupI18n({
      locale,
      messages: { [locale]: messages },
    });
    return { ...acc, [locale]: i18n };
  },
  {}
);
```

### 3. 初始化工具 (initLingui.ts)

用于在服务端组件中初始化 Lingui：
```typescript
import "server-only";

import { setI18n } from "@lingui/react/server";
import { allI18nInstances } from "./appRouterI18n";
import { I18n } from "@lingui/core";

type SupportedLocales = string;

export type PageLangParam = {
  params: Promise<{ lang: string }>;
};

export const getI18nInstance = (locale: SupportedLocales): I18n => {
  if (!allI18nInstances[locale]) {
    console.warn(`No i18n instance found for locale "${locale}"`);
  }
  return allI18nInstances[locale]! || allI18nInstances["en"]!;
};

export const initLingui = (lang: string) => {
  const i18n = getI18nInstance(lang);
  setI18n(i18n);
  return i18n;
};
```

### 4. 客户端 Provider (LinguiClientProvider.tsx)

为客户端组件提供国际化上下文：
```tsx
"use client";

import { I18nProvider } from "@lingui/react";
import { type Messages, setupI18n } from "@lingui/core";
import { useState } from "react";

type Props = {
  children: React.ReactNode;
  initialLocale: string;
  initialMessages: Messages;
};

export function LinguiClientProvider({
  children,
  initialLocale,
  initialMessages,
}: Props) {
  const [i18n] = useState(() => {
    return setupI18n({
      locale: initialLocale,
      messages: { [initialLocale]: initialMessages },
    });
  });

  return <I18nProvider i18n={i18n}>{children}</I18nProvider>;
}
```

## 在 App Router 中使用

### 语言路由 Layout (`app/[lang]/layout.tsx`)
```tsx
import { LinguiClientProvider } from "@/lingui/LinguiClientProvider";
import { PropsWithChildren } from "react";
import linguiConfig from "../../../lingui.config";
import { initLingui, PageLangParam } from "@/lingui/initLingui";
import { allMessages } from "@/lingui/appRouterI18n";

export default async function LangLayout({
  children,
  params,
}: PropsWithChildren<PageLangParam>) {
  const { lang } = await params;

  // 验证语言代码有效性，防止无效路径被误识别
  if (!linguiConfig.locales.includes(lang)) {
    return null;
  }

  initLingui(lang);

  return (
    <LinguiClientProvider
      initialLocale={lang}
      initialMessages={allMessages[lang]}
    >
      {children}
    </LinguiClientProvider>
  );
}
```

### 根 Layout (`app/layout.tsx`)
```tsx
import { PropsWithChildren } from "react";
import linguiConfig from "../../lingui.config";
import { PageLangParam } from "@/lingui/initLingui";

export default async function RootLayout({
  children,
  params,
}: PropsWithChildren<PageLangParam>) {
  const { lang } = await params;
  const validLang = linguiConfig.locales.includes(lang)
    ? lang
    : linguiConfig.sourceLocale;

  return (
    <html lang={validLang}>
      <body>{children}</body>
    </html>
  );
}
```

## 提取翻译文本

在 `package.json` 中添加脚本：
```json
{
  "scripts": {
    "i18n:extract": "lingui extract --clean"
  }
}
```

运行提取命令：
```bash
bun i18n:extract
```

执行后会在 `src/locales` 目录下生成各语言的 `.po` 文件，编辑这些文件添加翻译内容即可。

## 总结

通过以上配置，你可以在 Next.js 16+ 的 Turbopack 环境中顺利使用 Lingui 进行国际化开发。关键改动在于 `next.config.ts` 中需要使用 `turbopack.rules` 替代原来的 Webpack loader 配置。