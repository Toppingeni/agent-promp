## โครงสร้าง Auth/Login ในโปรเจคนี้

- Frontend SPA (React, Vite)
  - `client/auth/tokens.ts`
  - `client/auth/context.tsx`
  - `client/auth/api.ts`
  - `client/auth/LoginPage.tsx`
  - `client/auth/TokenLogin.tsx`
  - `client/auth/PrivateRoute.tsx`
  - `client/auth/index.ts`
- API Client ทั่วไปที่ใช้ JWT เดียวกัน
  - `client/tanstackQuery/api.ts`
  - `client/lib/fetchJson.ts`

---

## ตัวแปรสภาพแวดล้อม (env) ที่เกี่ยวข้อง

Frontend (Vite):

- `VITE_API_AUTH_URL`
  - base URL สำหรับ Authentication Server
  - ใช้ใน `client/auth/api.ts` เป็น `baseURL` ของ axios instance
- `VITE_BYPASS_AUTH`
  - `'true' | 'false'`
  - ถ้า `true`:
    - `LoginPage` จะ redirect ออกจากหน้า `/login` ทันที (Dev bypass)
    - `PrivateRoute` จะไม่ตรวจ token/role เลย
    - `AuthProvider` จะสร้าง mock user และ token ปลอม
- `VITE_API_URL`
  - base URL สำหรับ main backend API (เช่น `/api/repairs/...`)
  - ใช้ใน `client/tanstackQuery/api.ts` และ `client/lib/fetchJson.ts`
- `VITE_API_TIMEOUT_MS` (ไม่บังคับ)
  - ใช้กำหนด timeout ของ axios client ใน `client/tanstackQuery/api.ts`

Backend (Node/Express):

- `ORACLE_CLIENT_PATH`
  - ใช้ใน `server/index.ts` และ `server/migrations/run_migration.ts` สำหรับ init Oracle Client
- ตัวแปรอื่นของ DB/Auth อยู่ฝั่ง server/auth จริง (นอก scope ไฟล์ frontend login)

---

### ตัวอย่างไฟล์ `.env` (สำหรับส่วนที่เกี่ยวกับ Login/Auth)

```env
# Frontend (Vite) - Auth/Login
VITE_API_AUTH_URL=http://localhost:5000
VITE_API_URL=http://localhost:3001/api
VITE_API_TIMEOUT_MS=300000
VITE_BYPASS_AUTH=false

# Backend (Node/Express) - Database/Auth backend
ORACLE_CLIENT_PATH=/path/to/oracle/instantclient
```

---

## Token & User Provider (`client/auth/tokens.ts`)

ใช้ `jwt-decode` ในการ decode JWT และจัดการ token ใน `localStorage` / `sessionStorage`

```ts
import { jwtDecode } from 'jwt-decode';

const TOKEN_KEY = 'serviceToken';
const STORAGE_HINT_KEY = 'serviceTokenStorage'; // 'local' | 'session'
type StorageKind = 'local' | 'session';

/**
 * ข้อมูลผู้ใช้ที่ decode จาก JWT token
 */
export type DecodedUser = {
  UserName: string;
  UserType: string;
  UnitId: string;
  ORG: string;
  nameid: string;
  TrackingStatus: string;
  exp?: number;
};

/**
 * ฟังก์ชันสำหรับเลือก storage (localStorage หรือ sessionStorage)
 */
function getStorage(kind: StorageKind) {
  return kind === 'local' ? localStorage : sessionStorage;
}

/**
 * บันทึก token ลง storage
 * @param token JWT token
 * @param remember true = localStorage, false = sessionStorage
 */
export function saveToken(token: string, remember = true) {
  const target: StorageKind = remember ? 'local' : 'session';
  const other: StorageKind = remember ? 'session' : 'local';

  // บันทึกใน storage ที่เลือก และลบออกจาก storage อื่น
  getStorage(target).setItem(TOKEN_KEY, token);
  getStorage(other).removeItem(TOKEN_KEY);

  // บันทึก hint ว่าใช้ storage ไหน
  localStorage.setItem(STORAGE_HINT_KEY, target);
}

/**
 * ดึง token จาก storage
 * @returns JWT token หรือ null
 */
export function getToken() {
  const hint = (localStorage.getItem(STORAGE_HINT_KEY) as StorageKind | null) ?? null;
  const first = hint ? getStorage(hint).getItem(TOKEN_KEY) : null;

  return first || localStorage.getItem(TOKEN_KEY) || sessionStorage.getItem(TOKEN_KEY);
}

/**
 * ลบ token ออกจาก storage ทั้งหมด
 */
export function clearToken() {
  localStorage.removeItem(TOKEN_KEY);
  sessionStorage.removeItem(TOKEN_KEY);
  localStorage.removeItem(STORAGE_HINT_KEY);
}

/**
 * Decode JWT token เป็นข้อมูลผู้ใช้
 * @param token JWT token
 * @returns ข้อมูลผู้ใช้หรือ null ถ้า decode ไม่ได้
 */
export function decodeUser(token: string): DecodedUser | null {
  try {
    return jwtDecode<DecodedUser>(token);
  } catch {
    return null;
  }
}

/**
 * ตรวจสอบว่า token หมดอายุหรือไม่
 * @param decoded ข้อมูลผู้ใช้ที่ decode แล้ว
 * @returns true ถ้าหมดอายุ
 */
export function isExpired(decoded: DecodedUser | null): boolean {
  if (!decoded?.exp) return false;
  return Date.now() >= decoded.exp * 1000;
}
```

