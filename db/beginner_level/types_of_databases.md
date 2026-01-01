# 🗄️ Types of Databases

---

## 1. Relational Databases (SQL)
- Data is stored in multiple tables, and tables can be **related** to each other using keys.  
- Ideal for structured data with clear relationships.  

**Examples:**  
- MySQL  
- PostgreSQL  
- Oracle  

**Think of it as:**  
A set of interconnected spreadsheets with rules.  

---

## 2. Key–Value Databases
- **How they work:** Data is stored as **key → value** pairs.  
- **Best for:**  
  - Caching  
  - Session storage  
  - Fast lookups  

**Examples:**  
- Redis  
- DynamoDB  
- Riak  

**Think of it as:**  
A JavaScript object or a dictionary.  

---

## 3. Column-Based (Wide-Column) Databases
- **How they work:** Data is stored in **columns instead of rows**.  
- **Best for:**  
  - Large-scale analytics  
  - Time-series data  
  - High write throughput  

**Examples:**  
- Cassandra  
- HBase  
- ScyllaDB  

**Think of it as:**  
A massive spreadsheet optimized for scale.  

---

## 4. Graph Databases
- **How they work:** Data is stored as **nodes and relationships**.  
- **Best for:**  
  - Social networks  
  - Recommendation systems  
  - Fraud detection  

**Examples:**  
- Neo4j  
- Amazon Neptune  
- ArangoDB  

**Think of it as:**  
A network or graph structure.  

---

## 5. Document Databases
- **How they work:** Data is stored as **documents** (usually JSON or BSON).  
- **Best for:**  
  - Flexible schemas  
  - APIs  
  - Content management systems  

**Examples:**  
- MongoDB  
- CouchDB  
- Firestore  

**Think of it as:**  
A collection of JSON files.  

---

## ⚡ Summary
- **Relational Databases = SQL**  
- **Key–Value, Column-Based, Graph, Document = NoSQL (4 types)**  
