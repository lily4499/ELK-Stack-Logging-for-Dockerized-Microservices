
# 📊 ELK Stack Logging for Dockerized Microservices

This project sets up the **ELK Stack (Elasticsearch, Logstash, Kibana)** to **centralize and visualize logs** from two microservices running in Docker containers: `user-service` and `order-service`.

---

## 📘 Real-World Scenario

You are a DevOps engineer at a company that runs two microservices—**User Service** and **Order Service**—in Docker containers.  
The company wants to:
- 📦 Collect logs centrally,
- 🪵 Filter logs by severity (`INFO`, `ERROR`),
- 📈 Visualize logs using dashboards in **Kibana**.

You will use the **ELK Stack** to achieve this.

---

## 🗂️ Project Structure

```
elk-docker-project/
├── elk/
│   ├── docker-compose.yml
│   ├── logstash/
│   │   ├── logstash.conf
│   │   └── pipelines.yml
├── microservices/
│   ├── user-service/
│   │   ├── app.js
│   │   ├── Dockerfile
│   │   └── logs/user.log
│   ├── order-service/
│   │   ├── app.js
│   │   ├── Dockerfile
│   │   └── logs/order.log
└── README.md
```
---

## file-setup.py

```python

import os

base_dir = "/home/lilia/VIDEOS/elk-docker-project"

files_content = {
    "elk/docker-compose.yml": """version: '3.7'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.10
    environment:
      - discovery.type=single-node
    ports:
      - "9200:9200"

  logstash:
    image: docker.elastic.co/logstash/logstash:7.17.10
    volumes:
      - ./logstash/logstash.conf:/usr/share/logstash/pipeline/logstash.conf
      - ../microservices/user-service/logs:/logs
      - ../microservices/order-service/logs:/logs
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:7.17.10
    ports:
      - "5601:5601"
    environment:
      ELASTICSEARCH_HOSTS: "http://elasticsearch:9200"
    depends_on:
      - elasticsearch
""",

    "elk/logstash/logstash.conf": """input {
  file {
    path => "/logs/user.log"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    tags => ["user-service"]
  }

  file {
    path => "/logs/order.log"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    tags => ["order-service"]
  }
}

filter {
  grok {
    match => { "message" => "\\[%{LOGLEVEL:level}\\] %{GREEDYDATA:log_message}" }
  }

  if "user-service" in [tags] {
    mutate {
      add_field => { "service_name" => "user-service" }
    }
  } else if "order-service" in [tags] {
    mutate {
      add_field => { "service_name" => "order-service" }
    }
  }

  mutate {
    add_field => {
      "host" => "%{host}"
    }
  }
}

output {
  elasticsearch {
    hosts => "http://elasticsearch:9200"
    index => "microservices-logs"
  }

  stdout { codec => rubydebug }
}
""",

    "elk/logstash/pipelines.yml": """- pipeline.id: main
  path.config: "/usr/share/logstash/pipeline/logstash.conf"
""",

    "microservices/user-service/app.js": """const express = require('express');
const fs = require('fs');
const app = express();

app.get('/', (req, res) => {
  const log = `[INFO] User requested at ${new Date().toISOString()}\\n`;
  fs.appendFileSync('./logs/user.log', log);
  res.send('User Service Running');
});

app.listen(3001, () => console.log('User Service on port 3001'));
""",

    "microservices/user-service/Dockerfile": """FROM node:18
WORKDIR /app
COPY . .
RUN mkdir -p logs && npm install express
EXPOSE 3001
CMD ["node", "app.js"]
""",

    "microservices/user-service/logs/user.log": "",

    "microservices/order-service/app.js": """const express = require('express');
const fs = require('fs');
const app = express();

app.get('/', (req, res) => {
  const log = `[ERROR] Order failed at ${new Date().toISOString()}\\n`;
  fs.appendFileSync('./logs/order.log', log);
  res.send('Order Service Running');
});

app.listen(3002, () => console.log('Order Service on port 3002'));
""",

    "microservices/order-service/Dockerfile": """FROM node:18
WORKDIR /app
COPY . .
RUN mkdir -p logs && npm install express
EXPOSE 3002
CMD ["node", "app.js"]
""",

    "microservices/order-service/logs/order.log": ""
}

# Create the files with their content
for rel_path, content in files_content.items():
    full_path = os.path.join(base_dir, rel_path)
    os.makedirs(os.path.dirname(full_path), exist_ok=True)
    with open(full_path, 'w') as f:
        f.write(content)

"All files created successfully in /home/lilia/VIDEOS/elk-docker-project."

```


