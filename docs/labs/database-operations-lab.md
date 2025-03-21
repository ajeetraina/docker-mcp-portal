---
layout: lab
title: Lab 3: Database Operations with MCP Servers
description: Learn how to use Docker MCP servers to interact with databases and perform various database operations through Gordon AI.
difficulty: Intermediate
time: 60 minutes
author: Docker Team
last_updated: March 18, 2025
prev_lab: /docs/labs/research-assistant-lab
next_lab: /docs/labs/production-deployment-lab
---

<div class="lab-prerequisites">
  <h2><i class="fas fa-clipboard-list"></i> Prerequisites</h2>
  <ul>
    <li>Completion of <a href="/docs/labs/mcp-101-lab">Lab 1: First Steps with Docker MCP Servers</a></li>
    <li>Docker Desktop installed</li>
    <li>Basic SQL knowledge</li>
  </ul>
</div>

<div class="learning-objectives">
  <h2><i class="fas fa-graduation-cap"></i> Learning Objectives</h2>
  <ol>
    <li>Configure MCP servers to work with both SQLite and PostgreSQL databases</li>
    <li>Perform complex database operations via natural language requests</li>
    <li>Understand the security implications of database access via MCP</li>
    <li>Create data analysis workflows combining multiple MCP servers</li>
  </ol>
</div>

<div class="lab-step">
  <div class="lab-step-header">
    <i class="fas fa-play-circle"></i> Step 1: Setting Up Your Environment
  </div>
  <div class="lab-step-content">
    <p>Create a new directory for your database lab:</p>

```bash
mkdir mcp-db-lab
cd mcp-db-lab
mkdir data
```
  </div>
</div>

<div class="lab-step">
  <div class="lab-step-header">
    <i class="fas fa-database"></i> Step 2: Create a Sample SQLite Database
  </div>
  <div class="lab-step-content">
    <p>First, let's create a SQL script to initialize our database:</p>

```bash
cat << EOF > data/init.sql
CREATE TABLE employees (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    department TEXT NOT NULL,
    salary REAL NOT NULL,
    hire_date TEXT NOT NULL
);

INSERT INTO employees (name, department, salary, hire_date) VALUES
('John Doe', 'Engineering', 85000, '2020-03-15'),
('Jane Smith', 'Marketing', 75000, '2019-11-01'),
('Bob Johnson', 'Engineering', 92000, '2018-05-20'),
('Alice Williams', 'HR', 65000, '2021-01-10'),
('Charlie Brown', 'Marketing', 78000, '2020-09-05'),
('David Miller', 'Engineering', 115000, '2015-07-12'),
('Emily Davis', 'Finance', 95000, '2017-04-22'),
('Frank Wilson', 'HR', 68000, '2019-08-30'),
('Grace Lee', 'Engineering', 105000, '2016-11-15'),
('Henry Taylor', 'Finance', 88000, '2018-06-03');

CREATE TABLE departments (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    budget REAL NOT NULL,
    location TEXT NOT NULL
);

INSERT INTO departments (name, budget, location) VALUES
('Engineering', 1500000, 'Building A'),
('Marketing', 800000, 'Building B'),
('HR', 400000, 'Building A'),
('Finance', 950000, 'Building C');

CREATE TABLE projects (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    start_date TEXT NOT NULL,
    end_date TEXT,
    department_id INTEGER,
    FOREIGN KEY (department_id) REFERENCES departments(id)
);

INSERT INTO projects (name, start_date, end_date, department_id) VALUES
('Website Redesign', '2023-01-15', '2023-06-30', 2),
('Cloud Migration', '2023-02-01', NULL, 1),
('Budget Analysis', '2023-03-10', '2023-04-15', 4),
('Hiring Campaign', '2023-02-20', '2023-05-10', 3),
('Mobile App', '2022-11-01', NULL, 1);

CREATE TABLE employee_projects (
    employee_id INTEGER,
    project_id INTEGER,
    role TEXT NOT NULL,
    PRIMARY KEY (employee_id, project_id),
    FOREIGN KEY (employee_id) REFERENCES employees(id),
    FOREIGN KEY (project_id) REFERENCES projects(id)
);

INSERT INTO employee_projects (employee_id, project_id, role) VALUES
(1, 2, 'Developer'),
(1, 5, 'Lead Developer'),
(2, 1, 'Project Manager'),
(3, 2, 'DevOps Engineer'),
(3, 5, 'Developer'),
(4, 4, 'Coordinator'),
(5, 1, 'Content Creator'),
(6, 2, 'Architect'),
(7, 3, 'Analyst'),
(8, 4, 'Recruiter'),
(9, 5, 'Developer'),
(10, 3, 'Supervisor');
EOF
```
  </div>
</div>

