คุณเป็น Senior Developer ผู้เชี่ยวชาญด้าน template ขององค์กร: ช่วยสร้างการติดต่อกับ oracledb ตามเอกสารนี้หน่อย
* ห้ามเปลี่ยนแปลง code ใดๆยกเว้นการ import
* ยึด code ตามเอกสารเป็นหลักห้ามคิดเพิ่มใดๆ

## โครงสร้างไฟล์และ path ที่เกี่ยวข้อง

```text
server/libs/oracle/config.ts
server/libs/oracle/oracledb.ts
server/libs/oracle/index.ts
server/types/oracleType.ts
```

ด้านล่างคือโค้ด **เดิมทั้งหมด** ของไฟล์หลักใน `server/libs` และ type/ตัวอย่างการใช้งาน ที่อ้างอิงตรงจากโค้ดปัจจุบันในโปรเจกต์นี้

---

## 1. `server/libs/oracle/config.ts`

```ts
/* eslint-disable @typescript-eslint/no-explicit-any */
import fs from 'fs';
import { ITns } from '../../types/oracleType';
import { getTnsString } from '../../utils/databaseHelper';
import tnsMod from 'tns';

const tns = (tnsMod as any).default ?? (tnsMod as any);

/**
 * อ่านและแปลงไฟล์ tnsnames.ora เป็น connection string
 * สำหรับการเชื่อมต่อ Oracle Database
 */
export const getConfig = async () => {
  const base = process.env.TNS_PATH || process.cwd();
  const tnsPath = `${base}/tnsnames.ora`;
  const content = fs.readFileSync(tnsPath, 'utf-8');
  const allTns: ITns = tns(content);
  const tnsConnectString: Record<string, string> = {};

  for await (const key of Object.keys(allTns)) {
    const con_tns = allTns[key];

    if (con_tns.DESCRIPTION.ADDRESS_LIST) {
      tnsConnectString[key] = getTnsString(con_tns);
    }
  }

  return tnsConnectString;
};
```

---

## 2. `server/libs/oracle/oracledb.ts`

```ts
/* eslint-disable @typescript-eslint/no-explicit-any */
import type { Connection } from 'oracledb';
import oracledb from 'oracledb';
import { getConfig } from './config';

export type IOracleDB = ReturnType<typeof oracleDB>;

/**
 * สร้าง Oracle Database connection
 * @param mode - ชื่อ database mode ที่กำหนดใน tnsnames.ora
 * @returns Oracle connection instance
 */
async function oracleDB(mode: string) {
  const config = await getConfig();

  if (!config[mode]) throw new Error('Oracle connection string not found');

  return oracledb.getConnection({
    user: process.env.ORACLE_USER || '',
    password: process.env.ORACLE_PWD || '',
    connectString: config[mode],
  });
}

/**
 * จัดการ Oracle connection พร้อม auto-close
 * @param mode - ชื่อ database mode
 * @param callback - function ที่จะทำงานกับ connection
 * @returns ผลลัพธ์จาก callback function
 */
export async function oracleConnection(mode: string, callback: (connection: Connection) => Promise<any>) {
  const connection = await oracleDB(mode);

  try {
    return await callback(connection);
  } finally {
    if (connection) {
      try {
        await connection.close();
      } catch (err) {
        console.error(err);
      }
    }
  }
}
```

---

## 3. `server/libs/oracle/index.ts` (คลาส `Oracle`)

