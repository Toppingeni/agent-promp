# Logger System Implementation Guide

เอกสารฉบับนี้อธิบายขั้นตอนการสร้างระบบ Logger สำหรับ Backend (Node.js/TypeScript) ที่มีความสามารถ:

1. **Context-Aware**: ดึงข้อมูล User/TrackingStatus จาก JWT อัตโนมัติโดยไม่ต้องส่ง parameter ไปทุก function
2. **WebSocket Integration**: ส่ง Log ไปยัง Monitoring Server แบบ Fire & Forget
3. **Structured Logging**: จัด format log ให้เป็นระบบ (JSON/Text)
4. **Environment Config**: ปรับระดับ Log Level ได้ผ่าน .env
5. **Specialized Logging**: มีฟังก์ชันเฉพาะสำหรับ Request/Response และ SQL
6. **Production Safe**: ปิด Console Log ใน Production เพื่อประสิทธิภาพและความสะอาด

---

## 1. สร้าง Context Store (`utils/context.ts`)

ใช้ `AsyncLocalStorage` จาก Node.js เพื่อเก็บข้อมูล Request Scope (เช่น userId) ให้เรียกใช้ได้จากทุกที่ใน code โดยไม่ต้องส่งผ่าน function arguments

```typescript
import { AsyncLocalStorage } from 'async_hooks';

export interface RequestContext {
  userId?: string;
  userName?: string;
  requestId?: string;
  trackingStatus?: string; // ใช้สำหรับ logic พิเศษ เช่น ถ้าเป็น 'F' ไม่ต้องส่ง WebSocket
}

// สร้าง Instance ของ AsyncLocalStorage
export const context = new AsyncLocalStorage<RequestContext>();
```

---

## 2. สร้าง Middleware เพื่อแกะ JWT และเริ่ม Context (`middlewares/contextMiddleware.ts`)

Middleware นี้จะทำงานทุก Request เพื่อ:

1. อ่าน Header `Authorization`
2. แกะ JWT (Decode) เพื่อเอา User Info
3. เรียก `context.run()` เพื่อเริ่ม Scope ของ Request นั้น

```typescript
import { Request, Response, NextFunction } from 'express';
import { context } from '../utils/context';

export const contextMiddleware = (req: Request, res: Response, next: NextFunction): void => {
  let userId = 'anonymous';
  let userName = 'Anonymous';
  let trackingStatus: string | undefined;

  try {
    const authHeader = req.headers['authorization'];
    if (authHeader && authHeader.startsWith('Bearer ')) {
      const token = authHeader.substring(7);
      // Decode JWT แบบง่าย (ไม่ได้ Verify Signature ที่นี่เพื่อความเร็ว หรือถ้ามี Middleware Auth แยกก็ใช้ค่าจากตรงนั้นได้)
      const parts = token.split('.');
      if (parts.length === 3) {
        const payload = JSON.parse(Buffer.from(parts[1], 'base64').toString());

        // Map field ตาม Structure ของ JWT ที่ใช้
        userId = payload.nameid || payload.userId || payload.sub || 'unknown_user';
        userName = payload.UserName || payload.username || 'Unknown User';
        trackingStatus = payload.TrackingStatus;
      }
    }
  } catch (e) {
    // กรณี Decode ไม่ผ่าน ให้ใช้ค่า Default
  }

  const store = {
    userId,
    userName,
    requestId: (req.headers['x-request-id'] as string) || `req-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`,
    trackingStatus,
  };

  // Run next() ภายใน context.run เพื่อให้ store นี้ใช้ได้ตลอด flow ของ request นี้
  context.run(store, () => {
    next();
  });
};
```

---

## 3. สร้าง Logger Class (`utils/logger.ts`)

เป็น Class หลักสำหรับจัดการ Log ทั้งหมด โดยใช้ Singleton Pattern และรวม Methods สำหรับ SQL และ HTTP Request/Response ไว้ด้วย