---

## 🚀 Step-by-Step Setup

### ✅ Step 1: Create Microservices

#### `user-service/app.js`
```js
const express = require('express');
const fs = require('fs');
const app = express();

app.get('/', (req, res) => {
  const log = `[INFO] User requested at ${new Date().toISOString()}\n`;
  fs.appendFileSync('./logs/user.log', log);
  res.send('User Service Running');
});

app.listen(3001, () => console.log('User Service on port 3001'));
```

#### `order-service/app.js`
```js
const express = require('express');
const fs = require('fs');
const app = express();

app.get('/', (req, res) => {
  const log = `[ERROR] Order failed at ${new Date().toISOString()}\n`;
  fs.appendFileSync('./logs/order.log', log);
  res.send('Order Service Running');
});

app.listen(3002, () => console.log('Order Service on port 3002'));
```

---

### ✅ Step 2: Dockerize the Microservices

#### `Dockerfile` (shared)
```dockerfile
FROM node:18
WORKDIR /app
COPY . .
RUN mkdir -p logs && npm install express
EXPOSE 3001
CMD ["node", "app.js"]
```

> Change `EXPOSE` to 3002 for `order-service`.

---

### ✅ Step 3: Logstash Configuration

#### `elk/logstash/logstash.conf`
```conf
input {
  file {
    path => "/logs/user.log"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    tags => ["user-service"]
  }

  file {
    path => "/logs/order.log"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    tags => ["order-service"]
  }
}

filter {
  grok {
    match => { "message" => "\[%{LOGLEVEL:level}\] %{GREEDYDATA:log_message}" }
  }

  if "user-service" in [tags] {
    mutate {
      add_field => { "service_name" => "user-service" }
    }
  } else if "order-service" in [tags] {
    mutate {
      add_field => { "service_name" => "order-service" }
    }
  }

  mutate {
    add_field => {
      "host" => "%{host}"
    }
  }

  # Optional: Static IP (if known) or extract from logs if available
  # mutate {
  #   add_field => {
  #     "ip" => "192.168.1.100"
  #   }
  # }
}

output {
  elasticsearch {
    hosts => "http://elasticsearch:9200"
    index => "microservices-logs"
  }

  stdout { codec => rubydebug }
}

```

---

### ✅ Step 4: Logstash Pipeline File

#### `elk/logstash/pipelines.yml`
```yaml
- pipeline.id: main
  path.config: "/usr/share/logstash/pipeline/logstash.conf"
```

---

### ✅ Step 5: Docker Compose for ELK

#### `elk/docker-compose.yml`
```yaml
version: '3.7'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.10
    environment:
      - discovery.type=single-node
    ports:
      - "9200:9200"

  logstash:
    image: docker.elastic.co/logstash/logstash:7.17.10
    volumes:
      - ./logstash/logstash.conf:/usr/share/logstash/pipeline/logstash.conf
      - ../microservices/user-service/logs:/logs
      - ../microservices/order-service/logs:/logs
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:7.17.10
    ports:
      - "5601:5601"
    environment:
      ELASTICSEARCH_HOSTS: "http://elasticsearch:9200"
    depends_on:
      - elasticsearch
```

---

## 🧪 Run the Setup

### 🧱 Build Microservice Images

```bash
# Build user-service
cd microservices/user-service
docker build -t user-service .

# Build order-service
cd ../order-service
docker build -t order-service .
```

