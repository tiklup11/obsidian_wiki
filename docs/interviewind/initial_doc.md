# Technical Design Document: AI Interview Video Call Application (Revised)

[goback](./)
[contracts](./contracts.md)
[deployment](./deploy_doc.md)
[extras](./extras.md)

## 1. Introduction

This document outlines the technical design for an AI-powered video interview application. The application will allow candidates to participate in video interviews where an AI agent asks questions and evaluates responses. Human interviewers can also join the session. The system leverages the Gemini API for intelligent question generation and interaction, React with TypeScript for the frontend, and Go for the backend, following Domain-Driven Design (DDD) principles.

## 2. Architecture Overview

We will adopt a **Microservices Architecture** to promote separation of concerns, scalability, and maintainability. The primary components will be:

1.  **Frontend (React SPA):** The user interface handling video rendering, user interactions, and communication with the backend.
2.  **API Gateway:** A single entry point for all frontend requests, routing them to the appropriate backend services. Handles cross-cutting concerns like authentication and rate limiting.
3.  **Interview Service (Go):** Manages the lifecycle and state of interview sessions, participants, and permissions.
4.  **Signaling Service (Go):** Handles WebRTC signaling for establishing peer-to-peer connections via the SFU.
5.  **AI Service (Go):** Orchestrates the AI interviewer's interaction, including question generation (via Gemini), Speech-to-Text (STT), and Text-to-Speech (TTS).
6.  **Media Server (SFU):** A Selective Forwarding Unit (SFU) (e.g., using Pion or a managed service like LiveKit/Mediasoup) to efficiently handle multi-party video/audio streams.

```
graph TD
    User[Candidate/Interviewer Browser] -- HTTPS/WSS --> APIGateway[API Gateway]
    APIGateway -- REST --> InterviewService[Interview Service (Go)]
    APIGateway -- WebSocket --> SignalingService[Signaling Service (Go)]
    APIGateway -- REST --> AIService[AI Service (Go)]

    InterviewService -- CRUD --> Database[(Database - PostgreSQL)]
    AIService -- API Call --> Gemini[Gemini API]
    AIService -- API Call --> STT[Speech-to-Text Service]
    AIService -- API Call --> TTS[Text-to-Speech Service]

    User -- WebRTC --> SFU[Media Server (SFU)]
    SignalingService -- Control/API --> SFU[Manages SFU Rooms/Permissions]
```

_Diagram revised to clarify Signaling Service control over SFU._

## 3. Domain Model (DDD)

We identify the following core domains and aggregates:

- **Interview Domain:**
  - **`InterviewSession`** (Aggregate Root): Represents a single interview instance.
    - `SessionID` (Entity ID)
    - `Status` (Value Object: Scheduled, InProgress, Completed, Cancelled)
    - `ScheduledTime` (Value Object)
    - `Participants` (List of `Participant` Entities)
    - `ConversationHistory` (List of `Interaction` Value Objects)
    - `JobDescription` (Value Object)
    - `AccessPermissions` (Value Object Map: e.g., UserID -> Role)
  - **`Participant`** (Entity): Represents a user (Candidate, Human Interviewer, AI) connected to the session.
    - `ParticipantID` (Entity ID - unique within a session)
    - `UserID` (Value Object - links to a global user account)
    - `Role` (Value Object: Candidate, Interviewer, AI)
    - `ConnectionStatus` (Value Object: Joining, Connected, Disconnected, Failed)
    - `SignalingID` (Value Object - e.g., WebSocket connection ID)
  - **`Interaction`** (Value Object): Records a turn in the conversation.
    - `Timestamp`
    - `SpeakerParticipantID` (`ParticipantID`)
    - `Type` (Question, Answer, Comment, SystemMessage)
    - `Content` (Text)
    - `AudioReference` (Optional Value Object - link to stored audio)
- **AI Domain:**
  - **`AIConversation`** (Aggregate Root): Manages the state of the AI's interaction within a specific `InterviewSession`.
    - `SessionID` (Entity ID - maps 1:1 to InterviewSession)
    - `CurrentState` (Value Object: Idle, Listening, Processing, Speaking)
    - `CurrentTopic` (Value Object)
    - `EvaluationContext` (Value Object: Accumulated understanding, scores, notes)
    - `PendingQuestion` (Value Object - stores the next question before TTS/delivery)
