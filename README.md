# NextPlot App

Portal สำหรับประกาศ/จัดการข้อมูลที่ดิน–อสังหา ใช้ **Supabase + Next.js** (หรือเวอร์ชัน static ในอนาคต) พร้อมระบบ:
- Email/Password Auth
- ensure_profile() (สร้าง profile อัตโนมัติหลังสมัคร/ล็อกอิน)
- Roles: USER / ADMIN / SUPER_ADMIN
- Listings: properties + favorites
- Promote ผู้ใช้อื่น (เฉพาะ SUPER_ADMIN)
- Search / Filter ราคา / Favorites
- (Roadmap) อัปโหลดรูป, Tag Management, Audit Logs

---

## 1. สถาปัตยกรรม (Overview)

| Layer | ใช้เทคโนโลยี | หน้าที่ |
|-------|---------------|---------|
| Frontend | Next.js (App Router) หรือ Static HTML | UI / Sign Up / Sign In / เรียก RPC |
| Backend (Serverless API Routes หรือ Edge) | Next.js API + Supabase client (Service Role สำหรับงานจำเป็น) | Logic เพิ่มเติม (Optional) |
| Database + Auth | Supabase (Postgres + RLS) | Users, profiles, properties, favorites |
| Storage (Roadmap) | Supabase Storage | เก็บรูป property |
| Deploy | Vercel (แนะนำ) + GitHub Pages (ถ้าส่วน static) | Hosting |

---

## 2. ตารางหลัก (Schema)

หลัก ๆ:
- `profiles(id UUID PK, role user_role, display_name, created_at)`
- `properties(id UUID, code UNIQUE, title, location, price, published, created_at)`
- `favorites(user_id, property_id, created_at)`
- (เพิ่มเติมตามสคริปต์: raw_ingest, knowledge_links, audit_logs, tags, property_images, property_tags)

### Enum
```sql
CREATE TYPE user_role AS ENUM ('USER','ADMIN','SUPER_ADMIN');
```

### ฟังก์ชันสำคัญ
```sql
app_user_role()      -- คืน role ของ auth.uid()
ensure_profile()     -- Idempotent สร้าง profile แถว (USER) ถ้ายังไม่มี
guard_profile_role() -- Trigger ป้องกันผู้ไม่ใช่ SUPER_ADMIN เปลี่ยน role
```

### Policies ตัวอย่าง (profile)
```sql
CREATE POLICY select_profiles_self ON profiles
  FOR SELECT USING (auth.uid() = id OR app_user_role() IN ('ADMIN','SUPER_ADMIN'));

CREATE POLICY update_profiles_self ON profiles
  FOR UPDATE USING (auth.uid() = id OR app_user_role() = 'SUPER_ADMIN')
  WITH CHECK (auth.uid() = id OR app_user_role() = 'SUPER_ADMIN');
```

---

## 3. Environment Variables

สร้างไฟล์ `.env.local` (สำหรับ Next.js) หรือใส่ผ่าน UI ของ Vercel (Project → Settings → Environment Variables)

| Key | ตัวอย่าง | หมายเหตุ |
|-----|----------|----------|
| NEXT_PUBLIC_SUPABASE_URL | https://xxxx.supabase.co | Public |
| NEXT_PUBLIC_SUPABASE_ANON_KEY | eyJhbGciOi... | Public (ฝั่ง Browser) |
| SUPABASE_SERVICE_ROLE_KEY | (ห้ามใส่ใน client) | ใช้เฉพาะ API server-side |
| SUPABASE_JWT_SECRET | (ดึงจาก Settings) | ใช้ verify token ฝั่ง server |

> ห้าม commit service_role key หรือ JWT secret ลง repo

ดูตัวอย่างไฟล์: `.env.example` (แนบไว้ใน repo)

---

## 4. โครงสร้างไฟล์ (แนะนำ Next.js)

```
nextplot-app/
  app/
    page.tsx            # หน้าแรก
    login/page.tsx      # (ถ้าแยกหน้า login)
    properties/page.tsx # List/Filters
  lib/
    supabaseClient.ts
  components/
    AuthForm.tsx
    PropertyList.tsx
  public/
    favicon.ico
  .env.local (ไม่ commit)
  .env.example
  README.md
```

---

## 5. การตั้งค่า Supabase Auth

1. Dashboard → Authentication → Providers → Email
2. ถ้าช่วงเริ่ม dev ไม่อยากยืนยันอีเมล: ปิด “Confirm email”
3. Allowed Redirect URLs: เพิ่ม  
   ```
   http://localhost:3000
   https://YOUR-PROJECT.vercel.app
   https://YOUR-USERNAME.github.io
   ```
4. Allowed Origins (Beta): เพิ่มโดเมนเดียวกับด้านบน (ถ้ามีช่อง)

---

## 6. ขั้นตอน Deployment (Vercel + GitHub)

1. Push โค้ดขึ้น GitHub (repo นี้)
2. เข้า https://vercel.com → New Project → Import repo
3. Build Command (auto): `next build`
4. Output: `.next`
5. ใส่ Environment Variables (ขั้นตอน 3)
6. Deploy → ได้โดเมนเช่น `https://nextplot-app.vercel.app`
7. Copy โดเมนไปเพิ่มใน Supabase Auth (Allowed redirect URLs)