---

## Auth Context Provider (`client/auth/context.tsx`)

จัดการ state ของผู้ใช้ (DecodedUser) และ token ระดับแอป ทั้งยัง verify token กับ Auth API และรองรับ Dev bypass

```tsx
import React, { createContext, useContext, useEffect, useMemo, useState } from 'react';
import { decodeUser, DecodedUser, getToken, isExpired, saveToken, clearToken } from './tokens';
import { verifyToken } from './api';

/**
 * Type สำหรับ AuthContext
 */
type AuthContextType = {
  user: DecodedUser | null;
  token: string | null;
  isLoading: boolean;
  loginWithToken: (token: string, opts?: { remember?: boolean }) => void;
  logout: () => void;
};

const AuthContext = createContext<AuthContextType | null>(null);

/**
 * AuthProvider component สำหรับจัดการ authentication state
 */
export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [token, setToken] = useState<string | null>(null);
  const [user, setUser] = useState<DecodedUser | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  // ตรวจสอบ token ที่มีอยู่เมื่อ component mount
  useEffect(() => {
    // Dev bypass: สร้าง mock user สำหรับการพัฒนา
    if (import.meta.env.VITE_BYPASS_AUTH === 'true' || import.meta.env.VITE_BYPASS_AUTH === true) {
      const mockUser: DecodedUser = {
        nameid: 'sysadm',
        UserName: 'System Administrator',
        UnitId: 'B00',
        ORG: 'OPP',
        UserType: 'Admin',
        TrackingStatus: 'T',
      };
      setToken(process.env.VITE_BYPASS_AUTH_TOKEN || 'dev-token');
      setUser(mockUser);
      setIsLoading(false);
      return;
    }

    const existingToken = getToken();
    if (!existingToken) {
      setIsLoading(false);
      return;
    }

    const decodedUser = decodeUser(existingToken);
    if (isExpired(decodedUser)) {
      // Token หมดอายุ ลบออก
      clearToken();
      setIsLoading(false);
      return;
    }

    // Verify token กับ server
    verifyToken(existingToken)
      .then((isValid) => {
        if (isValid) {
          // Token ถูกต้อง set user state
          setToken(existingToken);
          setUser(decodedUser);
        } else {
          // Token ไม่ถูกต้อง ลบออก
          clearToken();
        }
        setIsLoading(false);
      })
      .catch(() => {
        // Error ในการ verify ลบ token ออก
        clearToken();
        setIsLoading(false);
      });
  }, []);

  /**
   * Login ด้วย token
   * @param newToken JWT token
   * @param opts options สำหรับการบันทึก token
   */
  const loginWithToken = (newToken: string, opts?: { remember?: boolean }) => {
    const decodedUser = decodeUser(newToken);

    if (!decodedUser || isExpired(decodedUser)) {
      throw new Error('Invalid or expired token');
    }

    saveToken(newToken, opts?.remember ?? true);
    setToken(newToken);
    setUser(decodedUser);
  };

  /**
   * Logout และลบ token
   */
  const logout = () => {
    clearToken();
    setToken(null);
    setUser(null);
  };

  const value = useMemo(() => ({ user, token, isLoading, loginWithToken, logout }), [user, token, isLoading]);

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

/**
 * Hook สำหรับใช้งาน AuthContext
 * @returns AuthContextType
 */
export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
}
```

