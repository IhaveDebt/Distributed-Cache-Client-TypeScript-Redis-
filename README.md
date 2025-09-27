/**
 * Distributed Cache Client (dist_cache.ts)
 *
 * Simple TypeScript client that demonstrates:
 * - Read-through cache with a primary Redis node and optional replica fallback
 * - Basic consistency option (WRITE_THROUGH vs WRITE_BACK)
 *
 * Requirements:
 *  npm install ioredis
 *
 * Usage:
 *  ts-node src/dist_cache.ts
 */
import Redis from 'ioredis';

const PRIMARY = new Redis({ host: '127.0.0.1', port: 6379 }); // primary
const REPLICA = new Redis({ host: '127.0.0.1', port: 6380 }); // replica (optional)

type Consistency = 'WRITE_THROUGH' | 'WRITE_BACK';

class DistCache {
  primary: Redis.Redis;
  replica?: Redis.Redis;
  consistency: Consistency;
  constructor(primary: Redis.Redis, replica?: Redis.Redis, consistency: Consistency = 'WRITE_THROUGH') {
    this.primary = primary;
    this.replica = replica;
    this.consistency = consistency;
  }

  async get(key: string): Promise<string | null> {
    // Try primary
    let v = await this.primary.get(key);
    if (v != null) return v;
    // Optionally fall back to replica
    if (this.replica) {
      v = await this.replica.get(key);
      if (v != null) {
        // refresh primary
        await this.primary.set(key, v);
      }
    }
    return v;
  }

  async set(key: string, value: string): Promise<void> {
    if (this.consistency === 'WRITE_THROUGH') {
      await this.primary.set(key, value);
      if (this.replica) await this.replica.set(key, value);
    } else {
      // WRITE_BACK: write to primary, schedule replica sync (naive)
      await this.primary.set(key, value);
      setTimeout(async () => {
        if (this.replica) await this.replica.set(key, value);
      }, 500);
    }
  }

  async del(key: string) {
    await this.primary.del(key);
    if (this.replica) await this.replica.del(key);
  }
}

// Demo
(async function demo() {
  const cache = new DistCache(PRIMARY, REPLICA, 'WRITE_THROUGH');
  console.log('Setting foo=bar');
  await cache.set('foo', 'bar');
  console.log('Getting foo:', await cache.get('foo'));
  await cache.del('foo');
  console.log('After delete, foo:', await cache.get('foo'));
  process.exit(0);
})();
