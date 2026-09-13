# NEW Case Study: Ticketmaster (High-Concurrency Booking)
- **Requirements:** Sell 50,000 Taylor Swift tickets in minutes without overselling.
- **Architecture:** Virtual Waiting Room + Distributed Locks.
- **Key Insight:** When a user clicks "Select Seat", acquire a Distributed Lock (Redis Redlock) with a TTL of 5 minutes. If they don't complete payment, the TTL expires and the seat is released. This temporarily reserves inventory without blocking the main DB.