```ts
import oracledb from 'oracledb';
import { oracleConnection } from './oracledb';
import { CommandsSpType } from '../../types/oracleType';
import { convertSQL } from '../../utils/sqlHelper';
import { logger } from '../../utils/logger';

/**
 * Oracle Database utility class
 * ให้บริการ CRUD operations และ stored procedure calls
 */
class Oracle {
  dbName: string;
  appID: string;
  options = {
    autoCommit: false,
    outFormat: oracledb.OUT_FORMAT_OBJECT,
  };
  optionExecuteMany = {
    autoCommit: false,
    batchErrors: true,
  };

  constructor(dbName?: string, appID?: string) {
    this.dbName = dbName || process.env.ORACLE_DB_NAME!;
    this.appID = appID || process.env.APP_ID!;
  }

  /**
   * Execute SELECT query
   * @param sql - SQL query string
   * @param params - Query parameters
   * @param options - Execute options
   * @returns Query results
   */
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

  /**
   * Execute multiple SELECT queries
   * @param queries - Array of query objects
   * @returns Array of query results
   */
  async queries<T>(queries: { sql: string; params: oracledb.BindParameters }[]) {
    await oracleConnection(this.dbName, async (connection) => {
      try {
        const results = await Promise.all(
          queries.map(async (query) => {
            const startTime = Date.now();
            try {
              const result = await connection.execute<T>(query.sql, query.params, this.options);
              const duration = Date.now() - startTime;
              logger.logSQL(query.sql, query.params, duration);

              return result.rows ? result.rows : [];
            } catch (error: unknown) {
              logger.logSQLError(query.sql, query.params, error);
              throw error;
            }
          }),
        );

        return results;
      } catch (error: unknown) {
        console.error(error);
        const message = error instanceof Error ? error.message : 'Error querying Oracle database';
        throw new Error(message);
      }
    });
  }

  /**
   * Execute INSERT/UPDATE/DELETE command
   * @param sql - SQL command string
   * @param params - Command parameters
   * @returns Execute result
   */
  async command<T>(sql: string, params: oracledb.BindParameters) {
    return await oracleConnection(this.dbName, async (connection) => {
      const startTime = Date.now();
      try {
        const result = await connection.execute<T>(sql, params, this.options);

        if (result.rowsAffected && result.rowsAffected > 0) {
          await connection.commit();
        }

        const duration = Date.now() - startTime;
        logger.logSQL(sql, params, duration);

        return result;
      } catch (error: unknown) {
        logger.logSQLError(sql, params, error);
        console.error(error);
        const message = error instanceof Error ? error.message : 'Error executing Oracle command';
        throw new Error(message);
      }
    });
  }

  /**
   * Execute multiple commands in a single transaction with callback
   * @param commands - Array of command objects
   * @param callback - Optional callback function to execute after commands, determines commit/rollback
   * @returns Array of execute results
   */
  async commands<T>(commands: { sql: string; params: oracledb.BindParameters }[], callback?: (results: oracledb.Result<T>[]) => Promise<void> | void) {
    return await oracleConnection(this.dbName, async (connection) => {
      try {
        const results = await Promise.all(
          commands.map(async (command) => {
            const startTime = Date.now();
            try {
              const result = await connection.execute<T>(command.sql, command.params, this.options);
              const duration = Date.now() - startTime;
              logger.logSQL(command.sql, command.params, duration);

              return result;
            } catch (error: unknown) {
              logger.logSQLError(command.sql, command.params, error);
              throw error;
            }
          }),
        );

        // ถ้ามี callback ให้เรียกใช้ก่อน
        if (callback) {
          try {
            await callback(results);
            // ถ้า callback สำเร็จ ให้ commit
            await connection.commit();
          } catch (callbackError) {
            // ถ้า callback ไม่สำเร็จ ให้ rollback และ throw error
            await connection.rollback();
            throw callbackError;
          }
        } else {
          // ถ้าไม่มี callback ให้ใช้ logic เดิม
          if (results.some((result) => result.rowsAffected && result.rowsAffected > 0)) {
            await connection.commit();
          }
        }

        return results;
      } catch (error: unknown) {
        // ถ้าเกิด error ในการ execute commands ให้ rollback
        await connection.rollback();
        console.error(error);
        const message = error instanceof Error ? error.message : 'Error executing Oracle command';
        throw new Error(message);
      }
    });
  }

  /**
   * Execute batch INSERT/UPDATE/DELETE with executeMany
   * @param sql - SQL command string
   * @param params - Array of parameters for batch execution
   * @param bindDefs - Bind definitions
   * @returns Execute many result
   */
  async commandMany<T>(sql: string, params: oracledb.BindParameters[], bindDefs: oracledb.BindDefinition) {
    return await oracleConnection(this.dbName, async (connection) => {
      try {
        const startTime = Date.now();
        const options = {
          ...this.optionExecuteMany,
          bindDefs,
        } as oracledb.ExecuteManyOptions;
        const result = await connection.executeMany<T>(sql, params, options);

        if (result.batchErrors && result.batchErrors.length > 0) {
          await connection.rollback();
          logger.logSQLError(sql, params, result.batchErrors);
          throw new Error(result.batchErrors[0].message);
        }

        const duration = Date.now() - startTime;
        logger.logSQL(sql, params, duration);

        return result;
      } catch (error: unknown) {
        logger.logSQLError(sql, params, error);
        console.error(error);
        const message = error instanceof Error ? error.message : 'Error executing Oracle command';
        throw new Error(message);
      }
    });
  }

  /**
   * Execute single stored procedure
   * @param queries - Stored procedure configuration
   * @returns Stored procedure result
   */
  async commandSp<T>(queries: CommandsSpType): Promise<{ rowsAffected: number; output: T }> {
    try {
      const result = await this.commandsSp([queries]);

      return result[0] as { rowsAffected: number; output: T };
    } catch (error: unknown) {
      console.error(error);
      const message = error instanceof Error ? error.message : 'Error executing Oracle command';
      throw new Error(message);
    }
  }

  /**
   * Execute multiple stored procedures
   * @param queries - Array of stored procedure configurations
   * @returns Array of stored procedure results
   */
  async commandsSp(queries: CommandsSpType[]): Promise<{ rowsAffected: number; output: Record<string, unknown> }[]> {
    return await oracleConnection(this.dbName, async (connection) => {
      try {
        const startTime = Date.now();
        const output: {
          rowsAffected: number;
          output: Record<string, unknown>;
        }[] = [];

        for await (const obj of queries) {
          const _sql = `
              BEGIN
              ${obj.spName}(${
                obj.input
                  ? Object.keys(obj.input)
                      .map((x) => `:${x}`)
                      .join(', ')
                  : ''
              }${obj.input ? ',' : ''}${
                obj.output
                  ? Object.keys(obj.output)
                      .map((x) => `:${x}`)
                      .join(', ')
                  : ''
              });
            END;`;

          const convertParam = obj.input
            ? Object.keys(obj.input).reduce((pre, curr) => {
                return { ...pre, [curr]: obj.input![curr].value };
              }, {})
            : undefined;

          const sql = convertSQL('oracle', _sql, convertParam);

          const bindOutput: Record<string, unknown> = {};

          if (obj.output !== undefined) {
            Object.keys(obj.output).forEach((x) => {
              bindOutput[x] = {
                type: obj.output![x].type,
                dir: obj.output![x].dir,
                value: obj.output![x].value,
              };
            });
          }
          const callStartTime = Date.now();
          try {
            const res = await connection.execute(sql, bindOutput as oracledb.BindParameters, {
              autoCommit: false,
            });

            const duration = Date.now() - callStartTime;
            logger.LogSqlResult(sql, [convertParam], duration, res.rowsAffected, (res.outBinds as Record<string, unknown>) || {});

            output.push({
              rowsAffected: res.rowsAffected || 0,
              output: (res.outBinds as Record<string, unknown>) || {},
            });
          } catch (error: unknown) {
            const duration = Date.now() - callStartTime;
            logger.logSQLError(sql, convertParam, error);
            throw error;
          }
        }

        await connection.commit();

        return Promise.resolve(output);
      } catch (error: unknown) {
        console.error(error);
        const message = error instanceof Error ? error.message : 'Error executing Oracle command';
        throw new Error(message);
      }
    });
  }

  /**
   * Get SQL statement from SQL_TAB_OPPN table
   * @param sqlNo - SQL number
   * @param _appId - Application ID (optional)
   * @returns SQL statement string
   */
  async getSqlStmt(sqlNo: number, _appId?: number): Promise<string> {
    const appId = _appId ?? this.appID;
    const sqlTab = `SELECT  SQL_STMT FROM KPDBA.SQL_TAB_OPPN sto WHERE app_id = ${appId} AND sql_no = ${sqlNo}`;
    try {
      const startTime = Date.now();
      const result = await this.query<{ SQL_STMT: string }>(sqlTab, [], {
        fetchInfo: {
          SQL_STMT: { type: oracledb.STRING },
        },
      });

      const duration = Date.now() - startTime;
      logger.logSQL(sqlTab, [], duration);

      return result[0].SQL_STMT;
    } catch (error: unknown) {
      const appId = _appId ?? this.appID;
      logger.logSQLError(sqlTab, [], error);
      const message = error instanceof Error ? error.message : String(error);
      throw new Error(message);
    }
  }

  /**
   * Execute query from SQL_TAB_OPPN table
   * @param sqlNo - SQL number
   * @param params - Query parameters
   * @returns Query results
   */
  async queryFromSqlTab<T>(sqlNo: number, params: oracledb.BindParameters): Promise<T[]> {
    try {
      const sql = await this.getSqlStmt(sqlNo);
      const result = await this.query<T>(sql as string, params);
      return result;
    } catch (error: unknown) {
      const message = error instanceof Error ? error.message : String(error);
      throw new Error(message);
    }
  }

  /**
   * Execute command from SQL_TAB_OPPN table
   * @param sqlNo - SQL number
   * @param params - Command parameters
   * @returns Command result
   */
  async commandFromSqlTab<T>(sqlNo: number, params: oracledb.BindParameters): Promise<oracledb.Result<T>> {
    try {
      const sql = await this.getSqlStmt(sqlNo);
      const result = await this.command<T>(sql as string, params);
      return result;
    } catch (error: unknown) {
      const message = error instanceof Error ? error.message : String(error);
      throw new Error(message);
    }
  }
}

// Export singleton instance
const oracleInstance = new Oracle(process.env.ORACLE_DB_NAME || 'ORCL', process.env.APP_ID || '');
export { oracleInstance as oracle };
export default Oracle;
```

