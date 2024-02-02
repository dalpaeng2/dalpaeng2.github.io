---
title: 'Supabase를 사용하여 Nextjs에서 로그인 기능 구현하기'
date: 2024-01-23T21:29:49+09:00
---

Next.js 앱에서 supabase auth를 사용해서 로그인 하는 방법을 정리합니다.

## Client Component를 사용한 로그인

### supabase 클라이언트 만들기

```typescript
import { createBrowserClient } from '@supabase/ssr';

export function supabaseBrowser() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
}
```

### 클라이언트 컴포넌트에서 로그인

```typescript
'use client';

import { supabaseBrowser } from '@/utils/supabase/browser';
import Link from 'next/link';
import React from 'react';

export default function Page() {
  const supabase = supabaseBrowser();

  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();

    await supabase.auth.signInWithPassword({
      email: (e.currentTarget.elements[0] as HTMLInputElement).value,
      password: (e.currentTarget.elements[1] as HTMLInputElement).value,
    });
  };

  return (
    <main className="flex flex-col items-center justify-between p-24">
      <h1>Log In</h1>
      <form className="flex flex-col gap-2" onSubmit={handleSubmit}>
        <input type="email" id="email" placeholder="Email" />
        <input type="password" id="password" placeholder="Password" />
        <button type="submit" className="bg-blue-500 p-1 rounded-xl">
          Login
        </button>
      </form>
      <div>
        Go to <Link href="/auth/signin">Sign In</Link>
      </div>
    </main>
  );
}
```

## Server Component Client를 사용한 로그인

### supabase 클라이언트 만들기

```typescript
import { CookieOptions, createServerClient } from '@supabase/ssr';
import { cookies } from 'next/headers';

export function supabaseServer() {
  const cookieStore = cookies();
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        get(name: string) {
          return cookieStore.get(name)?.value;
        },
        set(name: string, value: string, options: CookieOptions) {
          cookieStore.set({ name, value, ...options });
        },
        remove(name: string, options: CookieOptions) {
          cookieStore.delete({ name, ...options });
        },
      },
    }
  );
}
```

### API를 사용한 로그인

로그인 Form

```typescript
export default function Page() {
  const [email, setEmail] = React.useState('');
  const [password, setPassword] = React.useState('');

  const onChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    if (e.target.id === 'email') {
      setEmail(e.target.value);
    } else {
      setPassword(e.target.value);
    }
  };

  return (
    <main className="flex flex-col items-center justify-between p-24">
      <h1>Log In</h1>
      <form
        action="/api/auth/login"
        method="POST"
        className="flex flex-col gap-2"
      >
        <input
          type="email"
          id="email"
          name="email"
          placeholder="Email"
          value={email}
          onChange={onChange}
        />
        <input
          type="password"
          id="password"
          name="password"
          placeholder="Password"
          value={password}
          onChange={onChange}
        />
        <button type="submit" className="bg-blue-500 p-1 rounded-xl">
          Login
        </button>
      </form>
      <div>
        Go to <Link href="/auth/signin">Sign In</Link>
      </div>
    </main>
  );
}
```

로그인 API Handler

```typescript
import { supabaseServer } from '@/utils/supabase/server';
import { NextResponse } from 'next/server';

export const dynamic = 'force-dynamic'; // defaults to auto
export async function POST(request: Request) {
  const supabase = supabaseServer();
  const requestUrl = new URL(request.url);
  const formData = await request.formData();
  const email = formData.get('email') as string;
  const password = formData.get('password') as string;
  const data = await supabase.auth.signInWithPassword({
    email,
    password,
  });

  return NextResponse.redirect(requestUrl.origin, {
    status: 301,
  });
}
```

### Server Action을 사용한 로그인

server action

```typescript
'use server';
import { supabaseServer } from '@/utils/supabase/server';
import { redirect } from 'next/navigation';

export async function login(formData: FormData) {
  const supabase = supabaseServer();

  const email = formData.get('email') as string;
  const password = formData.get('password') as string;
  const data = await supabase.auth.signInWithPassword({
    email,
    password,
  });

  await supabase.auth.signInWithPassword({ email, password });

  redirect('/');
}
```

로그인 Form

```typescript
'use client';

import { login } from '@/app/action';
import { supabaseBrowser } from '@/utils/supabase/browser';
import Link from 'next/link';
import React from 'react';

export default function Page() {
  const [email, setEmail] = React.useState('');
  const [password, setPassword] = React.useState('');

  const onChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    if (e.target.id === 'email') {
      setEmail(e.target.value);
    } else {
      setPassword(e.target.value);
    }
  };

  return (
    <main className="flex flex-col items-center justify-between p-24">
      <h1>Log In</h1>
      <form action={login} className="flex flex-col gap-2">
        <input
          type="email"
          id="email"
          name="email"
          placeholder="Email"
          value={email}
          onChange={onChange}
        />
        <input
          type="password"
          id="password"
          name="password"
          placeholder="Password"
          value={password}
          onChange={onChange}
        />
        <button type="submit" className="bg-blue-500 p-1 rounded-xl">
          Login
        </button>
      </form>
      <div>
        Go to <Link href="/auth/signin">Sign In</Link>
      </div>
    </main>
  );
}
```
