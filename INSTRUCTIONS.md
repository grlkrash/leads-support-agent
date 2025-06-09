# AI Agent for SMBs - Realtime Voice-Enabled Multi-Channel Platform Implementation Instructions

## Project Overview

This implementation guide covers the setup and configuration of a comprehensive voice and chat-enabled AI agent platform for Small to Medium-Sized Businesses. The system includes:

- **Enhanced OpenAI Realtime API Integration** with bidirectional audio streaming and improved CallSid handling
- **Advanced WebSocket Architecture** for low-latency voice communication with audio queue management
- **Asynchronous Voice Processing** with non-blocking AI response generation
- **Plan Tier Architecture** (FREE, BASIC, PRO) with Realtime API access
- **Enhanced Emergency Handling** across all channels with real-time processing
- **Redis Session Management** with WebSocket tracking and analytics
- **Multi-Channel Lead Capture** and routing with real-time processing

## Phase 0: Enhanced Project Setup

### 1. Initialize Node.js Project with Voice Dependencies

```bash
mkdir leads-support-agent-smb
cd leads-support-agent-smb
npm init -y

# Core dependencies
npm install express dotenv openai pg prisma twilio redis ioredis

# Voice and session dependencies
npm install @types/twilio ssml-builder

# Development dependencies
npm install -D nodemon typescript ts-node @types/express @types/node @types/redis

# Initialize TypeScript
npx tsc --init
```

### 2. Enhanced package.json Scripts

```json
{
  "scripts": {
    "dev": "nodemon src/server.ts",
    "dev:voice": "NODE_ENV=development nodemon src/server.ts",
    "build": "yarn prisma:generate && tsc",
    "start": "node dist/server.js",
    "prisma:generate": "prisma generate",
    "prisma:migrate": "prisma migrate dev",
    "redis:cleanup": "node scripts/redis-cleanup.js",
    "test:voice": "jest tests/voice",
    "test:plans": "jest tests/plans",
    "health:check": "curl http://localhost:3000/health | jq"
  }
}
```

### 3. Enhanced Prisma Setup with Voice & Plan Features

```bash
npx prisma init --datasource-provider postgresql
```

Update `prisma/schema.prisma` with the complete enhanced schema:

```prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["postgresqlExtensions"]
  binaryTargets   = ["native", "linux-musl-arm64-openssl-3.0.x"]
}

datasource db {
  provider   = "postgresql"
  url        = env("DATABASE_URL")
  directUrl  = env("DIRECT_URL")
  extensions = [vector]
}

model Business {
  id                      String          @id @default(cuid())
  name                    String
  businessType            BusinessType    @default(OTHER)
  planTier                PlanTier        @default(FREE)
  twilioPhoneNumber       String?         @unique
  notificationEmail       String?
  notificationPhoneNumber String?
  createdAt               DateTime        @default(now())
  updatedAt               DateTime        @updatedAt
  
  agentConfig             AgentConfig?
  knowledgeBase           KnowledgeBase[]
  leads                   Lead[]
  users                   User[]

  @@map("businesses")
}

model AgentConfig {
  id                           String                @id @default(cuid())
  businessId                   String                @unique
  agentName                    String                @default("AI Assistant")
  personaPrompt                String                @default("You are a helpful and friendly assistant.")
  welcomeMessage               String                @default("Hello! How can I help you today?")
  colorTheme                   Json                  @default("{\"primary\": \"#0ea5e9\", \"secondary\": \"#64748b\"}")
  leadCaptureCompletionMessage String?
  
  // Voice-specific configuration
  voiceGreetingMessage         String?
  voiceCompletionMessage       String?
  voiceEmergencyMessage        String?
  voiceEndCallMessage          String?
  twilioVoice                  String                @default("alice")
  twilioLanguage               String                @default("en-US")
  
  createdAt                    DateTime              @default(now())
  updatedAt                    DateTime              @updatedAt
  
  business                     Business              @relation(fields: [businessId], references: [id], onDelete: Cascade)
  questions                    LeadCaptureQuestion[]

  @@map("agent_configs")
}

model LeadCaptureQuestion {
  id                      String         @id @default(cuid())
  configId                String
  questionText            String
  expectedFormat          ExpectedFormat @default(TEXT)
  order                   Int
  isRequired              Boolean        @default(true)
  mapsToLeadField         String?
  isEssentialForEmergency Boolean        @default(false)
  createdAt               DateTime       @default(now())
  updatedAt               DateTime       @updatedAt
  
  config                  AgentConfig    @relation(fields: [configId], references: [id], onDelete: Cascade)

  @@unique([configId, order])
  @@map("lead_capture_questions")
}

// Additional enums
enum PlanTier {
  FREE
  BASIC
  PRO
}

enum BusinessType {
  REAL_ESTATE
  LAW
  HVAC
  PLUMBING
  OTHER
}

enum ExpectedFormat {
  TEXT
  EMAIL
  PHONE
}

enum LeadStatus {
  NEW
  CONTACTED
  QUALIFIED
  CLOSED
}

enum LeadPriority {
  LOW
  NORMAL
  HIGH
  URGENT
}
```