---

## Auth API Client (`client/auth/api.ts`)

จัดการการเรียก API ที่เกี่ยวกับ Auth ทั้ง login และ verify token โดยใช้ `axios` และแนบ JWT ผ่าน Authorization header

```ts
import axios from 'axios';
import { clearToken, getToken } from './tokens';

/**
 * HTTP client สำหรับ authentication API
 * ใช้ VITE_API_AUTH_URL เป็น base URL
 */
export const api = axios.create({
  baseURL: import.meta.env.VITE_API_AUTH_URL,
});

/**
 * Request interceptor: แนบ Authorization header ถ้ามี token
 */
api.interceptors.request.use((cfg) => {
  const token = getToken();
  if (token) {
    cfg.headers.Authorization = `Bearer ${token}`;
  }
  return cfg;
});

/**
 * Response interceptor: จัดการ 401 Unauthorized
 * เมื่อได้รับ 401 จะลบ token และ redirect ไป login page
 */
api.interceptors.response.use(
  (response) => response,
  (error) => {
    const status = error?.response?.status;

    if (status === 401 && typeof window !== 'undefined') {
      // ลบ token ที่หมดอายุ
      clearToken();

      // สร้าง redirect URL พร้อม current path
      const currentPath = `${window.location.pathname}${window.location.search || ''}`;
      const redirectTo = encodeURIComponent(currentPath || '/');

      // Redirect ไป login page
      window.location.replace(`/login?redirectTo=${redirectTo}`);
    }

    return Promise.reject(error);
  },
);

/**
 * ฟังก์ชันสำหรับ verify token กับ server
 * @param token JWT token ที่ต้องการ verify
 * @returns Promise<boolean> true ถ้า token ถูกต้อง, false ถ้าไม่ถูกต้อง
 */
export async function verifyToken(token: string): Promise<boolean> {
  try {
    const response = await api.post(
      '/api/user/verify',
      { Token: token },
      {
        headers: {
          Authorization: `Bearer ${token}`,
        },
      },
    );

    return response.status === 200 && response.data === 'OK';
  } catch (error) {
    // ถ้าได้ 401 หรือ error อื่นๆ แสดงว่า token ไม่ถูกต้อง
    return false;
  }
}
```

---

## Login Page (Username/Password Provider) (`client/auth/LoginPage.tsx`)

Provider หลักคือฟอร์ม username/password ที่ยิงไปที่ `/api/user/login` บน Auth Server แล้วรับ JWT token กลับมา

ใช้:

- `react-router-dom` (`useNavigate`, `useSearchParams`)
- `zod` สำหรับ schema validation
- `react-hook-form` + `@hookform/resolvers/zod`
- `axios` (ผ่าน `api` จาก `client/auth/api.ts`)
- Shadcn UI components (`Card`, `Input`, `Button`, `Checkbox`, `Alert`, ฯลฯ)