- **User Domain (Potentially separate service or part of Interview Service):**
  - **`User`** (Aggregate Root): Represents a registered user account.
    - `UserID` (Entity ID)
    - `Name`
    - `Email`
    - `AuthenticationDetails`

## 4. Authentication and Authorization

- **Authentication:** Users authenticate before accessing the application (e.g., via JWT tokens issued by a dedicated Auth service or integrated within the User domain). The API Gateway validates the token on incoming requests.
- **Authorization:**
  - The Interview Service checks if the authenticated `UserID` has permission (based on `InterviewSession.AccessPermissions`) to join or manage a specific `SessionID`.
  - Roles (`Candidate`, `Interviewer`) dictate allowed actions within a session (e.g., only AI/Interviewer can trigger certain AI actions).
  - Signaling Service authorizes WebSocket connections based on a temporary token or session linkage provided after successful REST API join.

## 5. Components Deep Dive

- **Frontend (React TSX):**
  - **UI Components:** `VideoGrid`, `ParticipantVideo`, `AIChatDisplay`/`Transcript`, `Controls` (Mute/Unmute, Video On/Off, Leave, Start/Stop Recording - future).
  - **State Management:** Zustand (preferred for simplicity) or Redux Toolkit. Manages session state, participants, connection status, media device settings, AI messages/status.
  - **WebRTC Client:** Handles interaction with the SFU. Uses `getUserMedia` for media, negotiates connections via Signaling Service, manages `RTCPeerConnection` for SFU transport.
  - **API Client:** Axios (with interceptors for auth tokens). WebSocket client for signaling.
- **API Gateway (Go - e.g., Gin/Echo):**
  - Validates JWT tokens.
  - Routes requests based on path prefixes.
  - Aggregates responses if needed (though less common in pure microservices).
  - Handles TLS termination.
- **Interview Service (Go):**
  - **Repositories:** `InterviewSessionRepository`, `UserRepository` (if User domain included).
  - **Application Services:** `CreateInterview`, `AuthorizeJoin`, `GetSessionDetails`, `AddParticipant`, `UpdateParticipantStatus`, `RecordInteraction`.
  - **Domain Logic:** Encapsulated within aggregates.
  - **Database:** PostgreSQL chosen for its relational structure and ACID compliance. Stores sessions, participant details (linked to users), permissions, conversation history.
- **Signaling Service (Go):**
  - Uses WebSockets (Gorilla WebSocket).
  - Manages signaling "rooms" corresponding to `SessionID`.
  - Authenticates WebSocket connections (e.g., using a short-lived token from Interview Service).
  - Relays WebRTC SDP Offers/Answers and ICE Candidates between clients and the SFU.
  - **SFU Coordination:** Communicates with the SFU (via its API or control protocol) to manage media rooms, participant permissions within the SFU, and stream forwarding rules.
  - Broadcasts application-level events (join, leave, speaking status) to clients in the same room.
- **AI Service (Go):**
  - **State Machine:** Manages `AIConversation.CurrentState` (Idle -> Listening -> Processing -> Speaking -> Idle/Listening). Transitions triggered by events (e.g., candidate stops speaking, Gemini response received).
  - Listens for events indicating candidate speech completion (e.g., message from Frontend via Gateway or direct event bus).
  - Constructs prompts for Gemini API. Handles errors and retries for external API calls.
  - Interfaces with STT/TTS services.
  - Updates `InterviewSession.ConversationHistory` via Interview Service API or event bus.
  - Communicates AI status (speaking, processing) via Signaling Service for UI updates.
- **Media Server (SFU - e.g., Pion/LiveKit):**
  - Requires separate deployment and management.
  - Receives encrypted media streams (DTLS/SRTP).
  - Forwards streams based on Signaling Service/application logic instructions.

## 6. Key Workflows & Sequence Diagrams (Revised & Extended)

**a) Participant Joins Interview (Revised)**