---

## 4. `server/types/oracleType.ts` (types ที่ใช้ร่วมกับ Oracle libs)

```ts
import oracledb from 'oracledb';

export type ITnsConfig = {
  DESCRIPTION: {
    ADDRESS_LIST: {
      ADDRESS: { PROTOCOL: string; HOST: string; PORT: string };
    };
    CONNECT_DATA: { SID: string; SERVER?: string; SRVR?: string };
  };
};

export type ITns = Record<string, ITnsConfig>;
export type IMssqlConfig = {
  user: string;
  password: string;
  server: string;
  pool: {
    max: number;
    min: number;
    idleTimeoutMillis: number;
  };
  options: {
    trustServerCertificate: boolean;
  };
};

// Oracle data types that can be used in stored procedures
export type OracleDataType = typeof oracledb.STRING | typeof oracledb.NUMBER | typeof oracledb.DATE | typeof oracledb.CURSOR | typeof oracledb.BUFFER | typeof oracledb.CLOB | typeof oracledb.BLOB;

// Oracle parameter value types
export type OracleParamValue = string | number | Date | Buffer | null | undefined;

export type InOutParamsType = {
  [key: string]: {
    type: OracleDataType;
    value?: OracleParamValue;
    dir?: typeof oracledb.BIND_IN | typeof oracledb.BIND_OUT | typeof oracledb.BIND_INOUT;
  };
};

export type CommandsSpType = {
  spName: string;
  input: InOutParamsType;
  output: InOutParamsType | undefined;
};
```