```tsx
import React from 'react';
import { ErrorResponse, useNavigate, useSearchParams } from 'react-router-dom';
import { z } from 'zod';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { api } from './api';
import { useAuth } from './context';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { Button } from '@/components/ui/button';
import { Checkbox } from '@/components/ui/checkbox';
import { Alert, AlertDescription } from '@/components/ui/alert';
import { AxiosError } from 'axios';

/**
 * Schema สำหรับ validation ฟอร์ม login
 */
const loginSchema = z.object({
  userId: z.string().min(1, 'กรุณากรอกชื่อผู้ใช้'),
  password: z.string().min(1, 'กรุณากรอกรหัสผ่าน'),
  remember: z.boolean().default(true),
  enableTracking: z.boolean().default(false),
});

type FormValues = z.infer<typeof loginSchema>;

/**
 * Props สำหรับ LoginPage component
 */
type LoginPageProps = {
  org?: string;
  trackingStatus?: string;
  title?: string;
};

/**
 * หน้า Login สำหรับการเข้าสู่ระบบ
 */
export default function LoginPage({ org = 'OPP', trackingStatus = 'F', title = 'เข้าสู่ระบบ' }: LoginPageProps) {
  const [searchParams] = useSearchParams();
  const navigate = useNavigate();
  const { loginWithToken, user } = useAuth();
  const [errorMessage, setErrorMessage] = React.useState<string | null>(null);
  const [isSubmitting, setIsSubmitting] = React.useState(false);
  const [shouldRedirect, setShouldRedirect] = React.useState(false);

  const form = useForm<FormValues>({
    resolver: zodResolver(loginSchema),
    defaultValues: {
      userId: '',
      password: '',
      remember: true,
      enableTracking: trackingStatus === 'T',
    },
    mode: 'onSubmit',
  });

  /**
   * ฟังก์ชันสำหรับ sanitize redirect URL
   */
  const sanitizeRedirectUrl = (url: string | null | undefined): string => {
    if (!url) return '/';
    if (!url.startsWith('/')) return '/';
    if (url.startsWith('//')) return '/';
    if (url.includes('://')) return '/';
    return url;
  };

  // Dev bypass: redirect ออกจากหน้า login ถ้าเปิด bypass mode
  React.useEffect(() => {
    if (import.meta.env.VITE_BYPASS_AUTH === 'true') {
      const redirectTo = sanitizeRedirectUrl(searchParams.get('redirectTo'));
      navigate(redirectTo, { replace: true });
    }
  }, [navigate, searchParams]);

  // Redirect หลังจาก login สำเร็จ
  React.useEffect(() => {
    if (shouldRedirect && user) {
      const redirectTo = sanitizeRedirectUrl(searchParams.get('redirectTo'));
      navigate(redirectTo, { replace: true });
    }
  }, [shouldRedirect, user, navigate, searchParams]);

  /**
   * ฟังก์ชันสำหรับ submit ฟอร์ม login
   */
  const onSubmit = async (values: FormValues) => {
    setErrorMessage(null);
    setIsSubmitting(true);

    try {
      const response = await api.post('/api/user/login', {
        userId: values.userId,
        password: values.password,
        org: org,
        trackingstatus: values.enableTracking ? 'T' : 'F',
      });

      const token = response.data?.token as string | undefined;
      if (!token) {
        throw new Error('ไม่ได้รับ token จากเซิร์ฟเวอร์');
      }

      // Login ด้วย token ที่ได้รับ
      loginWithToken(token, { remember: values.remember });

      // ตั้งค่าให้ redirect หลังจาก user state update
      setShouldRedirect(true);
    } catch (error: unknown) {
      let message = 'เข้าสู่ระบบไม่สำเร็จ';

      if (typeof error === 'object' && error && 'response' in error) {
        // Axios error
        // eslint-disable-next-line @typescript-eslint/no-explicit-any
        const axiosError = error as any;
        message = axiosError.response?.data?.message ?? message;
      } else if (error instanceof Error) {
        message = error.message;
      }

      setErrorMessage(message);
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <div className="min-h-screen bg-gradient-to-br from-slate-50 to-slate-100 dark:from-slate-900 dark:to-slate-950 grid place-items-center px-4">
      <Card className="w-full max-w-md">
        <CardHeader className="items-center text-center space-y-3">
          <div className="w-10 h-10 rounded-xl flex items-center justify-center shadow-lg ring-2 ring-white/50 ring-offset-2 overflow-hidden bg-white">
            <img src="/favicon.ico" alt="KIM PAI THAI O.P.P. Logo" className="w-full h-full object-contain" />
          </div>
          <CardTitle className="text-2xl font-semibold tracking-tight">{title}</CardTitle>
          <CardDescription>กรอกข้อมูลเพื่อเข้าสู่ระบบ</CardDescription>
        </CardHeader>
        <CardContent>
          {errorMessage && (
            <Alert className="mb-4" variant="destructive">
              <AlertDescription>{errorMessage}</AlertDescription>
            </Alert>
          )}

          <form className="grid gap-4" onSubmit={form.handleSubmit(onSubmit)}>
            <div className="grid gap-2">
              <Label htmlFor="userId">ชื่อผู้ใช้</Label>
              <Input id="userId" autoComplete="username" placeholder="ชื่อผู้ใช้" {...form.register('userId')} />
              {form.formState.errors.userId && <p className="text-sm text-destructive">{form.formState.errors.userId.message}</p>}
            </div>

            <div className="grid gap-2">
              <Label htmlFor="password">รหัสผ่าน</Label>
              <Input id="password" type="password" autoComplete="current-password" placeholder="••••••••" {...form.register('password')} />
              {form.formState.errors.password && <p className="text-sm text-destructive">{form.formState.errors.password.message}</p>}
            </div>

            <div className="flex items-center gap-2">
              <Checkbox id="remember" checked={form.watch('remember')} onCheckedChange={(checked) => form.setValue('remember', Boolean(checked))} />
              <Label htmlFor="remember">จำการเข้าสู่ระบบ</Label>
            </div>

            <div className="flex items-center gap-2">
              <Checkbox id="enableTracking" checked={form.watch('enableTracking')} onCheckedChange={(checked) => form.setValue('enableTracking', Boolean(checked))} />
              <Label htmlFor="enableTracking">Tracking Status</Label>
            </div>

            <Button type="submit" className="w-full" disabled={isSubmitting}>
              {isSubmitting ? 'กำลังเข้าสู่ระบบ...' : 'เข้าสู่ระบบ'}
            </Button>
          </form>
        </CardContent>
      </Card>
    </div>
  );
}
```

