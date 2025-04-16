# Backend (NestJS) Intern Assignment: Real-Time Attendance System

## Introduction

This project involves creating a **real-time attendance system** using NestJS, WebSockets, and Redis caching. The goal is to enable teachers to mark attendance, and students to receive instant updates. Additionally, the system should handle offline scenarios, ensuring data synchronization when connectivity is restored.

This document provides step-by-step instructions, explanations, and examples to help you understand and implement each part of the system.


## 1. **Understanding the Core Concepts**

### What is WebSocket?

WebSocket is a protocol that allows real-time, two-way communication between a server and clients (like browsers or mobile apps). Unlike traditional HTTP requests, WebSocket maintains an open connection, enabling instant data exchange.

### What is Redis?

Redis is an in-memory database used for caching data. It helps store data temporarily for quick access, reducing load on your main database and improving performance.

### Why combine WebSocket and Redis?

- WebSocket provides real-time updates.
- Redis caches data for quick retrieval.
- Together, they make the system fast, scalable, and responsive.


## 2. **Setting Up Your Project**

### Tech Stack

| Component | Technology/Package | Purpose |
| :-- | :-- | :-- |
| Framework | NestJS | Core backend |
| WebSockets | `@nestjs/websockets`, `socket.io` | Real-time communication |
| Redis | `redis`, `@nestjs/cache-manager`, `cache-manager-redis-store` | Caching attendance data |
| Database | MongoDB | Persistent storage of records |
| Testing | Jest | Testing your code |

## 3. **Initial Setup**

### Step 1: Create a new NestJS project

```bash
nest new attendance-system
cd attendance-system
```


### Step 2: Install necessary packages

```bash
npm install --save @nestjs/websockets @nestjs/platform-socket.io socket.io redis @nestjs/cache-manager cache-manager-redis-store
```


### Step 3: Create a WebSocket Gateway

Generate a gateway:

```bash
nest generate gateway attendance
```

This creates `attendance.gateway.ts` and its test file
## 4. **Implementing the WebSocket Gateway**

### Basic Gateway Structure

```typescript
import { WebSocketGateway, WebSocketServer, SubscribeMessage, MessageBody, ConnectedSocket } from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';

@WebSocketGateway({ cors: true }) // Enable CORS for testing
export class AttendanceGateway {
  @WebSocketServer()
  server: Server;

  // When a teacher marks attendance
  @SubscribeMessage('markAttendance')
  handleMarkAttendance(@MessageBody() data: any, @ConnectedSocket() client: Socket): void {
    // Logic to handle attendance marking
  }

  // When a student subscribes to updates
  @SubscribeMessage('subscribeToClass')
  handleSubscribe(@MessageBody() data: { classId: string }, @ConnectedSocket() client: Socket): void {
    client.join(`class_${data.classId}`);
  }
}
```


### Explanation:

- `@WebSocketGateway`: Decorator to define a WebSocket server.
- `@WebSocketServer()`: Injects the server instance.
- `@SubscribeMessage()`: Handles specific message events.
- `client.join()`: Adds clients to specific rooms (like classes).


## 5. **Caching Attendance Data with Redis**

### Step 1: Setup Redis Cache Module

Create a cache module in NestJS:

```typescript
import { Module, CacheModule } from '@nestjs/common';
import * as redisStore from 'cache-manager-redis-store';

@Module({
  imports: [
    CacheModule.register({
      store: redisStore,
      host: 'localhost',
      port: 6379,
      ttl: 3600, // cache expiry in seconds
    }),
  ],
  exports: [CacheModule],
})
export class CacheConfigModule {}
```


### Step 2: Inject Cache in Gateway

```typescript
import { CACHE_MANAGER, Inject } from '@nestjs/common';
import { Cache } from 'cache-manager';

export class AttendanceGateway {
  constructor(@Inject(CACHE_MANAGER) private cacheManager: Cache) {}

  async handleMarkAttendance(data: any): Promise&lt;void&gt; {
    const { classId, studentId, status } = data;
    const cacheKey = `attendance_${classId}_${studentId}_${new Date().toISOString().slice(0,10)}`;

    // Save attendance in Redis
    await this.cacheManager.set(cacheKey, { status, timestamp: new Date() });
    
    // Broadcast to class room
    this.server.to(`class_${classId}`).emit('attendanceUpdate', { studentId, status });
  }
}
```