---

## 5. ตัวอย่างการใช้งานแบบ generic (นำไปใช้กับโปรเจกต์อื่นได้)

### 5.1 ใช้ singleton `oracle` จาก `server/libs/oracle/index.ts`

แนวคิดคือให้มีไฟล์รวม utility ชื่อ `libs/oracle/index.ts` ที่ export singleton `oracle` แล้วให้ layer อื่น (เช่น repository/service) import มาใช้:

```ts
// ในไฟล์ repository ใด ๆ
import { oracle } from '../libs/oracle';

// กำหนดรูปแบบ row ที่จะได้จากฐานข้อมูล
interface ExampleRow {
  ID: number;
  NAME: string;
}

// ใช้ query อ่านข้อมูล
const rows = await oracle.query<ExampleRow>(
  `
    SELECT ID, NAME
    FROM SOME_TABLE
    WHERE STATUS = :status
  `,
  {
    status: 'A',
  },
);
```

หลัก ๆ คือ

- import `oracle` จาก path กลางของโปรเจกต์ (เช่น `../libs/oracle`)
- สร้าง interface/ประเภทของ row เองให้ตรงกับคอลัมน์ของโปรเจกต์นั้น
- ส่ง `params` เป็น object ธรรมดา (key → ชื่อ bind parameter)