---

## Token Provider จาก External Service (`client/auth/TokenLogin.tsx`)

Provider ที่สองคือการรับ token จาก URL (เช่น SSO หรือระบบอื่นส่ง redirect มา) แล้วใช้ `loginWithToken` เหมือนกับหน้า Login ปกติ

```tsx
import React, { useEffect } from 'react';
import { useNavigate, useSearchParams } from 'react-router-dom';
import { useAuth } from './context';

/**
 * Component สำหรับ login ด้วย token ผ่าน URL
 * ใช้สำหรับกรณีที่ได้รับ token จาก external service
 * URL format: /token-login?token=xxx&redirectTo=/path
 */
export default function TokenLogin() {
  const [searchParams] = useSearchParams();
  const { loginWithToken } = useAuth();
  const navigate = useNavigate();

  /**
   * ฟังก์ชันสำหรับ sanitize redirect URL
   */
  const sanitizeRedirectUrl = (url: string | null | undefined): string => {
    if (!url) return '/';
    if (!url.startsWith('/')) return '/';
    if (url.startsWith('//')) return '/';
    if (url.includes('://')) return '/';
    return url;
  };

  useEffect(() => {
    const token = searchParams.get('token');
    const redirectTo = sanitizeRedirectUrl(searchParams.get('redirectTo'));

    // ถ้าไม่มี token redirect ไป login page
    if (!token) {
      navigate('/login', { replace: true });
      return;
    }

    try {
      // พยายาม login ด้วย token
      loginWithToken(token, { remember: true });

      // สำเร็จแล้ว redirect ไปหน้าที่ต้องการ
      navigate(redirectTo, { replace: true });
    } catch (error) {
      // Token ไม่ถูกต้อง redirect ไป login page
      console.error('Token login failed:', error);
      navigate('/login', { replace: true });
    }
  }, [searchParams, loginWithToken, navigate]);

  return (
    <div className="min-h-screen bg-gradient-to-br from-slate-50 to-slate-100 dark:from-slate-900 dark:to-slate-950 grid place-items-center px-4">
      <div className="text-center">
        <div className="animate-spin rounded-full h-8 w-8 border-b-2 border-primary mx-auto mb-4"></div>
        <p className="text-muted-foreground">กำลังเข้าสู่ระบบ...</p>
      </div>
    </div>
  );
}
```

---

## Route Guard / Protected Routes (`client/auth/PrivateRoute.tsx`)

ใช้ร่วมกับ `react-router-dom` และ `UserRoleContext` เพื่อบังคับ login และตรวจ role ก่อนเข้าแต่ละหน้า