```
sequenceDiagram
    participant FE as Frontend
    participant GW as API Gateway
    participant IS as Interview Service
    participant SS as Signaling Service
    participant SFU as Media Server (SFU)

    FE->>+GW: POST /api/interview/{sessionId}/join (Authorization: Bearer <token>)
    GW->>GW: Validate Token
    GW->>+IS: AuthorizeJoin(sessionId, userId from token, role?)
    IS->>IS: Check User Permissions for SessionID
    IS-->>-GW: Success, SignalingToken, WebSocketURL, SFU Info
    GW-->>-FE: Success, SignalingToken, WebSocketURL, SFU Info

    FE->>+SS: WebSocket Connect (/ws/signal?token=<SignalingToken>)
    SS->>SS: Validate SignalingToken, Associate Conn with SessionID/UserID
    SS-->>-FE: WebSocket Connected

    FE->>FE: Get Local Media (Camera/Mic)
    FE->>SS: Send Message: {"type": "request_media_config", "userId": ...}
    SS->>SFU: API Call: Prepare SFU endpoint for userId in sessionId
    SFU-->>SS: SFU Connection Details (e.g., URLs, credentials)
    SS->>FE: Send Message: {"type": "media_config", "sfuDetails": ...} // Info needed to connect to SFU

    Note over FE, SS: WebRTC Handshake for SFU connection, mediated by Signaling Service
    FE->>FE: Create SFU PeerConnection
    FE->>SS: Send Message: {"type": "offer", "target": "sfu", "sdp": ...}
    SS->>SFU: API Call: Process Offer for userId
    SFU-->>SS: Answer SDP
    SS->>FE: Send Message: {"type": "answer", "from": "sfu", "sdp": ...}
    FE->>FE: Set Remote Description (Answer)

    Note over FE, SS: ICE Candidate Exchange (mediated by SS, target SFU)
    FE->>SS: Send Message: {"type": "ice_candidate", "target": "sfu", "candidate": ...}
    SS->>SFU: API Call: Add ICE Candidate for userId
    SFU-->>SS: (If SFU generates candidates) ICE Candidate
    SS->>FE: Send Message: {"type": "ice_candidate", "from": "sfu", "candidate": ...}

    FE-->>SFU: Establish Encrypted WebRTC Media Transport (DTLS/SRTP)
    FE->>SS: Send Message: {"type": "connection_established"}
    SS->>IS: API Call: UpdateParticipantStatus(sessionId, userId, "Connected")
    SS->>FE: Send Message: {"type": "session_ready", "participants": [...]} // Includes self and others
    SS-->>FE: Broadcast to others: {"type": "participant_joined", "participant": ...}


```

_Diagram revised for clarity on Auth, Token usage, and explicit SFU API calls._

**b) AI Asks Question (Revised)**

```
sequenceDiagram
    participant FE as Frontend
    participant GW as API Gateway
    participant AI as AI Service
    participant Gemini as Gemini API
    participant TTS as TTS Service
    participant SS as Signaling Service
    participant IS as Interview Service

    Note over AI: Triggered by event (e.g., candidate finished speaking)
    AI->>AI: Set State: Processing
    AI->>SS: Broadcast via SS: {"type": "ai_status", "sessionId": ..., "status": "Processing"}

    AI->>+IS: GET /api/interview/{sessionId}/context (history, job desc)
    IS-->>-AI: Session Context Data

    AI->>+Gemini: Generate Question (context: history, job desc, last response)
    alt Success
        Gemini-->>-AI: Question Text
        AI->>AI: Store PendingQuestion Text
        AI->>+TTS: Generate Audio (text: question text)
        TTS-->>-AI: Audio Data/URL
        AI->>AI: Store PendingQuestion AudioRef
        AI->>AI: Set State: Speaking
        AI->>SS: Broadcast via SS: {"type": "ai_status", "sessionId": ..., "status": "Speaking"}
        AI->>SS: Broadcast via SS: {"type": "ai_message", "sessionId": ..., "text": ..., "audioRef": ...}

        FE->>FE: Receive ai_message, Display Text
        FE->>FE: Play Audio (from audioRef)
        FE-->>FE: On Audio End
        AI->>AI: Set State: Listening / Idle
        AI->>SS: Broadcast via SS: {"type": "ai_status", "sessionId": ..., "status": "Listening"}

    else Gemini/TTS Failure
        Gemini-->>-AI: Error Response
        AI->>AI: Log Error
        AI->>AI: Set State: Idle / Error
        AI->>SS: Broadcast via SS: {"type": "ai_status", "sessionId": ..., "status": "Error"}
        AI->>SS: Broadcast via SS: {"type": "system_message", "sessionId": ..., "text": "Sorry, I encountered an issue. Please wait."} // Inform users
        Note over AI: Implement retry logic or alternative flow
    end


```