Run migrations:
```bash
npx prisma migrate dev --name initial_voice_setup
```

### 4. Enhanced Project Structure

```
/src
├── api/                    # Express routes
│   ├── admin.ts           # Enhanced admin API with plan features
│   ├── authMiddleware.ts  # JWT auth with plan validation
│   ├── chatRoutes.ts      # Enhanced chat API
│   ├── voiceRoutes.ts     # NEW: Twilio voice integration
│   └── viewRoutes.ts      # Plan-aware view rendering
├── core/                  # Core business logic
│   ├── aiHandler.ts       # Enhanced with voice optimization
│   └── ragService.ts      # Voice-context aware RAG
├── services/              # Service layer
│   ├── voiceSessionService.ts  # NEW: Redis session management
│   ├── notificationService.ts  # Enhanced with voice notifications
│   ├── openai.ts              # Voice processing integration
│   └── db.ts                   # Database service
├── utils/                 # Helper functions
│   ├── voiceHelpers.ts    # NEW: Voice processing utilities
│   ├── planUtils.ts       # NEW: Plan tier management
│   ├── ssmlHelpers.ts     # NEW: SSML processing
│   └── emergencyDetection.ts # NEW: Emergency detection
├── types/                 # TypeScript definitions
│   ├── voice.ts          # NEW: Voice-related types
│   ├── plans.ts          # NEW: Plan tier types
│   └── emergency.ts      # NEW: Emergency handling types
├── middleware/            # Middleware functions
│   ├── planMiddleware.ts  # NEW: Plan-based access control
│   └── voiceMiddleware.ts # NEW: Voice-specific middleware
├── views/                 # EJS templates
│   ├── agent-settings.ejs # Enhanced with voice config
│   ├── voice-settings.ejs # NEW: PRO voice configuration
│   ├── dashboard.ejs      # Enhanced with analytics
│   └── analytics.ejs      # NEW: Session analytics
└── server.ts             # Enhanced main server
```

### 5. Enhanced Environment Variables (.env)

```env
# Application Configuration
PORT=3000
NODE_ENV=development

# Database Configuration
DATABASE_URL="postgresql://db_user:db_password@localhost:5433/app_db?schema=public"
DIRECT_URL="postgresql://db_user:db_password@localhost:5433/app_db?schema=public"

# Redis Configuration (Session Storage)
REDIS_URL="redis://localhost:6379"
REDIS_PASSWORD=""
REDIS_DB=0
REDIS_TTL=3600

# Security
JWT_SECRET="your-super-secret-jwt-key-with-at-least-32-characters"

# OpenAI Realtime API Integration
OPENAI_API_KEY="sk-your-openai-api-key-here"
OPENAI_REALTIME_MODEL="gpt-4o-realtime-preview-2024-10-01"
OPENAI_EMBEDDING_MODEL="text-embedding-3-small"

# Twilio Media Streams Integration
TWILIO_ACCOUNT_SID="AC_your_twilio_account_sid"
TWILIO_AUTH_TOKEN="your_twilio_auth_token"
TWILIO_WEBHOOK_BASE_URL="https://your-domain.com"
TWILIO_PHONE_NUMBER="+1234567890"
HOSTNAME="your-domain.com" # REQUIRED for WebSocket URL generation

# WebSocket Configuration
WEBSOCKET_PING_INTERVAL=30000
WEBSOCKET_PONG_TIMEOUT=5000
MAX_WEBSOCKET_CONNECTIONS=100

# Performance Configuration
MAX_MEMORY_USAGE_MB=1536
ENABLE_MEMORY_MONITORING=true
ENABLE_VERBOSE_LOGGING=false

# Email Configuration
SMTP_HOST="smtp.ethereal.email"
SMTP_PORT=587
SMTP_USER="your-ethereal-user"
SMTP_PASS="your-ethereal-password"

# Frontend URLs for CORS
APP_PRIMARY_URL="http://localhost:3000"
ADMIN_CUSTOM_DOMAIN_URL="https://app.yourcompany.com"
WIDGET_DEMO_URL="https://demo.yourcompany.com"
WIDGET_TEST_URL="http://127.0.0.1:8080"

# Feature Flags
ENABLE_REALTIME_API=true
ENABLE_WEBSOCKET_FEATURES=true
ENABLE_PLAN_TIERS=true
ENABLE_EMERGENCY_VOICE_CALLS=true
```