### Explanation:

- Cache attendance data with a key based on class, student, and date.
- Broadcast updates to all clients in the class room.


## 6. **Handling Offline Scenarios \& Data Synchronization**

### Approach:

- Store attendance data locally when offline.
- When online, sync data with the server.
- Use a queue to manage unsent data.


### Implementation Tips:

- Use local storage (like AsyncStorage in React Native or local database).
- When connectivity is restored, send queued data to the server.
- On server, process and store data, then update Redis cache.


### Example:

```typescript
// Pseudo-code for offline queue
class OfflineSync {
  private queue: any[] = [];

  addToQueue(data: any) {
    this.queue.push(data);
  }

  async sync() {
    while (this.queue.length &gt; 0) {
      const data = this.queue.shift();
      await this.sendToServer(data);
    }
  }

  async sendToServer(data: any) {
    // Call API to sync data
  }
}
```

## 7. **Testing Your System**

### What to Test:

- WebSocket connection establishment
- Event handling (`markAttendance`, `subscribeToClass`)
- Redis caching (cache hit/miss)
- Offline sync logic


### Example Test Snippet:

```typescript
import { Test, TestingModule } from '@nestjs/testing';

describe('AttendanceGateway', () =&gt; {
  let gateway: AttendanceGateway;

  beforeEach(async () =&gt; {
    const module: TestingModule = await Test.createTestingModule({
      providers: [AttendanceGateway],
    }).compile();

    gateway = module.get&lt;AttendanceGateway&gt;(AttendanceGateway);
  });

  it('should be defined', () =&gt; {
    expect(gateway).toBeDefined();
  });

  // Additional tests for events and caching
});
```

## 8. **Additional Tips \& References**

- **Security**: Implement JWT authentication for WebSocket connections.
- **Rooms**: Use `client.join()` and `server.to(room).emit()` for targeted messaging.
- **Scalability**: Use Redis adapter for socket.io to run across multiple server instances ([see socket.io-redis](https://socket.io/docs/v4/redis-adapter/)).
- **Testing WebSockets**: Use `socket.io-client` for client-side testing ([see demo here](https://dev.to/jfrancai/demystifying-nestjs-websocket-gateways-a-step-by-step-guide-to-effective-testing-1a1f)).


## 9. **Summary**

| Step | What to Do | Purpose |
| :-- | :-- | :-- |
| Setup | Create NestJS project, install packages | Foundation of the system |
| Gateway | Build WebSocket gateway with event handlers | Real-time communication |
| Cache | Setup Redis cache, cache attendance data | Fast data retrieval |
| Offline | Implement offline data queue and sync | Offline support |
| Test | Write unit and integration tests | Ensure reliability |

## 10. **Helpful Links \& References**

- [NestJS WebSocket Gateway Guide](https://dev.to/jfrancai/demystifying-nestjs-websocket-gateways-a-step-by-step-guide-to-effective-testing-1a1f)
- [Socket.io Redis Adapter](https://socket.io/docs/v4/redis-adapter/)
- [Redis Cache in Node.js](https://digitalocean.com/community/tutorials/how-to-implement-caching-in-node-js-using-redis)
- [NestJS WebSocket Basics](https://delightfulengineering.com/blog/nest-websockets/basics/)
- [WebSocket in NestJS YouTube Demo](https://www.youtube.com/watch?v=eEa3u3wyYu4)


## 11. **Submission Instructions**  

- Make your repository **private**.
- Add `ankurkr07` and `ayushflows` as **collaborators**.
- Send us the **link of your GitHub repo**.
- Ensure your project has a **proper README** explaining your setup, how to run the project, and any important details you wanna tell us.


This detailed guide should help you understand each part of the assignment and implement it step-by-step. Remember, focus on understanding how WebSockets, Redis, and offline sync work together to create a fast, reliable, and scalable attendance system.

**Good luck!** 🚀
