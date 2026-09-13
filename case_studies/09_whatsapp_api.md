# Case Study: WhatsApp (API Protocol Selection)
- **Requirements:** Fast internal service communication vs broad external client support.
- **Architecture:** API Gateway routing.
- **Key Insight:** Use REST externally for broad client compatibility. Use gRPC internally (Protobuf over HTTP/2) for dense, low-latency, strongly-typed microservice-to-microservice communication. Use WebSockets for the actual message delivery channel.