## Phase 1: Voice Agent System Implementation

### 1. Implement Voice Routes (`src/api/voiceRoutes.ts`)

```typescript
import express from 'express';
import twilio from 'twilio';
import { VoiceSessionService } from '../services/voiceSessionService.js';
import { AIHandler } from '../core/aiHandler.js';
import { validateTwilioSignature } from '../middleware/voiceMiddleware.js';

const router = express.Router();
const voiceSessionService = new VoiceSessionService();
const aiHandler = new AIHandler();

// Handle incoming calls with Media Streams (Realtime API)
router.post('/incoming', validateTwilioSignature, async (req, res) => {
  try {
    console.log('[VOICE STREAM] Incoming call received:', req.body.CallSid)
    
    // Validate HOSTNAME environment variable
    if (!process.env.HOSTNAME) {
      throw new Error('HOSTNAME environment variable is required for WebSocket streaming')
    }
    
    const callSid = req.body.CallSid
    
    // Create VoiceResponse for bidirectional media streaming
    const response = new twilio.twiml.VoiceResponse()
    const connect = response.connect()
    
    // Create stream connection to WebSocket server
    const streamUrl = `wss://${process.env.HOSTNAME}/`
    const stream = connect.stream({ url: streamUrl })
    
    // Add CallSid as parameter (reliable method)
    stream.parameter({
      name: 'callSid',
      value: callSid
    })
    
    // Add pause to keep call active
    response.pause({ length: 14400 }) // 4 hours max
    
    res.type('text/xml')
    res.send(response.toString())
    
  } catch (error) {
    console.error('[VOICE STREAM] Error:', error)
    // Fallback response
    const response = new twilio.twiml.VoiceResponse()
    response.say({ voice: 'alice' }, 'We are experiencing technical difficulties. Please try again.')
    response.hangup()
    res.type('text/xml')
    res.send(response.toString())
  }
})

// Process speech input asynchronously
router.post('/handle-speech', validateTwilioSignature, async (req, res) => {
  try {
    const speechResult = req.body.SpeechResult
    const callSid = req.body.CallSid
    
    // Background AI processing function
    const processAiResponse = async () => {
      try {
        // Get business and process with AI
        const business = await prisma.business.findFirst({
          where: { twilioPhoneNumber: req.body.To }
        })
        
        const session = await voiceSessionService.getSession(callSid)
        const aiResponse = await aiHandler.processMessage(
          speechResult,
          session.history,
          business.id,
          session.currentFlow,
          callSid
        )
        
        // Generate audio URL
        const audioUrl = await generateAudioUrl(aiResponse.reply)
        
        // Update live call with response
        await twilioClient.calls(callSid).update({
          url: `${process.env.APP_PRIMARY_URL}/api/voice/continue-conversation?audioUrl=${audioUrl}&nextAction=${aiResponse.nextAction}`,
          method: 'POST'
        })
        
      } catch (error) {
        console.error('[AI PROCESSING] Error:', error)
        // Redirect to error handler
        await twilioClient.calls(callSid).update({
          url: `${process.env.APP_PRIMARY_URL}/api/voice/handle-error`,
          method: 'POST'
        })
      }
    }
    
    // Start background processing (non-blocking)
    processAiResponse().catch(console.error)
    
    // Respond immediately with thinking sound
    const twiml = new twilio.twiml.VoiceResponse()
    twiml.play(`${process.env.APP_PRIMARY_URL}/sounds/thinking-sound-fx.mp3`)
    twiml.pause({ length: 8 })
    
    res.type('text/xml')
    res.send(twiml.toString())
    
  } catch (error) {
    console.error('[HANDLE SPEECH] Error:', error)
    // Error response
    const twiml = new twilio.twiml.VoiceResponse()
    twiml.say({ voice: 'alice' }, 'I apologize, but I encountered an error.')
    twiml.hangup()
    res.type('text/xml')
    res.send(twiml.toString())
  }
})

