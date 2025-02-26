
#### Sequence diagram
```mermaid
sequenceDiagram
    participant Client1 as Client 1 (React Native)
    participant Server as Signaling Server (Golang)
    participant Client2 as Client 2 (React Native)
 
    Note over Client1, Client2: Room Creation and Joining Flow
    Client1->>Server: Create Room (POST /api/rooms)
    Server-->>Client1: Return Room ID
 
    Client2->>Server: Join Room (POST /api/rooms/{roomId}/join)
    Server-->>Client2: Return Room Details
    Server->>Client1: Notify New Participant (WebSocket)
 
    Note over Client1, Client2: WebRTC Connection Establishment
    Client1->>Server: Send SDP Offer (WebSocket)
    Server->>Client2: Forward SDP Offer (WebSocket)
    Client2->>Server: Send SDP Answer (WebSocket)
    Server->>Client1: Forward SDP Answer (WebSocket)
 
    Client1->>Server: Send ICE Candidate (WebSocket)
    Server->>Client2: Forward ICE Candidate (WebSocket)
    Client2->>Server: Send ICE Candidate (WebSocket)
    Server->>Client1: Forward ICE Candidate (WebSocket)
 
    Note over Client1, Client2: P2P Connection Established
 
    Note over Client1, Client2: Chat and Video Communication Flow
    Client1->>Client2: Direct P2P Media Stream (WebRTC)
    Client2->>Client1: Direct P2P Media Stream (WebRTC)
 
    Client1->>Server: Send Chat Message (WebSocket)
    Server->>Client2: Forward Chat Message (WebSocket)
    Client2->>Server: Send Chat Message (WebSocket)
    Server->>Client1: Forward Chat Message (WebSocket)
 
    Note over Client1, Client2: Disconnection Flow
    Client1->>Server: Leave Room (POST /api/rooms/{roomId}/leave)
    Server->>Client2: Notify Participant Left (WebSocket)
    Client2->>Server: Leave Room (POST /api/rooms/{roomId}/leave)
    Server->>Client1: Notify Participant Left (WebSocket)

```



#### Detailed sequence diagram
```mermaid
sequenceDiagram
    participant User1 as User 1
    participant MobileApp1 as Mobile App (User 1)
    participant Backend as Go Backend Server
    participant SignalingServer as WebRTC Signaling Server
    participant MobileApp2 as Mobile App (User 2)
    participant User2 as User 2
    
    %% Room Creation Flow
    User1->>MobileApp1: Enter app
    MobileApp1->>Backend: Authenticate user
    Backend->>MobileApp1: Authentication token
    User1->>MobileApp1: Create/Join room (enter room ID)
    MobileApp1->>Backend: Request to join room
    Backend->>MobileApp1: Room joined successfully
    MobileApp1->>SignalingServer: Connect to signaling server (WebSocket)
    
    %% User 2 Join Flow
    User2->>MobileApp2: Enter app
    MobileApp2->>Backend: Authenticate user
    Backend->>MobileApp2: Authentication token
    User2->>MobileApp2: Enter same room ID
    MobileApp2->>Backend: Request to join room
    Backend->>MobileApp2: Room joined successfully
    MobileApp2->>SignalingServer: Connect to signaling server (WebSocket)
    
    %% WebRTC Connection Establishment
    SignalingServer->>MobileApp1: Notify new peer joined
    MobileApp1->>SignalingServer: Send SDP offer
    SignalingServer->>MobileApp2: Forward SDP offer
    MobileApp2->>SignalingServer: Send SDP answer
    SignalingServer->>MobileApp1: Forward SDP answer
    
    %% ICE Candidate Exchange
    MobileApp1->>SignalingServer: Send ICE candidates
    SignalingServer->>MobileApp2: Forward ICE candidates
    MobileApp2->>SignalingServer: Send ICE candidates
    SignalingServer->>MobileApp1: Forward ICE candidates
    
    %% Media Connection Established
    MobileApp1-->>MobileApp2: Establish P2P WebRTC connection
    
    %% Chat and Video Streaming
    User1->>MobileApp1: Send chat message
    MobileApp1-->>MobileApp2: Send via data channel
    MobileApp2->>User2: Display message
    
    User2->>MobileApp2: Send chat message
    MobileApp2-->>MobileApp1: Send via data channel
    MobileApp1->>User1: Display message
    
    %% Video streams are automatically flowing after connection established
    
    %% Disconnect Flow
    User1->>MobileApp1: Leave room
    MobileApp1->>SignalingServer: Notify disconnect
    SignalingServer->>MobileApp2: Notify peer disconnected
    MobileApp2->>User2: Display peer disconnected
```
