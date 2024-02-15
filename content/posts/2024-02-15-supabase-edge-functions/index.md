---
title: Supabase Edge Functions
date: 2024-02-15T21:04:01+09:00
tags: []
---

## Edge function 만들기

먼저 supabase 프로젝트를 초기화 합니다.

```bash
$ supabase init
```

다음으로 supabase cli를 사용해 edge function을 만듭니다.

```bash
$ supabase functions new my-function
```

## 배포하기

supabase cli를 사용해서 edge function을 배포합니다.

배포하기 전에 먼저 원격 프로젝트와 연결합니다.

```bash
# 프로젝트 연결
$ supabase link --project-ref your-project-id

# 모든 function 배포
$ supabase functions deploy

# 특정 function 배포
$ supabase functions deploy hello-world

# 특정 function을 JWT 인증 없이 배포
$ supabase functions deploy hello-world --no-verify-jwt
```

cli를 사용해서 배포하기 위해서는 Docker가 필요합니다.

> Docker required
>
> Since Supabase CLI version 1.123.4, you must have [Docker Desktop](https://docs.docker.com/desktop/) installed to deploy Edge Functions.

다음과 같은 간단한 edge function을 배포해서 호출한 결과입니다.

```tsx
Deno.serve(async (req) => {
  const { name } = await req.json();
  const data = {
    message: `Hello ${name}!`,
  };

  return new Response(JSON.stringify(data), {
    headers: { 'Content-Type': 'application/json' },
  });
});
```

```bash
➜ curl -XPOST https://{PROJECT}.supabase.co/functions/v1/hello-world -d '{"name": "World"}'
{"message":"Hello World!"}%
```

## Supabase client 만들기

supabase client를 생성해서 db, storage등을 접근할 수 있습니다.

먼저 관리자용 supabase client에서는 SUPABASE_SERVICE_ROLE_KEY 를 인자로 supabase client를 만듭니다.

```tsx
// 관리자 계정 supabase client 만들기
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

export function supabaseAdmin({ authHeader }: { authHeader: string }) {
  return createClient(
    Deno.env.get('SUPABASE_URL') ?? '',
    Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') ?? '',
    { global: { headers: { Authorization: authHeader } } }
  );
}
```

사용자 계정 supabase client는 SUPABASE_ANON_KEY **를 인자로 supabase client를 만듭니다.**

```tsx
// 사용자 계정 supabase client 만들기
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

export function supabaseClient({ authHeader }: { authHeader: string }) {
  return createClient(
    Deno.env.get('SUPABASE_URL') ?? '',
    Deno.env.get('SUPABASE_ANON_KEY') ?? '',
    { global: { headers: { Authorization: authHeader } } }
  );
}
```

edge function에서는 용도에 맞는 supabase client를 만들어서 사용합니다.

```tsx
import { supabaseClient } from '../_shared/supabaseClient';

Deno.serve(async (req) => {
  const supabase = supabaseClient({
    authHeader: req.headers.get('Authorization') ?? '',
  });

  const {
    data: { user },
  } = await supabase.auth.getUser();

  const { name } = await req.json();
  const data = {
    message: `Hello ${name}!`,
  };

  return new Response(JSON.stringify(data), {
    headers: { 'Content-Type': 'application/json' },
  });
});
```

## CORS 처리

edge function을 브라우저에서 호출하기 위해서는 CORS처리가 필요합니다.

\_shared 폴더에 cors.ts 파일을 만들어서 여러 edge function들이 해당 설정을 공유할 수 있도록 합니다.