<div class="lab-step">
  <div class="lab-step-header">
    <i class="fas fa-cogs"></i> Step 3: Configure MCP Servers for Database Operations
  </div>
  <div class="lab-step-content">
    <p>Create a <code>gordon-mcp.yml</code> file with SQLite MCP server configuration:</p>

```yaml
services:
  sqlite:
    image: mcp/sqlite
    volumes:
      - ./data:/data
  
  fs:
    image: mcp/filesystem
    command:
      - /rootfs
    volumes:
      - ./data:/rootfs/data
      - ./:/rootfs/output
```

    <p>This configuration sets up two MCP servers:</p>
    <ul>
      <li>A SQLite server that can interact with SQLite databases</li>
      <li>A filesystem server that gives access to our data directory</li>
    </ul>
  </div>
</div>

<div class="lab-step">
  <div class="lab-step-header">
    <i class="fas fa-database"></i> Step 4: Basic SQLite Database Operations
  </div>
  <div class="lab-step-content">
    <p>Let's start with some basic SQLite operations:</p>

```bash
# Initialize the database
docker ai "Create a new SQLite database called 'company.db' in the data directory and run the SQL commands from the init.sql file to set it up."
```

    <p>You should see Gordon AI using the SQLite MCP server to create the database. Now let's try some queries:</p>

```bash
# Basic query
docker ai "Query the company database and list all employees in the Engineering department with their salaries."
```

    <p>You should see a list of employees in the Engineering department. Let's try a more complex query:</p>

```bash
# More complex query
docker ai "Find the average salary by department and identify which department has the highest average salary. Format the results as a markdown table."
```

    <div class="lab-tip">
      <h4><i class="fas fa-lightbulb"></i> Tip</h4>
      <p>You can ask Gordon AI to format results in various ways, such as tables, lists, or even generate visualizations with descriptions.</p>
    </div>
  </div>
</div>

<div class="lab-step">
  <div class="lab-step-header">
    <i class="fas fa-server"></i> Step 5: Set Up PostgreSQL for Advanced Operations
  </div>
  <div class="lab-step-content">
    <p>For more advanced database operations, let's add PostgreSQL to our environment. First, we'll start a PostgreSQL container with our sample data:</p>

```bash
# Start a PostgreSQL container
docker run -d \
  --name postgres-mcp-lab \
  -e POSTGRES_PASSWORD=labpassword \
  -e POSTGRES_USER=labuser \
  -e POSTGRES_DB=company \
  -p 5432:5432 \
  postgres:14
```

    <p>Now, update your <code>gordon-mcp.yml</code> file to include the PostgreSQL MCP server:</p>

```yaml
services:
  sqlite:
    image: mcp/sqlite
    volumes:
      - ./data:/data
  
  fs:
    image: mcp/filesystem
    command:
      - /rootfs
    volumes:
      - ./data:/rootfs/data
      - ./:/rootfs/output
  
  postgres:
    image: mcp/postgres
    command: postgresql://labuser:labpassword@host.docker.internal:5432/company
```

    <p>Initialize our PostgreSQL database with the same schema:</p>

```bash
docker ai "Connect to the PostgreSQL database and create the same schema as in our SQLite database. Use the init.sql file as reference. Then confirm the tables were created correctly."
```
  </div>
</div>

<div class="lab-step">
  <div class="lab-step-header">
    <i class="fas fa-chart-bar"></i> Step 6: Advanced Database Operations
  </div>
  <div class="lab-step-content">
    <p>Now let's perform more complex database tasks:</p>

    <h3>Task 1: Cross-Database Analysis</h3>

```bash
docker ai "Compare the schema between our SQLite database and PostgreSQL database. Are there any differences? Which database would you recommend for our company data and why?"
```

    <h3>Task 2: Data Analysis and Reporting</h3>

```bash
docker ai "Analyze the employee and project data in the PostgreSQL database. Which employees are working on multiple projects? Create a report showing the workload distribution across departments. Save the report as a markdown file called workload_analysis.md."
```

    <h3>Task 3: Database Schema Improvements</h3>

```bash
docker ai "Suggest improvements to our database schema. What indexes, constraints, or additional tables would you recommend? Create a SQL script with your recommendations and save it to output/schema_improvements.sql."
```

    <div class="lab-note">
      <h4><i class="fas fa-info-circle"></i> Understanding MCP Database Operations</h4>
      <p>When you ask Gordon AI to perform database operations:</p>
      <ol>
        <li>Gordon identifies the task requires database interaction</li>
        <li>It selects the appropriate MCP server (SQLite or PostgreSQL)</li>
        <li>The MCP server translates natural language into SQL queries</li>
        <li>Results are returned and formatted according to your request</li>
      </ol>
      <p>This powerful abstraction allows non-technical users to interact with databases using natural language.</p>
    </div>
  </div>
</div>

