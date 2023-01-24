---
layout: ../../layouts/PostLayout.astro
title: มาลองใช้ Next-Auth.js ทํา Authentication กัน
author: Weerawong Vonggatunyu
authorImg:
slug: next-auth
categories: [next.js, next-auth]
createdAt: 1-24-2023
editedAt: 1-24-2023
thumbnail: https://next-auth.js.org/img/social-media-card.png
description:
---

ในวันนี้้ผมจะพาทุกคนมาลองใช้ NextAuth ทําระบบ Authentication แบบง่ายๆ กันนะครับ โดยสิ่งที่ทุกคนต้องมีก็คือโปรเจค Next.js ถ้ายังไม่มีให้สร้างขึ้นมาโดยใช้คําสั่ง

```bash
npx create-next-app@latest
# or
yarn create next-app
# or
pnpm create next-app
```

## ติดตั้ง NextAuth

```bash
npm install next-auth
```

สร้างไฟล์ชื่อ `[...nextauth].ts` ใน `pages/api/auth`
แล้วกําหนด provider ที่จะใช้ในที่นี้จะใช้ของ Google ครับ

```ts
import NextAuth from 'next-auth'
import GoogleProvider from 'next-auth/providers/google'
import type { NextAuthOptions } from 'next-auth'

export const authOptions: NextAuthOptions = {
  providers: [
    GoogleProvider({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_SECRET!,
    }),
  ],
}

export default NextAuth(authOptions)
```

โดยเราต้องมี Google Client id และ Secret id ก่อนนะครับโดยสามารถเอามาได้จากที่ [Google Developer Console](https://console.developers.google.com)
โดยเข้าไปในส่วน Credentials เลือกสร้าง Credentials และเลือก OAuth Client ID ครับ เอาเก็บใน `.env` ได้เลยครับ

จากนั้นไปที่ `_app.tsx` แล้วทําการเอา `SessionProvider` มาครอบตัว Component เอาไว้เราจะได้เรียก `useSession` ได้จากทุกที่ใน app ซึ่งก็จะเป็นตัวที่เราจะใช้เรียก `session` เพื่อยืนยันว่า user authenticated แล้วหรือยังนั่นเอง

```ts
import { SessionProvider } from 'next-auth/react'
export default function App({
  Component,
  pageProps: { session, ...pageProps },
}) {
  return (
    <SessionProvider session={session}>
      <Component {...pageProps} />
    </SessionProvider>
  )
}
```

ใน `index.tsx` มาสร้าง ui หน้าล็อกอินเพื่อลองใช้งานกันเลยครับ

```ts
import { useSession, signIn, signOut } from 'next-auth/react'
export default function Component() {
  const { data: session } = useSession()
  if (session) {
    return (
      <>
        Signed in as {session.user.email} <br />
        <button onClick={() => signOut()}>Sign out</button>
      </>
    )
  }
  return (
    <>
      Not signed in <br />
      <button onClick={() => signIn('google')}>Sign in</button>
    </>
  )
}
```

จากโค้ดมีการเรียก `session` จาก `useSession()` ซึ่งจะทําการเช็คว่าผู้ใช้เข้าสู่ระบบแล้วหรือยัง ถ้ายังจะโชว์ปุ่ม Sign in ถ้าไม่โชว์ปุ่ม Sign Out

โดยในที่นี้เราใช้ฟังก์ชัน `signIn()` จาก NextAuth ซึ่งจะทําการ redirect เราไปยังหน้า sign in ของ google ค่าที่ใส่เข้าเป็นการบอกว่าเราจะใช้บริการของ google ซึ่งเมื่อ sign in สําหรับก็จะได้ token ไปเก็บไว้ใน session แค่นี้ก็ยืนยันตัวตนได้แล้ว

และ `signOut()` ก็จะทําการ clear session เพื่อพาเราออกจากระบบ

## Conclusion

เพียงเท่านี้เราก็จะได้ระบบ Authentication แบบง่ายๆ กันแล้วนะครับ โดย NextAuth ยังมี feature ต่างๆ อยู่อีกมากมายสามารถไปศึกษาเพิ่มเติมกันได้ที่ [documentation](https://next-auth.js.org/) เลยครับ

## Reference

- https://next-auth.js.org/getting-started/example

---