```tsx
import React from 'react';
import { Navigate, useLocation } from 'react-router-dom';
import { useAuth } from './context';
import { useUserRole } from '../contexts/UserRoleContext';
import { UserRole } from '../types/role';

/**
 * Component สำหรับป้องกันหน้าที่ต้องการการ login
 * ถ้าผู้ใช้ยังไม่ได้ login จะ redirect ไป login page
 * ถ้าผู้ใช้ไม่มีสิทธิ์เข้าถึงหน้านั้น (ตาม roles) จะ redirect ไปหน้าแรก
 */
export default function PrivateRoute({ children, roles }: { children: JSX.Element; roles?: string[] }) {
  const { user, isLoading: authLoading } = useAuth();
  const { hasAnyRole, isLoading: roleLoading } = useUserRole();
  const location = useLocation();

  // แสดง loading ระหว่างรอ token verification หรือ role checking
  if (authLoading || roleLoading) {
    return <div>Loading...</div>;
  }

  // Dev bypass: ข้าม authentication ในโหมดพัฒนา
  if (import.meta.env.VITE_BYPASS_AUTH === 'true') {
    return children;
  }

  // ถ้าไม่มี user (ยังไม่ได้ login) redirect ไป login page
  if (!user) {
    const redirectTo = encodeURIComponent(location.pathname + (location.search || ''));
    return <Navigate to={`/login?redirectTo=${redirectTo}`} replace />;
  }

  // ถ้ามีการระบุ roles และ user ไม่มีสิทธิ์
  if (roles && roles.length > 0) {
    // cast roles เป็น UserRole[] เพื่อใช้กับ hasAnyRole
    const hasPermission = hasAnyRole(roles as UserRole[]);

    if (!hasPermission) {
      console.warn(`User ${user.UserName} does not have permission to access ${location.pathname}. Required roles: ${roles.join(', ')}`);
      // Redirect ไปหน้าแรก (ซึ่งจะ handle redirect ต่อไปตาม role ของ user โดย RoleBasedRedirect)
      return <Navigate to="/" replace />;
    }
  }

  return children;
}
```

---

## Main API Client ที่ใช้ JWT เดียวกัน (`client/tanstackQuery/api.ts`)

Client นี้ใช้ JWT จาก `getToken()` เหมือน auth/api และใช้ `VITE_API_URL` เป็น base URL

```ts
import axios, { AxiosResponse } from 'axios';
import { getToken, clearToken } from '@/auth/tokens';
import {
  RepairRequest,
  RepairOrder,
  RepairRequestFilters,
  RepairProcessFilters,
  CreateRepairRequestDto,
  CreateRepairOrderDto,
  UpdateRepairRequestDto,
  SaveRepairActionDto,
  DashboardStats,
  RepairDocument,
  SaveCloseRepairProcessDto,
  CancelRepairRequestDto,
  RepairLogEntry,
} from '../../shared/types/repairs';

/**
 * Shared API Client สำหรับ Kim Pai Repair Flow
 * ใช้งานร่วมกันระหว่าง client และ server
 */

// Base API configuration
const API_BASE_URL = typeof window !== 'undefined' ? import.meta.env?.VITE_API_URL || 'http://localhost:3001/api' : process.env.VITE_API_URL || 'http://localhost:3001/api';
// เพิ่ม timeout เป็น 5 นาที และรองรับการตั้งค่าผ่าน env: VITE_API_TIMEOUT_MS
const API_TIMEOUT_MS = typeof window !== 'undefined' ? Number(import.meta.env?.VITE_API_TIMEOUT_MS ?? 300000) : Number(process.env.VITE_API_TIMEOUT_MS ?? 300000);

const apiClient = axios.create({
  baseURL: API_BASE_URL,
  timeout: API_TIMEOUT_MS,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Request interceptor สำหรับ authentication
apiClient.interceptors.request.use(
  (config) => {
    const token = typeof window !== 'undefined' ? getToken() : null;
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => {
    return Promise.reject(error);
  },
);

// Response interceptor สำหรับ error handling
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401 && typeof window !== 'undefined') {
      // Token invalid or expired – clear stored credentials and redirect to login
      clearToken();

      const currentPath = `${window.location.pathname}${window.location.search || ''}`;
      const redirectTo = encodeURIComponent(currentPath || '/');
      window.location.replace(`/login?redirectTo=${redirectTo}`);
    }
    return Promise.reject(error);
  },
);
```

---

## Fetch-based API Helper ที่ใช้ JWT เดียวกัน (`client/lib/fetchJson.ts`)

อีก provider สำหรับเรียก API ด้วย `fetch` (ไม่ใช่ axios) แต่ใช้ JWT และ `VITE_API_URL` แบบเดียวกัน