```typescript
import { context } from './context';

/* eslint-disable @typescript-eslint/no-explicit-any */

export enum LogLevel {
  ERROR = 0,
  WARN = 1,
  INFO = 2,
  DEBUG = 3,
}

export class Logger {
  private static instance: Logger;
  private enabled: boolean;
  private logLevel: LogLevel;
  private wsLogServerUrl: string | undefined;

  private constructor() {
    // Config จาก Environment Variables
    this.enabled = process.env.ENABLE_LOGGING !== 'false';
    this.logLevel = this.parseLogLevel(process.env.LOG_LEVEL || 'INFO');
    this.wsLogServerUrl = process.env.WS_LOG_SERVER_URL;
  }

  public static getInstance(): Logger {
    if (!Logger.instance) {
      Logger.instance = new Logger();
    }
    return Logger.instance;
  }

  private parseLogLevel(level: string): LogLevel {
    switch (level.toUpperCase()) {
      case 'ERROR':
        return LogLevel.ERROR;
      case 'WARN':
        return LogLevel.WARN;
      case 'INFO':
        return LogLevel.INFO;
      case 'DEBUG':
        return LogLevel.DEBUG;
      default:
        return LogLevel.INFO;
    }
  }

  private shouldLog(level: LogLevel): boolean {
    return this.enabled && level <= this.logLevel;
  }

  private formatMessage(level: string, message: string, meta?: any): string {
    const timestamp = new Date().toISOString();
    const metaStr = meta ? ` | ${JSON.stringify(meta)}` : '';
    return `[${timestamp}] [${level}] ${message}${metaStr}`;
  }

  /**
   * ส่ง Log ไป WebSocket (Fire & Forget)
   * ใช้ context.getStore() เพื่อดึง User Info อัตโนมัติ
   */
  private async sendToWsServer(level: string, message: string, meta?: any) {
    if (!this.wsLogServerUrl) return;

    // ดึง Context ปัจจุบัน
    const store = context.getStore();

    // Logic ตรวจสอบ TrackingStatus (ถ้ามี)
    const trackingStatus = meta?.trackingStatus || store?.trackingStatus;
    if (trackingStatus === 'F') return;

    try {
      const logPayload = {
        timestamp: new Date().toISOString(),
        level,
        service: 'kim-pai-repair-backend',
        message,
        userId: store?.userId, // Auto-inject user id
        userName: store?.userName,
        trackingStatus,
        ...meta,
      };

      if (typeof fetch !== 'undefined') {
        fetch(this.wsLogServerUrl, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(logPayload),
        }).catch(() => {
          /* Ignore errors */
        });
      }
    } catch (e) {
      // Prevent crash
    }
  }

  // --- Standard Logging Methods ---

  public error(message: string, meta?: any): void {
    if (this.shouldLog(LogLevel.ERROR)) {
      if (process.env.NODE_ENV === 'development') console.error(this.formatMessage('ERROR', message, meta));
      this.sendToWsServer('ERROR', message, meta);
    }
  }

  public warn(message: string, meta?: any): void {
    if (this.shouldLog(LogLevel.WARN)) {
      if (process.env.NODE_ENV === 'development') console.warn(this.formatMessage('WARN', message, meta));
      this.sendToWsServer('WARN', message, meta);
    }
  }

  public info(message: string, meta?: any): void {
    if (this.shouldLog(LogLevel.INFO)) {
      if (process.env.NODE_ENV === 'development') console.log(this.formatMessage('INFO', message, meta));
      this.sendToWsServer('INFO', message, meta);
    }
  }

  public debug(message: string, meta?: any): void {
    if (this.shouldLog(LogLevel.DEBUG)) {
      if (process.env.NODE_ENV === 'development') console.log(this.formatMessage('DEBUG', message, meta));
      this.sendToWsServer('DEBUG', message, meta);
    }
  }

  // --- Specialized Logging Methods ---

  public logRequest(method: string, url: string, body?: any, headers?: any): void {
    if (this.shouldLog(LogLevel.INFO)) {
      // Extract User ID from JWT manually here if needed for meta,
      // but sendToWsServer will also get it from context.
      // This part ensures we have user info even in the console log meta.
      let userId = 'anonymous';
      let userName = 'Anonymous';
      let trackingStatus: string | undefined;

      try {
        const authHeader = headers?.['authorization'];
        if (authHeader && authHeader.startsWith('Bearer ')) {
          const token = authHeader.substring(7);
          const parts = token.split('.');
          if (parts.length === 3) {
            const payload = JSON.parse(Buffer.from(parts[1], 'base64').toString());
            userId = payload.nameid || payload.userId || payload.sub || 'unknown_user';
            userName = payload.UserName || payload.username || 'Unknown User';
            trackingStatus = payload.TrackingStatus;
          }
        }
      } catch (e) {
        /* Ignore */
      }

      const meta = {
        method,
        url,
        userId,
        userName,
        trackingStatus,
        body: body ? JSON.stringify(body) : undefined,
        userAgent: headers?.['user-agent'],
        ip: headers?.['x-forwarded-for'] || headers?.['x-real-ip'],
      };
      this.info(`HTTP Request: ${method} ${url}`, meta);
    }
  }

  public logResponse(method: string, url: string, statusCode: number, duration: number): void {
    if (this.shouldLog(LogLevel.INFO)) {
      const meta = {
        method,
        url,
        statusCode,
        duration: `${duration}ms`,
      };
      this.info(`HTTP Response: ${method} ${url} - ${statusCode}`, meta);
    }
  }

  public logSQL(sql: string, params?: any, duration?: number): void {
    // ใช้ INFO level แทน DEBUG เพื่อให้แสดงผลใน default config และส่ง WebSocket ได้ง่ายขึ้น
    if (this.shouldLog(LogLevel.INFO)) {
      const meta = {
        sql: sql.replace(/\s+/g, ' ').trim(),
        params,
        duration: duration ? `${duration}ms` : undefined,
      };
      // ใช้ info แทน debug เพื่อให้เห็น log SQL ได้ง่ายขึ้น
      this.debug('SQL Query Executed', meta);
    }
  }

  public logSQLError(sql: string, params?: any, error?: any): void {
    if (this.shouldLog(LogLevel.ERROR)) {
      const meta = {
        sql: sql.replace(/\s+/g, ' ').trim(),
        params,
        error: error?.message || error,
      };
      this.error('SQL Query Failed', meta);
    }
  }

  // --- Utility Methods ---

  public setEnabled(enabled: boolean): void {
    this.enabled = enabled;
  }

  public setLogLevel(level: LogLevel): void {
    this.logLevel = level;
  }
}

// Export Singleton
export const logger = Logger.getInstance();

// Export Convenience Functions (Helper Wrappers)
export const logRequest = (method: string, url: string, body?: any, headers?: any) => logger.logRequest(method, url, body, headers);
export const logResponse = (method: string, url: string, statusCode: number, duration: number) => logger.logResponse(method, url, statusCode, duration);
```