export default router;
```

### 2. Implement Realtime Agent Service (`src/services/realtimeAgentService.ts`)

```typescript
import WebSocket from 'ws';
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

interface ConnectionState {
  isTwilioReady: boolean;
  isAiReady: boolean;
  streamSid: string | null;
  audioQueue: string[];
}

export class RealtimeAgentService {
  private twilioWs: WebSocket | null = null;
  private openAiWs: WebSocket | null = null;
  private callSid: string = '';
  private state: ConnectionState;
  public onCallSidReceived?: (callSid: string) => void;

  constructor(twilioWs: WebSocket) {
    this.twilioWs = twilioWs;
    
    this.state = {
      isTwilioReady: false,
      isAiReady: false,
      streamSid: null,
      audioQueue: [],
    };

    this.setupTwilioListeners();
  }

  private handleTwilioMessage(message: WebSocket.Data) {
    const msg = JSON.parse(message.toString());
    
    if (msg.event === "start") {
      // Extract CallSid from start message (reliable method)
      this.callSid = msg.start.callSid;
      
      if (this.onCallSidReceived) {
        this.onCallSidReceived(this.callSid);
      }
      
      this.state.streamSid = msg.start.streamSid;
      this.state.isTwilioReady = true;
      
      // Connect to OpenAI
      this.connectToOpenAI();
    } else if (msg.event === "media") {
      // Forward audio to OpenAI
      if (this.openAiWs?.readyState === WebSocket.OPEN) {
        this.openAiWs.send(JSON.stringify({
          type: 'input_audio_buffer.append',
          audio: msg.media.payload
        }));
      }
    }
  }

  private async configureOpenAiSession() {
    // Get business-specific configuration
    const business = await this.getBusinessConfig();
    
    const sessionConfig = {
      type: 'session.update',
      session: {
        modalities: ['text', 'audio'],
        instructions: business.instructions,
        voice: 'alloy',
        input_audio_format: 'g711_ulaw',
        output_audio_format: 'g711_ulaw',
        turn_detection: {
          type: 'server_vad',
          threshold: 0.5,
          prefix_padding_ms: 500,
          silence_duration_ms: 2500 // 2.5 seconds
        }
      }
    };

    this.openAiWs.send(JSON.stringify(sessionConfig));
  }

  // Handle audio queue for buffering
  private processAudioQueue() {
    if (!this.state.isTwilioReady || !this.state.streamSid) return;

    while (this.state.audioQueue.length > 0) {
      const audioChunk = this.state.audioQueue.shift();
      if (audioChunk && this.twilioWs?.readyState === WebSocket.OPEN) {
        this.twilioWs.send(JSON.stringify({
          event: "media",
          streamSid: this.state.streamSid,
          media: { payload: audioChunk },
        }));
      }
    }
  }
}
```

### 3. Implement WebSocket Server (`src/services/websocketServer.ts`)

```typescript
import WebSocket, { WebSocketServer } from 'ws';
import { Server as HttpServer } from 'http';
import { RealtimeAgentService } from './realtimeAgentService';

export class TwilioWebSocketServer {
  private wss: WebSocketServer;
  private activeConnections = new Map<string, RealtimeAgentService>();

  constructor(server: HttpServer) {
    this.wss = new WebSocketServer({ server });
    this.setupConnectionHandler();
  }