### 5.2 ใช้คลาส `Oracle` โดยตรง กรณีไม่อยากใช้ singleton

ในบางโปรเจกต์อาจต้องการสร้าง instance ของ `Oracle` แยกตาม database/app ID:

```ts
import Oracle from '../libs/oracle';

class ExampleRepository {
  private oracle: Oracle;

  constructor() {
    this.oracle = new Oracle(process.env.ORACLE_DB_NAME, process.env.APP_ID);
  }

  async getItems() {
    return this.oracle.query(
      `
        SELECT *
        FROM SOME_TABLE
      `,
    );
  }
}

export const exampleRepository = new ExampleRepository();
```

แนวคิดที่สำคัญ:

- ใช้ `new Oracle(dbName, appId)` ถ้าต้องการ control context เอง
- หรือใช้ singleton `oracle` ที่ export ไว้ ถ้าทั้งระบบใช้ค่าเดียวกัน

### 5.4 การ init Oracle Client ใน root `server/index.ts`

ตัวอย่างจาก [server/index.ts](file:///Users/admin/Documents/Code/Ingeni/OPPN/kim-pai-repair-flow-new/server/index.ts):

```ts
import 'dotenv/config';
import express from 'express';
import cors from 'cors';
import oracledb from 'oracledb';
import routes from './routes';
import { loggingMiddleware } from './middlewares/logging';
import { contextMiddleware } from './middlewares/contextMiddleware';

export function createServer(skipErrorHandlers = false) {
  if (process.env.ORACLE_CLIENT_PATH) {
    console.log('Initializing Oracle Client from:', process.env.ORACLE_CLIENT_PATH);
    try {
      oracledb.initOracleClient?.({
        libDir: process.env.ORACLE_CLIENT_PATH,
      });
    } catch (err: unknown) {
      const error = err as Error;
      if (error.message.includes('DPI-1047')) {
        console.warn('Oracle Client initialization skipped (already initialized or not needed for Thin mode).');
      } else {
        console.error('Failed to initialize Oracle Client:', error);
        throw new Error(`Cannot load Oracle Client from ${process.env.ORACLE_CLIENT_PATH}. Error: ${error.message}`);
      }
    }
  } else {
    console.warn('ORACLE_CLIENT_PATH is not set. Ensure the Oracle Client is installed and configured.');
  }
  const app = express();

  app.use(cors());
  app.use(express.json());
  app.use(express.urlencoded({ extended: true }));
  app.use(contextMiddleware);
  app.use(loggingMiddleware);

  app.use('/api', routes);

  return app;
}
```

---