---

## 4. การนำไปใช้งาน (Integration)

### 4.1 ลงทะเบียน Middleware ใน `app.ts` หรือ `index.ts`

```typescript
import express from 'express';
import { contextMiddleware } from './middlewares/contextMiddleware';

const app = express();
app.use(express.json());
app.use(contextMiddleware); // <--- สำคัญ: ใส่ก่อน Routes
// ...
```

### 4.2 การเรียกใช้ Logger ทั่วไป

```typescript
import { logger } from '../utils/logger';

// Log ธรรมดา
logger.info('Starting process...');
logger.error('Something went wrong', { errorCode: 500 });
```

### 4.3 การเรียกใช้ SQL Logger (ใน Database Layer)

```typescript
async query<T>(sql: string, params: oracledb.BindParameters = {}, options?: oracledb.ExecuteOptions): Promise<T[]> {
  return await oracleConnection(this.dbName, async (connection) => {
    const startTime = Date.now();
    try {
      const result = await connection.execute<T>(sql, params, {
        ...this.options,
        ...options,
      });

      const duration = Date.now() - startTime;
      logger.logSQL(sql, params, duration);

      return result.rows ? result.rows : [];
    } catch (error: unknown) {
      logger.logSQLError(sql, params, error);
      console.error(error);
      const message = error instanceof Error ? error.message : 'Error querying Oracle database';
      throw new Error(message);
    }
  });
}
```

### 4.4 การตั้งค่า .env

```env
ENABLE_LOGGING=true
LOG_LEVEL=INFO
WS_LOG_SERVER_URL=http://localhost:8080/logs
NODE_ENV=development # Set to 'production' to disable console logs
```
