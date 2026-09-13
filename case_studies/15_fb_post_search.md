# Case Study: Facebook Post Search (Security/BOLA)
- **Requirements:** Search billions of posts, strictly enforcing privacy rules (Friends Only, Public, Custom).
- **Architecture:** Unicorn (inverted index search) + privacy evaluation layer.
- **Key Insight:** Prevent BOLA (Broken Object Level Authorization) by applying ACLs dynamically at query time using graph intersection (Search matches INTERSECT user's visible graph), rather than trying to embed privacy into a static search index.
