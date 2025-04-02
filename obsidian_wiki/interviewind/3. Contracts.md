# AI Interview Application - Data Contracts (Microservices/gRPC)

This document defines the data contracts for the AI Interview Application, based on the revised Technical Design Document (`ai_interview_tdd_v1`) which specifies a microservices architecture with gRPC for inter-service communication.

**Note:** REST and WebSocket contracts define the **external communication** between the Frontend and the API Gateway/Signaling Service. gRPC contracts define the **internal communication** between backend microservices.

## 1. Common Data Structures (Go)

These structs define basic types used across different contract types (REST, WebSocket). The gRPC contracts will use corresponding Protocol Buffer message types defined in `.proto` files.

```go
package contracts // Example package name

import "time"

// Role defines the possible roles of a participant in an interview.
type Role string
const ( RoleCandidate Role = "candidate"; RoleInterviewer Role = "interviewer"; RoleAI Role = "ai" )
// SessionStatus defines the possible states of an interview session.
type SessionStatus string
const ( StatusScheduled SessionStatus = "scheduled"; StatusInProgress SessionStatus = "in_progress"; StatusCompleted SessionStatus = "completed"; StatusCancelled SessionStatus = "cancelled" )
// ParticipantStatus defines the connection status of a participant.
type ParticipantStatus string
const ( StatusJoining ParticipantStatus = "joining"; StatusConnected ParticipantStatus = "connected"; StatusDisconnected ParticipantStatus = "disconnected"; StatusFailed ParticipantStatus = "failed" )
// InteractionType defines the type of a conversation interaction.
type InteractionType string
const ( InteractionQuestion InteractionType = "question"; InteractionAnswer InteractionType = "answer"; InteractionComment InteractionType = "comment"; InteractionSystem InteractionType = "system_message" )
// AIStatus defines the possible states of the AI agent.
type AIStatus string
const ( AIStatusIdle AIStatus = "idle"; AIStatusListening AIStatus = "listening"; AIStatusProcessing AIStatus = "processing"; AIStatusSpeaking AIStatus = "speaking"; AIStatusError AIStatus = "error" )

// ParticipantInfo represents basic information about a participant for external APIs/WS.
type ParticipantInfo struct {
    ParticipantID string            `json:"participantId"` // Unique ID within the session
    UserID        string            `json:"userId"`        // Global User ID
    Role          Role              `json:"role"`
    Name          string            `json:"name"`          // User's display name (fetched via UserID)
    Status        ParticipantStatus `json:"status"`
}
// InteractionInfo represents a single interaction for external APIs/WS.
type InteractionInfo struct {
    InteractionID        string          `json:"interactionId"`
    Timestamp            time.Time       `json:"timestamp"`
    SpeakerParticipantID string          `json:"speakerParticipantId"`
    SpeakerUserID        string          `json:"speakerUserId"` // Denormalized for convenience
    SpeakerRole          Role            `json:"speakerRole"`   // Denormalized for convenience
    Type                 InteractionType `json:"type"`
    Content              string          `json:"content"` // Text content
    AudioReference       *string         `json:"audioReference,omitempty"` // Optional URL/path to audio
}
// ErrorResponse is a standard structure for API error responses.
type ErrorResponse struct {
	Error   string `json:"error"`
	Details string `json:"details,omitempty"`
}
````

## 2. REST API Contracts (External: Frontend <-> API Gateway)

These define the request and response bodies for the REST endpoints managed by the API Gateway, primarily routing to the Interview and AI services. All endpoints require `Authorization: Bearer <jwt_token>` header unless otherwise specified.

### 2.1 Interview Service API (`/api/interview/*`) via Gateway

**`a) POST /api/interview - Create Interview Session`**

- **Description:** Creates a new interview session.
    
- **Request Body:** `CreateInterviewRequest`
    
    ```
    type CreateInterviewRequest struct {
        ScheduledTime   time.Time       `json:"scheduledTime"`
        JobDescription  string          `json:"jobDescription"`
        ParticipantPermissions map[string]Role `json:"participantPermissions"` // Map UserID -> Role
    }
    ```
    
- **Response Body (201 Created):** `CreateInterviewResponse`
    
    ```
    type CreateInterviewResponse struct {
        SessionID     string        `json:"sessionId"`
        Status        SessionStatus `json:"status"`
        ScheduledTime time.Time     `json:"scheduledTime"`
    }
    ```
    

**`b) GET /api/interview/{sessionId} - Get Interview Session Details`**

- **Description:** Retrieves details for a specific interview session.
    
- **Path Parameter:** `sessionId` (string)
    
- **Request Body:** None
    
- **Response Body (200 OK):** `GetInterviewResponse`
    
    ```
    type GetInterviewResponse struct {
        SessionID           string             `json:"sessionId"`
        Status              SessionStatus      `json:"status"`
        ScheduledTime       time.Time          `json:"scheduledTime"`
        JobDescription      string             `json:"jobDescription"`
        Participants        []ParticipantInfo  `json:"participants"`
        ConversationHistory []InteractionInfo  `json:"conversationHistory"`
    }
    ```
    

**`c) POST /api/interview/{sessionId}/join - Authorize Join & Get Signaling Info`**

- **Description:** Authenticated user requests to join a session.
    
- **Path Parameter:** `sessionId` (string)
    
- **Request Body:** None (UserID from JWT)
    
- **Response Body (200 OK):** `JoinInterviewResponse`
    
    ```
    type JoinInterviewResponse struct {
        SignalingToken string `json:"signalingToken"`
        WebSocketURL   string `json:"webSocketUrl"`
        SfuInfo        interface{} `json:"sfuInfo,omitempty"`
    }
    ```
    

_(Note: Internal operations like RecordInteraction, GetAIContext, UpdateParticipantStatus are now handled via gRPC, see Section 4)_

### 2.2 AI Service API (`/api/ai/*`) via Gateway

**`a) POST /api/ai/{sessionId}/process_response - Process Candidate Response`**

- **Description:** Frontend sends the final transcript of a candidate's response.
    
- **Path Parameter:** `sessionId` (string)
    
- **Request Body:** `ProcessResponseRequest`
    
    ```
    type ProcessResponseRequest struct {
        Transcript string `json:"transcript"`
        UserID     string `json:"userId"`
    }
    ```
    
- **Response Body (202 Accepted):** Empty body.
    

## 3. WebSocket Message Contracts (External: Frontend <-> Signaling Service)

Messages exchanged over the WebSocket connection (`/ws/signal`). All messages use the `WebSocketMessage` wrapper.

```
package contracts

import "encoding/json"

// Generic WebSocket message structure
type WebSocketMessage struct {
	Type    string          `json:"type"` // Defines the message type
	Payload json.RawMessage `json:"payload"` // The actual data
}

// Helper functions (NewWebSocketMessage, UnmarshalPayload) omitted for brevity

// --- Client-to-Server Message Payloads ---
type RequestMediaConfigPayload struct {} // Type: "request_media_config"
type SdpPayload struct { Target string `json:"target"`; Sdp string `json:"sdp"` } // Type: "offer", "answer"
type IceCandidatePayload struct { Target string `json:"target"`; Candidate interface{} `json:"candidate"` } // Type: "ice_candidate"
type ConnectionEstablishedPayload struct {} // Type: "connection_established"
type SpeakingStatusPayload struct { UserID string `json:"userId"`; Speaking bool `json:"speaking"` } // Type: "speaking_status"
type LeaveRequestPayload struct { UserID string `json:"userId"` } // Type: "leave_request"

// --- Server-to-Client Message Payloads ---
type MediaConfigPayload struct { SfuDetails interface{} `json:"sfuDetails"` } // Type: "media_config"
// SdpPayload (Type: "offer", "answer") - Same as Client->Server
// IceCandidatePayload (Type: "ice_candidate") - Same as Client->Server
type SessionReadyPayload struct { Participants []ParticipantInfo `json:"participants"` } // Type: "session_ready"
type ParticipantJoinedPayload struct { Participant ParticipantInfo `json:"participant"` } // Type: "participant_joined"
type ParticipantLeftPayload struct { UserID string `json:"userId"` } // Type: "participant_left"
type ParticipantSpeakingPayload struct { UserID string `json:"userId"`; Speaking bool `json:"speaking"` } // Type: "participant_speaking"
type AIStatusPayload struct { SessionID string `json:"sessionId"`; Status AIStatus `json:"status"` } // Type: "ai_status"
type AIMessagePayload struct { SessionID string `json:"sessionId"`; Text string `json:"text"`; AudioReference *string `json:"audioReference,omitempty"` } // Type: "ai_message"
type SystemMessagePayload struct { SessionID string `json:"sessionId"`; Text string `json:"text"`; Level string `json:"level"` } // Type: "system_message"
type LeaveAckPayload struct {} // Type: "leave_ack"

```

## 4. gRPC Service Definitions (.proto) (Internal: Backend <-> Backend)

This section defines the contracts for internal communication between backend microservices using Protocol Buffers v3.

### 4.1 `common.proto`

```
syntax = "proto3";

package interview.common;

import "google/protobuf/timestamp.proto";

option go_package = "gen/common";

enum Role { ROLE_UNSPECIFIED = 0; ROLE_CANDIDATE = 1; ROLE_INTERVIEWER = 2; ROLE_AI = 3; }
enum SessionStatus { SESSION_STATUS_UNSPECIFIED = 0; SESSION_STATUS_SCHEDULED = 1; SESSION_STATUS_IN_PROGRESS = 2; SESSION_STATUS_COMPLETED = 3; SESSION_STATUS_CANCELLED = 4; }
enum ParticipantStatus { PARTICIPANT_STATUS_UNSPECIFIED = 0; PARTICIPANT_STATUS_JOINING = 1; PARTICIPANT_STATUS_CONNECTED = 2; PARTICIPANT_STATUS_DISCONNECTED = 3; PARTICIPANT_STATUS_FAILED = 4; }
enum InteractionType { INTERACTION_TYPE_UNSPECIFIED = 0; INTERACTION_TYPE_QUESTION = 1; INTERACTION_TYPE_ANSWER = 2; INTERACTION_TYPE_COMMENT = 3; INTERACTION_TYPE_SYSTEM_MESSAGE = 4; }
enum AIStatus { AI_STATUS_UNSPECIFIED = 0; AI_STATUS_IDLE = 1; AI_STATUS_LISTENING = 2; AI_STATUS_PROCESSING = 3; AI_STATUS_SPEAKING = 4; AI_STATUS_ERROR = 5; }

message Participant { string participant_id = 1; string user_id = 2; Role role = 3; string name = 4; ParticipantStatus status = 5; }
message Interaction { string interaction_id = 1; google.protobuf.Timestamp timestamp = 2; string speaker_participant_id = 3; string speaker_user_id = 4; Role speaker_role = 5; InteractionType type = 6; string content = 7; optional string audio_reference = 8; }
```

### 4.2 `interview_service.proto`

```
syntax = "proto3";

package interview.v1;

import "google/protobuf/timestamp.proto";
import "common.proto";

option go_package = "gen/interview/v1";

// Service to manage interview sessions
service InterviewService {
  rpc CreateInterview(CreateInterviewRequest) returns (CreateInterviewResponse);
  rpc GetInterviewDetails(GetInterviewDetailsRequest) returns (GetInterviewDetailsResponse);
  rpc RecordInteraction(RecordInteractionRequest) returns (RecordInteractionResponse);
  rpc GetAIContext(GetAIContextRequest) returns (GetAIContextResponse);
  rpc UpdateParticipantStatus(UpdateParticipantStatusRequest) returns (UpdateParticipantStatusResponse);
  rpc GetParticipant(GetParticipantRequest) returns (GetParticipantResponse);
  // Add other RPCs as needed, e.g., GetSessionStatus
}

// --- Messages for InterviewService RPCs ---
message CreateInterviewRequest { google.protobuf.Timestamp scheduled_time = 1; string job_description = 2; map<string, interview.common.Role> participant_permissions = 3; }
message CreateInterviewResponse { string session_id = 1; interview.common.SessionStatus status = 2; google.protobuf.Timestamp scheduled_time = 3; }
message GetInterviewDetailsRequest { string session_id = 1; }
message GetInterviewDetailsResponse { string session_id = 1; interview.common.SessionStatus status = 2; google.protobuf.Timestamp scheduled_time = 3; string job_description = 4; repeated interview.common.Participant participants = 5; repeated interview.common.Interaction conversation_history = 6; }
message RecordInteractionRequest { string session_id = 1; string speaker_participant_id = 2; string speaker_user_id = 3; interview.common.Role speaker_role = 4; interview.common.InteractionType type = 5; string content = 6; optional string audio_reference = 7; }
message RecordInteractionResponse { string interaction_id = 1; }
message GetAIContextRequest { string session_id = 1; }
message GetAIContextResponse { string job_description = 1; repeated interview.common.Interaction conversation_history = 2; }
message UpdateParticipantStatusRequest { string session_id = 1; string user_id = 2; interview.common.ParticipantStatus status = 3; optional string participant_id = 4; }
message UpdateParticipantStatusResponse { interview.common.Participant participant = 1; }
message GetParticipantRequest { string session_id = 1; string user_id = 2; }
message GetParticipantResponse { interview.common.Participant participant = 1; }
```

### 4.3 `ai_service.proto`

```
syntax = "proto3";

package ai.v1;

import "common.proto";

option go_package = "gen/ai/v1";

// Service for AI interactions (minimal RPCs exposed, mostly acts as client)
service AIService {
  // Potentially add RPCs if other services need to trigger AI actions via gRPC
  // e.g., rpc ForceQuestion(ForceQuestionRequest) returns (ForceQuestionResponse);
}

// --- Messages for AIService RPCs (if any) ---
```

### 4.4 `signaling_service.proto`

```
syntax = "proto3";

package signaling.v1;

import "common.proto";

option go_package = "gen/signaling/v1";

// Service for signaling-related internal communication (e.g., broadcasting)
service SignalingInternalService { // Renamed to avoid conflict if external gRPC signaling is ever needed
    // Called by AI Service (or via event bus) to broadcast status/messages
    rpc BroadcastAIStatus(BroadcastAIStatusRequest) returns (BroadcastAIStatusResponse);
    rpc BroadcastAIMessage(BroadcastAIMessageRequest) returns (BroadcastAIMessageResponse);
    rpc BroadcastSystemMessage(BroadcastSystemMessageRequest) returns (BroadcastSystemMessageResponse);
}

// --- Messages for SignalingInternalService RPCs ---
message BroadcastAIStatusRequest { string session_id = 1; interview.common.AIStatus status = 2; }
message BroadcastAIStatusResponse {}
message BroadcastAIMessageRequest { string session_id = 1; string text = 2; optional string audio_reference = 3; }
message BroadcastAIMessageResponse {}
message BroadcastSystemMessageRequest { string session_id = 1; string text = 2; string level = 3; }
message BroadcastSystemMessageResponse {}
```

## 5. Database Schema (PostgreSQL)

_(This section remains the same as the persistence layer is independent of the inter-service communication protocol)_

```
-- Represents registered users
CREATE TABLE users (
    user_id VARCHAR(255) PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- Main table for interview sessions
CREATE TABLE interview_sessions (
    session_id VARCHAR(255) PRIMARY KEY,
    status VARCHAR(50) NOT NULL DEFAULT 'scheduled',
    scheduled_time TIMESTAMPTZ,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    job_description TEXT,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- Stores which users are permitted to join which session and their role
CREATE TABLE access_permissions (
    permission_id SERIAL PRIMARY KEY,
    session_id VARCHAR(255) NOT NULL REFERENCES interview_sessions(session_id) ON DELETE CASCADE,
    user_id VARCHAR(255) NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    role VARCHAR(50) NOT NULL,
    UNIQUE (session_id, user_id)
);

-- Tracks participants connected to a session (can be ephemeral or persisted)
CREATE TABLE participants (
    participant_id VARCHAR(255) PRIMARY KEY,
    session_id VARCHAR(255) NOT NULL REFERENCES interview_sessions(session_id) ON DELETE CASCADE,
    user_id VARCHAR(255) NOT NULL REFERENCES users(user_id) ON DELETE RESTRICT,
    role VARCHAR(50) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'joining',
    joined_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    left_at TIMESTAMPTZ,
    signaling_id VARCHAR(255)
);

-- Stores the conversation history
CREATE TABLE interactions (
    interaction_id VARCHAR(255) PRIMARY KEY,
    session_id VARCHAR(255) NOT NULL REFERENCES interview_sessions(session_id) ON DELETE CASCADE,
    timestamp TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    speaker_participant_id VARCHAR(255) REFERENCES participants(participant_id) ON DELETE SET NULL,
    speaker_user_id VARCHAR(255) NOT NULL,
    speaker_role VARCHAR(50) NOT NULL,
    interaction_type VARCHAR(50) NOT NULL,
    content TEXT NOT NULL,
    audio_reference VARCHAR(1024)
);

-- Indexes for common lookups
CREATE INDEX idx_access_permissions_session ON access_permissions(session_id);
CREATE INDEX idx_participants_session ON participants(session_id);
CREATE INDEX idx_interactions_session ON interactions(session_id);
-- Add other necessary indexes

-- Optional: Table for AI Conversation State
CREATE TABLE ai_conversations (
    session_id VARCHAR(255) PRIMARY KEY REFERENCES interview_sessions(session_id) ON DELETE CASCADE,
    current_state VARCHAR(50) NOT NULL DEFAULT 'idle',
    current_topic TEXT,
    evaluation_context JSONB,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
```