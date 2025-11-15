<h1 align="center">pg2s-baileys</h1>
<p align="center">
  <a href="https://nodejs.org/"><img src="https://img.shields.io/badge/Node.js-%3E%3D18-blue.svg" alt="Node.js version" /></a>
  <a href="https://www.npmjs.com/package/pg2s-baileys"><img src="https://badge.fury.io/js/pg2s-baileys.svg?cache=0" alt="npm version" /></a>
  <a href="https://www.npmjs.com/package/pg2s-baileys"><img src="https://img.shields.io/npm/dt/pg2s-baileys.svg?cache=0" alt="npm downloads" /></a>
  <a href="https://opensource.org/licenses/MIT"><img src="https://img.shields.io/badge/License-MIT-yellow.svg" alt="License" /></a>
</p>

> Lightweight Baileys auth state adapter using **PostgreSQL** and **NodeCache**  
> Built for efficient, persistent, and cache-optimized WhatsApp authentication sessions with multi-instance support.

## Table of Contents

- [Installation](#installation)
- [Usage](#usage)
- [Description](#description)
- [License](#license)

## Installation

Install the package using your preferred JavaScript package manager:

### npm
```bash
npm install pg2s-baileys
```

### pnpm
```bash
pnpm add pg2s-baileys
```

### yarn
```yarn
yarn add pg2s-baileys
```

### bun
```bash
bun add pg2s-baileys
```

### Dependencies
- Requires `pg` (PostgreSQL client) and `node-cache` (installed automatically).
- Ensure you have a running PostgreSQL server (version 12+ recommended).

## Usage

### Single Instance
```javascript
import { makeWASocket, DisconnectReason, proto, initAuthCreds, BufferJSON } from '@whiskeysockets/baileys';
// or
// import { makeWASocket, DisconnectReason, proto, initAuthCreds, BufferJSON } from 'baileys';
import usePostgresAuthState from 'pg2s-baileys';

const { state, saveCreds, resetSession } = await usePostgresAuthState(
  { connectionString: 'postgres://user:pass@localhost:5432/mydb' },  // PostgreSQL connection options (object)
  '6281234567890',  // phoneNumber (string) for multi-instance support
  { proto, initAuthCreds, BufferJSON }  // Baileys utils
);

// Initialize WhatsApp socket
const sock = makeWASocket({
  auth: state,
});

// Save credentials on update
sock.ev.on('creds.update', saveCreds);

// Example: Reset session
sock.ev.on('connection.update', async (update) => {
  const { lastDisconnect, connection } = update;
  if (connection === "close") {
    const statusCode = lastDisconnect?.error?.output?.statusCode;
    if (statusCode === DisconnectReason.loggedOut) {
      await resetSession();
    }
  }
});
```

### Multi-Instance (Multiple Sessions)
For handling multiple WhatsApp accounts in the same app (e.g., different phone numbers), call `usePostgresAuthState` multiple times with different `phoneNumber` values. All data is stored in a shared PostgreSQL table (`auth_state`), scoped by `phone_number`.

```javascript
import { makeWASocket, DisconnectReason, proto, initAuthCreds, BufferJSON } from '@whiskeysockets/baileys';
import usePostgresAuthState from 'pg2s-baileys';

const connOptions = { connectionString: 'postgres://user:pass@localhost:5432/mydb' };
const baileysUtils = { proto, initAuthCreds, BufferJSON };

// Instance 1: Phone A
const { state: state1, saveCreds: saveCreds1, resetSession: resetSession1 } = await usePostgresAuthState(
  { connectionString: 'postgres://user:pass@localhost:5432/mydb' },
  '6281234567890',
  { proto, initAuthCreds, BufferJSON }
);
const sock1 = makeWASocket({ auth: state1 });
sock1.ev.on('creds.update', saveCreds1);
sock1.ev.on('connection.update', async (update) => {
  if (update.connection === 'close' && update.lastDisconnect?.error?.output?.statusCode === DisconnectReason.loggedOut) {
    await resetSession1();
  }
});

// Instance 2: Phone B
const { state: state2, saveCreds: saveCreds2, resetSession: resetSession2 } = await usePostgresAuthState(
  { connectionString: 'postgres://user:pass@localhost:5432/mydb' },
  '6289876543210',
  { proto, initAuthCreds, BufferJSON }
);
const sock2 = makeWASocket({ auth: state2 });
sock2.ev.on('creds.update', saveCreds2);
sock2.ev.on('connection.update', async (update) => {
  if (update.connection === 'close' && update.lastDisconnect?.error?.output?.statusCode === DisconnectReason.loggedOut) {
    await resetSession2();
  }
});

// Now manage sock1 and sock2 independently (e.g., in an array or map)
```

## Description

`pg2s-baileys` is a drop-in replacement for Baileys' default auth state management. It replaces file-based JSON storage with PostgreSQL for scalable persistence (using connection pooling via `pg.Pool`) and NodeCache for fast in-memory access (no auto-TTL for auth data).

**Key Benefits:**
- **Efficiency**: Reduces I/O with caching and async queries.
- **Scalability**: Supports multi-instance via `phoneNumber` parameter; ideal for high-traffic bots handling multiple WhatsApp accounts.
- **Persistence**: All auth data (creds, keys, signals) in a single PostgreSQL table with BIGINT `phone_number` for scoping.
- **Lightweight**: Minimal overhead, with manual UPSERT for writes and error handling.

Ideal for production WhatsApp bots with multiple sessions. Use a single database for all instances.

**Arguments:**
- `connOptions` (object): PostgreSQL config, e.g., `{ connectionString: 'postgres://...' }` or full `{ host, port, user, password, database }`.
- `phoneNumber` (string): Numeric phone number (e.g., "6281234567890") for scoping auth data; required for multi-instance.
- `baileysUtils` (object): `{ proto, initAuthCreds, BufferJSON }` from Baileys. Validates inputs for errors.

**Error Handling:** Throws descriptive errors for invalid inputs, DB failures, or missing dependencies. Logs errors to console for reads/writes. Ensures numeric `phoneNumber` conversion to BIGINT.

**Database Schema (Auto-Created):**
```sql
CREATE TABLE IF NOT EXISTS auth_state (
  id SERIAL PRIMARY KEY,
  phone_number BIGINT NOT NULL,
  key TEXT NOT NULL,
  value TEXT
);
```

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

Developed with ❤️ by [Brother of Ijul](https://github.com/brotherofijul).