_Diagram revised to include AI state updates broadcast via Signaling, context fetching, and basic error handling._

**c) Candidate Responds (Revised)**

```
sequenceDiagram
    participant FE as Frontend
    participant STT as STT Service
    participant GW as API Gateway
    participant AI as AI Service
    participant SS as Signaling Service
    participant IS as Interview Service

    Note over FE: User unmutes and speaks
    FE->>SS: Send Message: {"type": "speaking_status", "userId": ..., "speaking": true}
    SS-->>FE: Broadcast to others: {"type": "participant_speaking", "userId": ..., "speaking": true}

    FE->>FE: Start Audio Capture & Processing (e.g., VAD - Voice Activity Detection)

    FE->>+STT: Stream Audio Data / Send Audio Chunk
    STT-->>-FE: Real-time Transcription / Final Transcript

    FE->>FE: Display real-time transcript (optional)

    Note over FE: Detect end of speech (VAD or timeout)
    FE->>SS: Send Message: {"type": "speaking_status", "userId": ..., "speaking": false}
    SS-->>FE: Broadcast to others: {"type": "participant_speaking", "userId": ..., "speaking": false}

    FE->>+GW: POST /api/ai/{sessionId}/process_response (transcript: final_transcript, userId: ...)
    GW->>+AI: ProcessResponse(sessionId, transcript, userId)
    AI->>AI: Update AIConversation context
    AI->>+IS: POST /api/interview/{sessionId}/interaction (speakerId: userId, type: 'Answer', content: transcript)
    IS-->>-AI: Interaction Saved Ack
    AI-->>-GW: Ack / Success
    GW-->>-FE: Ack / Success

    Note over AI: AI Service now likely triggers the "AI Asks Question" workflow (State -> Processing)


```

_Diagram revised to include speaking status broadcasts and saving interaction to Interview Service._

**d) Participant Leaves**

```
sequenceDiagram
    participant FE as Frontend
    participant GW as API Gateway
    participant IS as Interview Service
    participant SS as Signaling Service
    participant SFU as Media Server (SFU)

    FE->>FE: User clicks "Leave" button

    FE->>SS: Send Message: {"type": "leave_request", "userId": ...}
    SS->>SFU: API Call: Disconnect participant userId from sessionId
    SFU-->>SS: Acknowledgment

    SS->>IS: API Call: UpdateParticipantStatus(sessionId, userId, "Disconnected")
    IS-->>SS: Acknowledgment

    SS-->>FE: Broadcast to others: {"type": "participant_left", "userId": ...}
    SS->>FE: Send Message: {"type": "leave_ack"} // Acknowledge to leaving user
    FE->>SS: Close WebSocket Connection
    FE->>FE: Cleanup UI, redirect user


```

**e) Error Handling Example: Gemini API Failure**

```
sequenceDiagram
    participant AI as AI Service
    participant Gemini as Gemini API
    participant SS as Signaling Service
    participant IS as Interview Service

    Note over AI: In "Processing" state, attempting to generate question
    AI->>+Gemini: Generate Question (context...)
    Gemini-->>-AI: Error (e.g., 500 Server Error, Rate Limit, Invalid Input)

    AI->>AI: Log the specific error from Gemini
    alt Retry Logic (if applicable)
        AI->>AI: Wait (exponential backoff)
        AI->>+Gemini: Retry Generate Question
        Gemini-->>-AI: Success or Error again
        Note over AI: If retries exhausted, proceed to failure handling
    end

    Note over AI: Handle Failure Gracefully
    AI->>AI: Set State: Idle / Error
    AI->>SS: Broadcast via SS: {"type": "ai_status", "sessionId": ..., "status": "Error"}
    AI->>SS: Broadcast via SS: {"type": "system_message", "sessionId": ...,

```