```ts
import { getToken } from '@/auth/tokens';
import type { ApiError as IApiError } from '../../shared/types';
import { HTTP_STATUS, ERROR_CODES } from '../../shared/constants';
import { fetchWithMiddleware, fetchWithRetry, fetchWithTimeout } from './apiMiddleware';

// Base API URL from environment
const API_BASE_URL = import.meta.env.VITE_API_URL || 'http://localhost:3000';

// API Error class for better error handling
export class ApiError extends Error implements IApiError {
  constructor(
    message: string,
    public status?: number,
    public code?: string,
    public details?: Record<string, unknown>,
  ) {
    super(message);
    this.name = 'ApiError';
  }
}

// Request options interface
interface FetchOptions extends RequestInit {
  params?: Record<string, string | number | boolean>;
  timeout?: number;
}

// Response wrapper
interface ApiResponse<T = unknown> {
  data: T;
  status: number;
  statusText: string;
}

// Build URL with query parameters
function buildUrl(endpoint: string, params?: Record<string, string | number | boolean>): string {
  const url = new URL(endpoint, API_BASE_URL);

  if (params) {
    Object.entries(params).forEach(([key, value]) => {
      if (value !== undefined && value !== null) {
        url.searchParams.append(key, String(value));
      }
    });
  }

  return url.toString();
}

// Main fetchJson function
export async function fetchJson<T = unknown>(endpoint: string, options: FetchOptions = {}): Promise<ApiResponse<T>> {
  const { params, timeout = 30000, headers = {}, ...fetchOptions } = options;

  // Build URL with parameters
  const url = buildUrl(endpoint, params);

  // Get authentication token
  const token = getToken();

  // Prepare headers
  const requestHeaders: Record<string, string> = {
    'Content-Type': 'application/json',
  };

  // Add custom headers
  if (headers) {
    Object.entries(headers).forEach(([key, value]) => {
      if (typeof value === 'string') {
        requestHeaders[key] = value;
      }
    });
  }

  // Add authorization header if token exists
  if (token) {
    requestHeaders.Authorization = `Bearer ${token}`;
  }

  // Create abort controller for timeout
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), timeout);

  try {
    const response = await fetchWithMiddleware(url, {
      ...fetchOptions,
      headers: requestHeaders,
      signal: controller.signal,
    });

    clearTimeout(timeoutId);

    // Parse response
    let data: T;
    const contentType = response.headers.get('content-type');

    if (contentType && contentType.includes('application/json')) {
      data = await response.json();
    } else {
      data = (await response.text()) as unknown as T;
    }

    // Handle HTTP errors
    if (!response.ok) {
      let errorMessage = `HTTP ${response.status}: ${response.statusText}`;
      let errorCode = 'HTTP_ERROR';
      let errorDetails: Record<string, unknown> = {};

      // Map common HTTP status codes to error codes
      switch (response.status) {
        case HTTP_STATUS.UNAUTHORIZED:
          errorCode = ERROR_CODES.UNAUTHORIZED;
          errorMessage = 'ไม่ได้รับอนุญาตให้เข้าถึง';
          break;
        // กรณีอื่น ๆ ถูกจัดการต่อจากนี้ในไฟล์จริง
      }

      throw new ApiError(errorMessage, response.status, errorCode, errorDetails);
    }

    return {
      data,
      status: response.status,
      statusText: response.statusText,
    };
  } finally {
    clearTimeout(timeoutId);
  }
}
```

---

## สรุป Flow การ Login

- หน้า `/login`
  - รับ `userId`, `password`, `remember`, `enableTracking`
  - ส่งไป `POST {VITE_API_AUTH_URL}/api/user/login`
  - รับ `token` จาก response
  - เรียก `loginWithToken(token, { remember })` เพื่อเก็บ token และ decode user
  - Redirect ไป path ใน query param `redirectTo` (sanitize แล้ว)
- หน้า `/token-login`
  - รับ `token` และ `redirectTo` จาก query string
  - เรียก `loginWithToken(token, { remember: true })`
  - Redirect ไป `redirectTo`
- ทุก request ไป main API (`VITE_API_URL`)
  - แนบ `Authorization: Bearer <token>` ผ่าน axios/fetch interceptors
  - ถ้าได้ 401 -> `clearToken()` และ redirect กลับ `/login?redirectTo=...`