---

## 7. Troubleshooting (Signup / Login)

| อาการ | สาเหตุหลัก | วิธีแก้ |
|-------|------------|---------|
| ปุ่ม Sign Up ไม่ทำงาน | ไม่มี onClick / component ไม่เป็น client component | ใส่ `\"use client\"` ด้านบนไฟล์ page.tsx หรือ component |
| Signup error: Invalid login credentials | Password ไม่ตรง policy (default ≥ 6 อักขระ) | ใช้ ≥ 8 ตัว ผสมตัวเลข |
| ensure_profile error: function does not exist | ยังไม่รันสคริปต์สร้างฟังก์ชัน | รันสคริปต์ reset/soft |
| Profile fetch error: permission denied | RLS policy ยังไม่สร้าง | รัน soft reset script |
| Cannot read properties of undefined (auth) | สร้าง supabase client ซ้ำ / import ผิด | ตรวจไฟล์ `lib/supabaseClient.ts` |
| 400 CORS / redirect not allowed | ยังไม่ได้ใส่โดเมนใน Allowed redirect URLs | ใส่โดเมน แล้วลองใหม่ |
| Promote error: Only SUPER_ADMIN can change role | ผู้ใช้ยังไม่เป็น SUPER_ADMIN | UPDATE ใน SQL: `UPDATE profiles SET role='SUPER_ADMIN' WHERE id='UUID';` |

---

## 8. ตัวอย่าง supabaseClient.ts

```ts
// lib/supabaseClient.ts
import { createClient } from '@supabase/supabase-js';

export const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
);
```

---

## 9. ตัวอย่างการเรียก ensure_profile ใน Client

```ts
const { error } = await supabase.rpc('ensure_profile');
if (error) console.error('ensure_profile error', error.message);
```

เรียกหลัง login/signup สำเร็จ (หรือ `onAuthStateChange`)

---

## 10. สคริปต์ Soft Reset (ย่อ)

```sql
BEGIN;
DO $$ BEGIN
 IF NOT EXISTS (SELECT 1 FROM pg_type WHERE typname='user_role') THEN
  CREATE TYPE user_role AS ENUM ('USER','ADMIN','SUPER_ADMIN');
 END IF;
END $$;

DROP FUNCTION IF EXISTS ensure_profile();
DROP FUNCTION IF EXISTS app_user_role();
DROP FUNCTION IF EXISTS guard_profile_role();
DROP TRIGGER IF EXISTS guard_profile_role ON profiles;

CREATE TABLE IF NOT EXISTS profiles(
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  role user_role NOT NULL DEFAULT 'USER',
  display_name TEXT,
  created_at timestamptz DEFAULT now()
);

ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;

CREATE OR REPLACE FUNCTION app_user_role()
RETURNS user_role
LANGUAGE sql STABLE AS $$
  SELECT role FROM profiles WHERE id=auth.uid();
$$;

CREATE OR REPLACE FUNCTION ensure_profile()
RETURNS void
LANGUAGE plpgsql SECURITY DEFINER SET search_path=public AS $$
DECLARE _uid uuid; em text;
BEGIN
  SELECT auth.uid() INTO _uid;
  IF _uid IS NULL THEN RETURN; END IF;
  SELECT email INTO em FROM auth.users WHERE id=_uid;
  IF em IS NULL THEN RETURN; END IF;
  INSERT INTO profiles(id,role,display_name) VALUES(_uid,'USER',em)
  ON CONFLICT(id) DO NOTHING;
END; $$;

CREATE OR REPLACE FUNCTION guard_profile_role()
RETURNS trigger
LANGUAGE plpgsql SECURITY DEFINER SET search_path=public AS $$
BEGIN
  IF NEW.role IS DISTINCT FROM OLD.role
     AND (SELECT app_user_role()) <> 'SUPER_ADMIN' THEN
    RAISE EXCEPTION 'Only SUPER_ADMIN can change role';
  END IF;
  RETURN NEW;
END; $$;

CREATE TRIGGER guard_profile_role
BEFORE UPDATE ON profiles
FOR EACH ROW EXECUTE FUNCTION guard_profile_role();

DROP POLICY IF EXISTS select_profiles_self ON profiles;
DROP POLICY IF EXISTS update_profiles_self ON profiles;

CREATE POLICY select_profiles_self ON profiles
  FOR SELECT USING (auth.uid()=id OR app_user_role() IN ('ADMIN','SUPER_ADMIN'));

CREATE POLICY update_profiles_self ON profiles
  FOR UPDATE USING (auth.uid()=id OR app_user_role()='SUPER_ADMIN')
  WITH CHECK (auth.uid()=id OR app_user_role()='SUPER_ADMIN');

INSERT INTO profiles(id,role,display_name)
SELECT u.id,'USER',u.email
FROM auth.users u
LEFT JOIN profiles p ON p.id=u.id
WHERE p.id IS NULL;

COMMIT;
```

---

## 11. Roadmap (สั้น)
- [ ] Image upload (Storage) + property_images table
- [ ] Tag management UI
- [ ] Property detail page
- [ ] Audit log viewer (SUPER_ADMIN)
- [ ] Pagination / Sorting
- [ ] Bulk import (raw_ingest)

---

## 12. License
เลือก: MIT