  private setupConnectionHandler() {
    this.wss.on('connection', (ws: WebSocket) => {
      const agent = new RealtimeAgentService(ws);
      
      // Temporary connection ID until CallSid is received
      const tempId = `temp_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
      this.activeConnections.set(tempId, agent);
      
      // Listen for CallSid
      agent.onCallSidReceived = (callSid: string) => {
        // Move from temp ID to actual CallSid
        this.activeConnections.delete(tempId);
        this.activeConnections.set(callSid, agent);
      };

      ws.on('close', () => {
        const callSid = agent.getCallSid();
        const key = callSid || tempId;
        this.activeConnections.get(key)?.cleanup();
        this.activeConnections.delete(key);
      });
    });
  }
}
```

## Phase 2: Plan Tier System Implementation

### 1. Implement Plan Utilities (`src/utils/planUtils.ts`)

```typescript
import { PlanTier } from '@prisma/client';

export class PlanManager {
  static readonly PLAN_HIERARCHY = {
    FREE: 0,
    BASIC: 1,
    PRO: 2
  };

  static isPlanSufficient(userPlan: PlanTier, requiredPlan: PlanTier): boolean {
    return this.PLAN_HIERARCHY[userPlan] >= this.PLAN_HIERARCHY[requiredPlan];
  }

  static getAvailableFeatures(planTier: PlanTier): string[] {
    switch (planTier) {
      case 'FREE':
        return [
          'chat_widget',
          'basic_faq',
          'limited_questions', // Max 5 questions
          'email_notifications',
          'basic_analytics'
        ];
      
      case 'BASIC':
        return [
          'chat_widget',
          'advanced_faq',
          'unlimited_questions',
          'priority_email_notifications',
          'enhanced_analytics',
          'knowledge_base_management'
        ];
      
      case 'PRO':
        return [
          'all_basic_features',
          'voice_agent',
          'premium_voices',
          'emergency_voice_calls',
          'advanced_analytics',
          'session_management',
          'branding_removal',
          'voice_configuration',
          'priority_support'
        ];
      
      default:
        return [];
    }
  }

  static getQuestionLimit(planTier: PlanTier): number {
    switch (planTier) {
      case 'FREE':
        return 5;
      case 'BASIC':
      case 'PRO':
        return -1; // Unlimited
      default:
        return 0;
    }
  }

  static canAccessVoiceFeatures(planTier: PlanTier): boolean {
    return planTier === 'PRO';
  }

  static shouldShowBranding(planTier: PlanTier): boolean {
    return planTier !== 'PRO';
  }
}
```

### 2. Implement Plan Middleware (`src/middleware/planMiddleware.ts`)

```typescript
import { Request, Response, NextFunction } from 'express';
import { PlanTier } from '@prisma/client';
import { PlanManager } from '../utils/planUtils.js';

interface AuthenticatedRequest extends Request {
  user: {
    id: string;
    businessId: string;
    business: {
      planTier: PlanTier;
    };
  };
}

export const requirePlan = (requiredPlan: PlanTier) => {
  return (req: AuthenticatedRequest, res: Response, next: NextFunction) => {
    const userPlan = req.user.business.planTier;
    
    if (!PlanManager.isPlanSufficient(userPlan, requiredPlan)) {
      return res.status(403).json({
        error: 'Plan upgrade required',
        currentPlan: userPlan,
        requiredPlan: requiredPlan,
        upgradeUrl: '/admin/upgrade'
      });
    }
    
    next();
  };
};

export const requireVoiceFeatures = (req: AuthenticatedRequest, res: Response, next: NextFunction) => {
  const userPlan = req.user.business.planTier;
  
  if (!PlanManager.canAccessVoiceFeatures(userPlan)) {
    return res.status(403).json({
      error: 'Voice features require PRO plan',
      currentPlan: userPlan,
      upgradeUrl: '/admin/upgrade'
    });
  }
  
  next();
};

export const addPlanContext = (req: AuthenticatedRequest, res: Response, next: NextFunction) => {
  const planTier = req.user.business.planTier;
  
  res.locals.planTier = planTier;
  res.locals.availableFeatures = PlanManager.getAvailableFeatures(planTier);
  res.locals.canAccessVoice = PlanManager.canAccessVoiceFeatures(planTier);
  res.locals.showBranding = PlanManager.shouldShowBranding(planTier);
  res.locals.questionLimit = PlanManager.getQuestionLimit(planTier);
  
  next();
};
```

## Phase 3: Enhanced Emergency System

### 1. Implement Emergency Detection (`src/utils/emergencyDetection.ts`)

```typescript
export interface EmergencyKeywords {
  URGENT: string[];
  HIGH: string[];
  NORMAL: string[];
}

export class EmergencyDetectionEngine {
  private static readonly EMERGENCY_KEYWORDS: EmergencyKeywords = {
    URGENT: [
      'flooding', 'flood', 'burst pipe', 'pipe burst',
      'electrical fire', 'fire', 'gas leak', 'no heat',
      'no hot water', 'carbon monoxide', 'sparking',
      'smoke', 'burning smell', 'water everywhere'
    ],
    HIGH: [
      'leak', 'leaking', 'clogged', 'blocked',
      'strange smell', 'unusual noise', 'not working',
      'broken', 'damaged', 'malfunctioning'
    ],
    NORMAL: [
      'maintenance', 'quote', 'estimate', 'schedule',
      'appointment', 'inspection', 'consultation',
      'service', 'repair', 'installation'
    ]
  };

  static detectEmergency(message: string, channel: 'CHAT' | 'VOICE' = 'CHAT'): EmergencyLevel {
    const lowercaseMessage = message.toLowerCase();
    
    // Check for urgent keywords first
    for (const keyword of this.EMERGENCY_KEYWORDS.URGENT) {
      if (lowercaseMessage.includes(keyword)) {
        return 'URGENT';
      }
    }
    
    // Check for high priority keywords
    for (const keyword of this.EMERGENCY_KEYWORDS.HIGH) {
      if (lowercaseMessage.includes(keyword)) {
        return 'HIGH';
      }
    }
    
    // Check for normal service keywords
    for (const keyword of this.EMERGENCY_KEYWORDS.NORMAL) {
      if (lowercaseMessage.includes(keyword)) {
        return 'NORMAL';
      }
    }
    
    // Default to normal if no keywords match
    return 'NORMAL';
  }

  static async handleEmergency(
    emergencyLevel: EmergencyLevel,
    leadData: any,
    businessId: string,
    channel: 'CHAT' | 'VOICE'
  ): Promise<void> {
    // Create priority lead with emergency flag
    const priorityLead = await createEmergencyLead(leadData, emergencyLevel, businessId);
    
    // Send appropriate notifications based on plan tier
    if (emergencyLevel === 'URGENT') {
      await sendEmergencyNotifications(priorityLead, channel);
    }
    
    // Update session analytics
    await updateSessionAnalytics(priorityLead.sessionId, {
      emergencyDetected: true,
      emergencyLevel: emergencyLevel
    });
  }

  static getEssentialQuestions(questions: any[], emergencyLevel: EmergencyLevel): any[] {
    if (emergencyLevel === 'URGENT') {
      return questions.filter(q => q.isEssentialForEmergency);
    }
    return questions;
  }
}

export type EmergencyLevel = 'LOW' | 'NORMAL' | 'HIGH' | 'URGENT';
```

## Phase 4: Enhanced Admin Interface with Plan-Aware Features

### 1. Update Admin Routes (`src/api/admin.ts`)

Add plan-aware endpoints:

```typescript
// Voice configuration endpoint (PRO only)
router.post('/config/voice', authMiddleware, requireVoiceFeatures, async (req, res) => {
  try {
    const { businessId } = req.user;
    const voiceConfig = req.body;
    
    // Validate voice configuration
    if (!validateVoiceConfiguration(voiceConfig)) {
      return res.status(400).json({ error: 'Invalid voice configuration' });
    }
    
    const updatedConfig = await prisma.agentConfig.update({
      where: { businessId },
      data: {
        voiceGreetingMessage: voiceConfig.voiceGreetingMessage,
        voiceCompletionMessage: voiceConfig.voiceCompletionMessage,
        voiceEmergencyMessage: voiceConfig.voiceEmergencyMessage,
        voiceEndCallMessage: voiceConfig.voiceEndCallMessage,
        twilioVoice: voiceConfig.twilioVoice,
        twilioLanguage: voiceConfig.twilioLanguage
      }
    });
    
    res.json(updatedConfig);
  } catch (error) {
    console.error('Voice config update error:', error);
    res.status(500).json({ error: 'Failed to update voice configuration' });
  }
});

// Analytics endpoint (PRO only)
router.get('/analytics', authMiddleware, requirePlan('PRO'), async (req, res) => {
  try {
    const { businessId } = req.user;
    const analytics = await getBusinessAnalytics(businessId);
    res.json(analytics);
  } catch (error) {
    console.error('Analytics error:', error);
    res.status(500).json({ error: 'Failed to fetch analytics' });
  }
});
```

### 2. Enhanced View Routes with Plan Context (`src/api/viewRoutes.ts`)

```typescript
// Agent settings with plan-aware features
router.get('/settings', authMiddleware, addPlanContext, async (req, res) => {
  try {
    const { businessId } = req.user;
    const config = await prisma.agentConfig.findUnique({
      where: { businessId },
      include: { questions: true }
    });
    
    res.render('agent-settings', {
      config,
      planTier: res.locals.planTier,
      canAccessVoice: res.locals.canAccessVoice,
      showBranding: res.locals.showBranding,
      availableFeatures: res.locals.availableFeatures
    });
  } catch (error) {
    console.error('Settings page error:', error);
    res.status(500).send('Error loading settings');
  }
});

// Voice settings (PRO only)
router.get('/voice-settings', authMiddleware, requireVoiceFeatures, async (req, res) => {
  try {
    const { businessId } = req.user;
    const config = await prisma.agentConfig.findUnique({
      where: { businessId }
    });
    
    res.render('voice-settings', {
      config,
      voiceOptions: getAvailableVoiceOptions(res.locals.planTier),
      languageOptions: getAvailableLanguageOptions()
    });
  } catch (error) {
    console.error('Voice settings page error:', error);
    res.status(500).send('Error loading voice settings');
  }
});
```

## Phase 5: Enhanced Deployment with Docker Compose

### 1. Update docker-compose.yml

```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://db_user:db_password@db:5432/app_db
      - REDIS_URL=redis://redis:6379
      - NODE_ENV=development
    depends_on:
      - db
      - redis
    volumes:
      - .:/app
      - /app/node_modules
    command: npm run dev

  db:
    image: pgvector/pgvector:pg15
    environment:
      POSTGRES_DB: app_db
      POSTGRES_USER: db_user
      POSTGRES_PASSWORD: db_password
    ports:
      - "5433:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru

volumes:
  postgres_data:
  redis_data:
```

### 2. Enhanced Dockerfile

```dockerfile
FROM node:20-alpine

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci --only=production

# Copy source code
COPY . .

# Generate Prisma client
RUN npx prisma generate

# Build application
RUN npm run build

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

EXPOSE 3000

CMD ["npm", "start"]
```

## Phase 6: Testing & Validation

### 1. Voice System Testing

```bash
# Test Twilio webhook connectivity
curl -X POST http://localhost:3000/api/voice/incoming \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "CallSid=test123&From=+1234567890&To=+1987654321"

# Test voice session creation
npm run test:voice
```

### 2. Plan Tier Testing

```bash
# Test plan-restricted endpoints
curl -H "Authorization: Bearer $JWT_TOKEN" \
     http://localhost:3000/api/admin/config/voice

# Run plan tier tests
npm run test:plans
```

### 3. Emergency Detection Testing

```bash
# Test emergency keyword detection
curl -X POST http://localhost:3000/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"My basement is flooding", "businessId":"test-business"}'
```

## Phase 7: Production Deployment

### 1. Production Environment Setup

```bash
# Set production environment variables
export NODE_ENV=production
export DATABASE_URL="postgresql://prod_user:prod_pass@prod_host:5432/prod_db"
export REDIS_URL="redis://prod_redis:6379"
export TWILIO_WEBHOOK_BASE_URL="https://your-production-domain.com"
export HOSTNAME="your-production-domain.com" # REQUIRED for WebSocket

# Run production migrations
npx prisma migrate deploy

# Start production server
npm start
```

### 2. Twilio Webhook Configuration

Configure Twilio webhooks to point to your production endpoints:
- Incoming calls: `https://your-domain.com/api/voice/incoming`
- Status callback: `https://your-domain.com/api/voice/status`

### 3. WebSocket Configuration

Ensure your production environment supports WebSocket connections:
- SSL certificates configured for WSS
- Firewall rules allow WebSocket traffic
- Load balancer supports WebSocket sticky sessions

### 4. Health Monitoring

```bash
# Check system health
curl https://your-domain.com/health | jq

# Monitor WebSocket connections
curl https://your-domain.com/health | jq '.webSocket'

# Monitor Redis sessions
curl https://your-domain.com/health | jq '.sessions'
```

## Success Metrics & Monitoring

- **Voice System**: Call completion rates, transcription accuracy, response times, async processing latency
- **WebSocket Performance**: Connection stability, audio quality metrics, CallSid association success rate
- **Plan Tiers**: Feature adoption rates, upgrade conversion, plan-specific usage
- **Emergency System**: Emergency detection accuracy, response times, false positive rates
- **Overall Performance**: Session analytics, lead conversion rates, system uptime

This comprehensive implementation guide now covers all aspects of the enhanced voice-enabled, plan-tier based AI agent platform with asynchronous processing, improved connection management, and sophisticated emergency handling capabilities. 