### 🐳 Run Microservices

```bash
docker run -d --name user-service -p 3001:3001 -v $PWD/logs:/app/logs user-service
docker run -d --name order-service -p 3002:3002 -v $PWD/logs:/app/logs order-service
```

### ⚙️ Start the ELK Stack

```bash
cd elk
docker-compose up -d
```

---

## 🌐 Access Kibana

Visit [http://localhost:5601](http://localhost:5601)

### ➕ Create an Index Pattern
- Go to **Management > Index Patterns**
- Create index pattern: `microservices-logs`

### 🔎 Filter Logs
- Use Kibana Discover to filter by:
  - `level: ERROR`
  - `level: INFO`

### 📈 Create Dashboards
- Create charts:
  - Pie chart by `level`
  - Line graph over time
  - Table grouped by log type

---

## 📌 Summary of What Each Step Does

| Step | Purpose |
|------|---------|
| 1 | Builds microservices that generate logs |
| 2 | Dockerizes the services |
| 3 | Sets up Logstash to ingest and parse logs |
| 4 | Configures Logstash pipeline |
| 5 | Deploys Elasticsearch, Logstash, and Kibana via Docker Compose |
| 6 | Runs services and mounts log folders |
| 7 | Starts ELK containers |
| 8 | Accesses Kibana to view and filter logs |
| 9 | Builds dashboards to monitor microservice behavior |

---
---

# 📊 Kibana Visualizations for ELK-Logged Microservices

## 📌 Prerequisites

Before creating visualizations, ensure:

- ELK stack is running (via Docker Compose)
- Logstash is ingesting logs from both services
- You can access Kibana at: [http://localhost:5601](http://localhost:5601)
- You've created an index pattern (e.g., `microservices-logs`)
- Logs include enriched fields such as:
  - `level` (e.g., INFO, ERROR)
  - `service_name` (e.g., user-service, order-service)
  - `timestamp`

---

## 🍩 1. Pie Chart by Log Level (`level`)

### ✅ Purpose
Show proportion of logs by severity: `INFO`, `ERROR`, etc.

### 📍 Steps
1. Go to **Visualize Library** → **Create Visualization**
2. Select **Pie** chart
3. Choose the index pattern: `microservices-logs`
4. Under **Buckets**, click **Add > Split Slices**
   - Aggregation: `Terms`
   - Field: `level.keyword`
   - Size: `5`
5. Click **Apply**

This displays the breakdown of logs by severity level.

---

## 📈 2. Line Graph of Logs Over Time

### ✅ Purpose
Visualize trends in log volume over time.

### 📍 Steps
1. Go to **Visualize Library** → **Create Visualization**
2. Choose **Line** chart
3. Select `microservices-logs` index
4. **X-axis**:
   - Aggregation: `Date Histogram`
   - Field: `@timestamp`
5. **Y-axis**:
   - Aggregation: `Count`
6. (Optional) Split series by:
   - Aggregation: `Terms`
   - Field: `service_name.keyword`

This creates a line graph showing log frequency and spikes over time.

---

## 📋 3. Table Grouped by `service_name` and `level`

### ✅ Purpose
View how many log messages of each type were generated by each service.

### 📍 Steps
1. Create a **Data Table** visualization
2. Select `microservices-logs` index
3. **Metrics**:
   - Aggregation: `Count`
4. **Buckets**:
   - Split Rows → Aggregation: `Terms`, Field: `service_name.keyword`
   - Split Rows → Aggregation: `Terms`, Field: `level.keyword`
5. Click **Apply**

This produces a table showing log counts grouped by service and severity.

---

## 📊 Add All Visuals to a Dashboard

### ✅ Steps
1. Go to **Dashboard > Create Dashboard**
2. Click **Add** and include:
   - Pie chart
   - Line graph
   - Data table
3. Save the dashboard as: `Microservices Log Dashboard`

Now you have a single view for real-time monitoring of your app logs!

---

