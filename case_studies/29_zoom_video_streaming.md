# NEW Case Study: Zoom / WebEx (Video Streaming)
- **Requirements:** Multi-participant video, low latency (<150ms).
- **Architecture:** SFU (Selective Forwarding Unit) over UDP (WebRTC).
- **Key Insight:** Don't use TCP (packet retransmission causes lag). Use UDP. Instead of Mesh networking (O(N^2) connections), route all streams through a central SFU server that intelligently forwards streams based on who is speaking and bandwidth availability.
