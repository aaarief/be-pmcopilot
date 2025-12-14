# be-pmcopilot
Berikut repository berisi readme.md untuk mereplikasi langkah-langkah pembuatan backend

# PM Copilot Backend API

[![Node.js](https://img.shields.io/badge/Node.js-18+-green.svg)](https://nodejs.org/)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.0+-blue.svg)](https://www.typescriptlang.org/)
[![Prisma](https://img.shields.io/badge/Prisma-5.0+-2D3748.svg)](https://www.prisma.io/)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

Backend API untuk sistem **Predictive Maintenance Copilot** yang mengintegrasikan Machine Learning prediction, automated ticketing, dan AI chatbot assistant untuk monitoring kesehatan mesin industri secara real-time.

## Deskripsi

PM Copilot Backend adalah REST API yang menghubungkan sensor data mesin, model Machine Learning untuk prediksi kegagalan, dan AI-powered chatbot untuk membantu engineer dalam maintenance preventif. Sistem ini mampu:

-  Mengambil data sensor dari 20 mesin secara berkala (suhu, RPM, torque, tool wear)
-  Menganalisis pola anomali menggunakan ML model
-  Membuat maintenance ticket secara otomatis ketika potensi kerusakan terdeteksi
-  Menyediakan AI assistant untuk konsultasi dan diagnosis
-  Caching prediksi dengan Redis untuk performa optimal

##  Fitur Utama

### 1. **Real-time Machine Monitoring**
- Monitoring 20 mesin industri (12 L-sized, 6 M-sized, 2 H-sized)
- Sensor data: Air Temperature, Process Temperature, RPM, Torque, Tool Wear
- Status detection: Normal, Warning, Maintenance Mode
- Real-time updates via Socket.IO

### 2. **Automated ML Prediction**
- Integrasi dengan ML API untuk prediksi failure
- 4 tipe failure detection:
  - **Tool Wear Failure** (TWF)
  - **Heat Dissipation Failure** (HDF)
  - **Power Failure** (PWF)
  - **Overstrain Failure** (OSF)
- Risk score calculation (0-100)
- Countdown time-to-failure estimation

### 3. **Auto-Ticketing System**
- Ticket otomatis dibuat ketika prediksi failure terdeteksi
- AI-generated ticket title & description (Google Gemini 2.0)
- Priority logic:
  - **URGENT**: ≤24 jam
  - **HIGH**: ≤72 jam
  - **MEDIUM**: Berdasarkan failure type
- Anti-duplikasi (tidak membuat ticket baru jika sudah ada ticket OPEN)

### 4. **AI Chatbot Assistant (Gemini-Powered)**
- Conversational interface untuk maintenance engineer
- Kemampuan:
  - Menjawab pertanyaan global ("Berapa jumlah mesin yang rusak?")
  - Deep-dive spesifik mesin ("Bagaimana kondisi L_001?")
  - Rekomendasi maintenance berdasarkan sensor data
- Session-based chat dengan history (LangChain)
- Auto-detection Machine ID dari konteks percakapan

### 5. **Scheduler (Background Jobs)**
- Automated batch prediction untuk semua mesin
- Configurable interval (default: setiap jam)
- Manual trigger tersedia via API endpoint
- State management dengan Redis

### 6. **Performance Optimization**
- Redis caching (TTL 55 menit, hit rate ~80%)
- Anti race-condition dengan in-memory lock
- Database query optimization dengan indexing
- Atomic transactions (Prisma $transaction)

##  Tech Stack

| Kategori | Technology |
|----------|-----------|
| **Runtime** | Node.js 18+ |
| **Framework** | Express.js |
| **Database** | PostgreSQL (Neon Serverless) |
| **ORM** | Prisma 5.0+ |
| **Cache** | Redis (Upstash) |
| **AI/ML** | Google Gemini 2.0 Flash (LangChain) |
| **Validation** | Zod |
| **Real-time** | Socket.IO |
| **Documentation** | Swagger/OpenAPI |
| **Deployment** | Vercel (Serverless Functions) |

##  Installation

### Prerequisites

```bash
Node.js >= 18.0.0
PostgreSQL database
Redis instance
Google Gemini API Key
ML API endpoint
```

### Setup Steps

1. **Clone repository**
```bash
git clone https://github.com/your-org/pmcopilot-backend.git
cd pmcopilot-backend
```

2. **Install dependencies**
```bash
npm install
```

3. **Setup environment variables**
```bash
cp .env.example .env
```

Edit `.env` dengan credentials Anda:
```env
# Database
DATABASE_URL="postgresql://user:password@host:5432/dbname"
DIRECT_URL="postgresql://user:password@host:5432/dbname"

# Redis
REDIS_URL="redis://default:password@host:6379"

# AI Services
GOOGLE_API_KEY="your-gemini-api-key"

# ML API
ML_API_URL="https://your-ml-api.com"

# Server
PORT=3000
NODE_ENV=development
```

4. **Run database migrations**
```bash
npx prisma migrate dev
```

5. **Generate Prisma client**
```bash
npx prisma generate
```

6. **(Optional) Seed initial data**
```bash
npx prisma db seed
```

7. **Start development server**
```bash
npm run dev
```

Server akan berjalan di `http://localhost:3000`

##  API Endpoints

### Machines

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/machines` | List all machines (pagination support) |
| `GET` | `/machines/:machineId` | Get specific machine detail |
| `POST` | `/machines` | Trigger batch ML prediction (all machines) |
| `POST` | `/machines/:machineId` | Trigger prediction for specific machine |

**Query Parameters:**
- `limit`: Number of results (default: 20)
- `offset`: Pagination offset (default: 0)
- `status`: Filter by status (normal, warning, maintenance)

### Tickets

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/tickets` | List maintenance tickets |
| `GET` | `/tickets/:id` | Get ticket detail |
| `PATCH` | `/tickets/:id/status` | Update ticket status (open/close) |
| `GET` | `/tickets/history` | Get archived tickets |

**Query Parameters:**
- `machineId`: Filter by machine
- `status`: Filter by status (open, closed)
- `priority`: Filter by priority (urgent, high, medium, low)

### AI Chatbot

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/agent/chat` | Send message to AI assistant |
| `GET` | `/agent/chat/sessions` | List chat sessions |
| `GET` | `/agent/chat/:sessionId` | Get chat history |

**Request Body (POST /agent/chat):**
```json
{
  "message": "Berapa mesin yang butuh maintenance?",
  "sessionId": "optional-session-id"
}
```

### Scheduler

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/scheduler/status` | Check scheduler status |
| `POST` | `/scheduler/trigger` | Manually trigger batch update |

## Architecture

### Folder Structure

```
backend-pmcopilot-main/
├── api/                        # Vercel serverless functions
├── prisma/
│   └── schema.prisma          # Database schema
├── src/
│   ├── app.js                 # Express app configuration
│   ├── index.js               # Server entry point
│   ├── swagger.yaml           # API documentation
│   ├── config/
│   │   └── index.js           # Environment config
│   ├── modules/
│   │   ├── machines/          # Machine & ML prediction
│   │   │   ├── machines.controller.js
│   │   │   ├── machines.schema.js
│   │   │   ├── providers/
│   │   │   │   └── mlApi.provider.js
│   │   │   ├── services/
│   │   │   │   ├── batch.service.js
│   │   │   │   ├── cache.helper.js
│   │   │   │   ├── dataAccess.service.js
│   │   │   │   ├── dbTicket.service.js
│   │   │   │   ├── prediction.core.js
│   │   │   │   └── prediction.service.js
│   │   │   └── utils/
│   │   │       ├── payload.mapper.js
│   │   │       ├── recommendation.generator.js
│   │   │       └── risk.calculator.js
│   │   ├── tickets/           # Ticket management
│   │   │   ├── tickets.controller.js
│   │   │   ├── tickets.schema.js
│   │   │   └── services/
│   │   │       ├── ticket.ai.js
│   │   │       ├── ticket.crud.js
│   │   │       └── ticket.generator.js
│   │   ├── agent/             # AI Chatbot
│   │   │   ├── agent.controller.js
│   │   │   ├── agent.schema.js
│   │   │   └── services/
│   │   │       └── chat.service.js
│   │   └── scheduler/         # Background jobs
│   │       ├── scheduler.controller.js
│   │       ├── scheduler.schema.js
│   │       └── jobs/
│   │           └── predictAll.job.js
│   ├── shared/
│   │   ├── ai/
│   │   │   └── gemini.factory.js    # Gemini AI client
│   │   ├── cache/
│   │   │   └── redis.js             # Redis client
│   │   ├── db/
│   │   │   └── prisma.js            # Prisma client
│   │   ├── middleware/
│   │   │   ├── errorHandler.js
│   │   │   ├── logger.js
│   │   │   └── validator.js
│   │   └── utils/
│   │       ├── converter.js
│   │       └── customError.js
│   └── routes/
│       └── index.js                  # Route registry
└── package.json
```



### Database Schema

```prisma
model Machine {
  machineId       String              @id
  type            MachineType
  predictions     Prediction[]
  tickets         MaintenanceTicket[]
  ticketHistory   TicketHistory[]
  createdAt       DateTime            @default(now())
  updatedAt       DateTime            @updatedAt
}

model Prediction {
  id                  Int          @id @default(autoincrement())
  machineId           String
  machine             Machine      @relation(fields: [machineId], references: [machineId])
  airTemperature      Float
  processTemperature  Float
  rpm                 Float
  torque              Float
  toolWear            Float
  failureType         String
  riskScore           Float
  forecastTimestamp   DateTime?
  countdown           Int?
  recommendations     String?
  createdAt           DateTime     @default(now())
}

model MaintenanceTicket {
  id              Int          @id @default(autoincrement())
  machineId       String
  machine         Machine      @relation(fields: [machineId], references: [machineId])
  title           String
  description     String
  priority        Priority
  status          TicketStatus @default(OPEN)
  createdAt       DateTime     @default(now())
  updatedAt       DateTime     @updatedAt
}
```

##  Key Implementation Details

### 1. Risk Score Calculation

```javascript
const calculateRiskScore = (sensorData) => {
  const tempScore = normalizeTempScore(sensorData.processTemperature);
  const wearScore = normalizeWearScore(sensorData.toolWear);
  const torqueScore = normalizeTorqueScore(sensorData.torque);
  const rpmScore = normalizeRpmScore(sensorData.rpm);
  
  // Weighted formula
  const riskScore = 
    (tempScore * 0.35) + 
    (wearScore * 0.30) + 
    (torqueScore * 0.20) + 
    (rpmScore * 0.15);
    
  return Math.round(riskScore * 100) / 100;
};
```

### 2. Failure Type Detection

| Failure Type | Condition |
|--------------|-----------|
| **Tool Wear Failure (TWF)** | Tool Wear > 200 min |
| **Heat Dissipation Failure (HDF)** | Δ(Process - Air Temp) < 8.6K |
| **Power Failure (PWF)** | Power < 3500W atau > 9000W |
| **Overstrain Failure (OSF)** | Load > Threshold (varies by machine type) |

### 3. Ticket Priority Logic

```javascript
const determinePriority = (countdown, failureType) => {
  if (countdown <= 24) return 'URGENT';      // ≤24 jam
  if (countdown <= 72) return 'HIGH';        // ≤72 jam
  if (failureType === 'Power Failure') return 'HIGH';
  if (failureType === 'Heat Dissipation Failure') return 'MEDIUM';
  return 'LOW';
};
```

### 4. AI System Prompt (Dynamic)

```javascript
const buildSystemPrompt = async (machineId) => {
  const prompt = `
    ${DOMAIN_KNOWLEDGE}        // Formulas & thresholds
    ${await getFleetOverview()} // Global stats
    ${machineId ? await getMachineContext(machineId) : ''}
  `;
  return prompt;
};
```

##  Testing

### Run Tests

```bash
npm test
```

### Manual API Testing

```bash
# Test machine list
curl -X GET http://localhost:3000/machines

# Test specific machine
curl -X GET http://localhost:3000/machines/L_001

# Test chatbot
curl -X POST http://localhost:3000/agent/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"Ada berapa mesin yang rusak?"}'

# Trigger manual prediction
curl -X POST http://localhost:3000/machines/L_001
```

##  Performance Metrics

| Metric | Value |
|--------|-------|
| **Redis Cache Hit Rate** | ~80% |
| **Average Response Time** | <200ms (cached), <1s (ML API call) |
| **Database Queries** | Indexed on machineId, status, createdAt |
| **Concurrent Requests** | Anti race-condition dengan in-memory lock |
| **Uptime** | 99.9% (Vercel SLA) |

##  Deployment

### Vercel (Production)

1. **Connect repository** ke Vercel
2. **Set environment variables** di Vercel Dashboard
3. **Configure build settings**:
   ```json
   {
     "buildCommand": "npm run build",
     "outputDirectory": "dist",
     "installCommand": "npm install"
   }
   ```
4. **Deploy**: Automatic pada setiap push ke `main` branch

### Vercel Cron Jobs

Edit `vercel.json`:
```json
{
  "crons": [{
    "path": "/api/scheduler/trigger",
    "schedule": "0 * * * *"
  }]
}
```

##  Troubleshooting

### Common Issues

| Issue | Solution |
|-------|----------|
| **ML API Timeout** | Check `ML_API_URL` dan network connectivity |
| **Redis Connection Failed** | Verify `REDIS_URL` dan firewall rules |
| **Prisma Migration Error** | Run `npx prisma generate` |
| **Gemini Rate Limit** | Implement exponential backoff (sudah built-in) |
| **Socket.IO CORS** | Add frontend URL ke CORS whitelist |

### Debug Mode

```bash
DEBUG=* npm run dev
```

##  API Documentation

Akses **Swagger UI** untuk interactive documentation:

```
http://localhost:3000/api-docs
``` 