<div class="lab-step">
  <div class="lab-step-header">
    <i class="fas fa-code"></i> Step 7: Building a Database Dashboard
  </div>
  <div class="lab-step-content">
    <p>For a more advanced exercise, let's create a simple dashboard for our data:</p>

```bash
docker ai "Create an HTML dashboard that shows key metrics from our company database, including department budgets, employee salary distributions, and project statuses. Use JavaScript for any interactive elements. Save the dashboard to output/company_dashboard.html."
```

    <p>Once the dashboard is created, you can open it in your browser:</p>

```bash
# On macOS
open output/company_dashboard.html

# On Linux
xdg-open output/company_dashboard.html

# On Windows
start output/company_dashboard.html
```
  </div>
</div>

<div class="lab-step">
  <div class="lab-step-header">
    <i class="fas fa-shield-alt"></i> Step 8: Database Security Considerations
  </div>
  <div class="lab-step-content">
    <p>It's important to understand the security implications of using MCP servers with databases. Here are some best practices:</p>

    <ol>
      <li><strong>Connection Strings</strong>: Never expose database credentials in public repositories</li>
      <li><strong>Access Control</strong>: Use read-only connections when only queries are needed</li>
      <li><strong>Container Networks</strong>: Consider using Docker networks to isolate database containers</li>
      <li><strong>Volume Mounting</strong>: Be careful about what directories you mount to containers</li>
    </ol>

    <p>Let's implement a more secure configuration by creating a Docker network and updating our setup:</p>

```bash
# Create a Docker network
docker network create mcp-db-network

# Stop and remove the existing PostgreSQL container
docker stop postgres-mcp-lab
docker rm postgres-mcp-lab

# Start PostgreSQL with the new network
docker run -d \
  --name postgres-mcp-lab \
  --network mcp-db-network \
  -e POSTGRES_PASSWORD=labpassword \
  -e POSTGRES_USER=labuser \
  -e POSTGRES_DB=company \
  postgres:14
```

    <p>Create a new <code>secure-gordon-mcp.yml</code> file:</p>

```yaml
services:
  sqlite:
    image: mcp/sqlite
    volumes:
      - ./data:/data:ro  # Read-only access
  
  fs:
    image: mcp/filesystem
    command:
      - /rootfs
    volumes:
      - ./data:/rootfs/data:ro  # Read-only access to data
      - ./output:/rootfs/output:rw  # Write access only to output
  
  postgres:
    image: mcp/postgres
    command: postgresql://labuser:labpassword@postgres-mcp-lab:5432/company
    networks:
      - mcp-db-network

networks:
  mcp-db-network:
    external: true
    name: mcp-db-network
```

    <p>Let's test our secure configuration:</p>

```bash
docker ai --file secure-gordon-mcp.yml "Query the PostgreSQL database to find the highest paid employee in each department."
```
  </div>
</div>

<div class="lab-tip">
  <h4><i class="fas fa-lightbulb"></i> Advanced Database Tips</h4>
  <ul>
    <li><strong>Performance Optimization</strong>: Ask Gordon AI to generate database indexes for better performance</li>
    <li><strong>Data Migration</strong>: Use MCP servers to help migrate data between different database systems</li>
    <li><strong>Query Generation</strong>: Have Gordon AI generate complex queries that you can reuse in your applications</li>
    <li><strong>Documentation</strong>: Ask for database documentation to be generated automatically</li>
  </ul>
</div>

<div class="lab-note">
  <h4><i class="fas fa-exclamation-triangle"></i> Troubleshooting</h4>
  <ul>
    <li>If PostgreSQL connections fail, ensure the host.docker.internal resolution is working</li>
    <li>For SQLite issues, check file permissions on the data directory</li>
    <li>If queries return unexpected results, verify the database schema and data</li>
  </ul>
</div>

<div class="lab-conclusion">
  <h2><i class="fas fa-flag-checkered"></i> Conclusion</h2>
  <p>Congratulations! You've successfully learned how to use Docker MCP servers to interact with databases. You've seen how Gordon AI can translate natural language into SQL queries, perform complex data analysis, and generate reports and visualizations.</p>
  <p>This powerful capability allows team members with varying levels of technical expertise to work with databases effectively, democratizing access to data while maintaining security and control.</p>
</div>

<div class="next-steps">
  <h2><i class="fas fa-arrow-circle-right"></i> Next Steps</h2>
  <p>Now that you've mastered database operations with MCP, you can continue your learning journey with:</p>
  <ul>
    <li><a href="/docs/labs/production-deployment-lab">Lab 4: Deploying MCP Servers to Production</a> - Learn how to deploy your MCP servers in a production environment</li>
    <li><a href="/docs/tutorials/custom-mcp-server">Tutorial: Building Custom MCP Servers</a> - Create your own specialized MCP servers</li>
  </ul>
</div>