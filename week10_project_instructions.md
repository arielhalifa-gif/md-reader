# **Week 10 Project: Contact Manager API with Docker Compose**

## **Table of Contents**

1. **Project Objectives**  
2. **Project Overview**  
3. **Functional Requirements**  
   * 2.1 Data Model  
   * 2.2 API Endpoints  
   * 2.3 Database Operations  
4. **Technical Requirements**  
   * 3.1 Required Project Structure  
   * 3.2 Class Design Requirements  
   * 3.3 Docker Requirements  
   * 3.4 Configuration Requirements  
5. **Git & Documentation Requirements**  
   * 4.1 Git Requirements  
   * 4.2 Documentation Requirements  
6. **Testing Requirements**  
7. **Submission Guidelines**  
   * 8.1 What to Submit  
   * 8.2 Submission Checklist  
   * 8.3 Testing Your Submission  
8. **Recommended Timeline**  
   * 9.1 Day 1 (6 hours)  
   * 9.2 Day 2 (6 hours)  
9. **Resources**  
   * 10.1 Recommended Material  
   * 10.2 Helpful References  
   * 10.3 Example Datasets  
10. **Frequently Asked Questions**  
11. **Important Notes**  
    * 12.1 Code Quality Expectations  
    * 12.2 Common Mistakes to Avoid

## Project Objectives

By completing this project, you will demonstrate:

- **Object-Oriented Programming** \- Designing and implementing Python classes  
- **Database Operations** \- Writing SQL queries manually (using string formatting, without ORM)  
- **API Development** \- Creating REST endpoints with FastAPI  
- **Containerization** \- Building and deploying with Docker  
- **Orchestration** \- Managing multi-container applications with Docker Compose  
- **Git Workflow** \- Professional version control practices

## Project Overview

You will build a **Contact Manager API** \- a REST API service that allows users to manage a contact list. The application will:

- Store contacts in a MySQL database  
- Provide HTTP endpoints for CRUD operations (Create, Read, Update, Delete)  
- Run in Docker containers using Docker Compose  
- Persist data using Docker volumes

**Technology Stack:**

- Python 3.11+  
- FastAPI (web framework)  
- MySQL 8.0 (database)  
- Docker & Docker Compose

**Duration:** 2 days

## Functional Requirements

### 1\. Data Model

Each contact must have the following fields:

| Field | Type | Constraints |
| :---- | :---- | :---- |
| id | Integer | Primary Key, Auto-increment |
| first\_name | String (50 chars) | Not null |
| last\_name | String (50 chars) | Not null |
| phone\_number | String (20 chars) | Not null, Unique |

### 

### 2\. API Endpoints

Your API must implement these 4 endpoints:

| Method | Endpoint | Description | Request Body | Response |
| :---- | :---- | :---- | :---- | :---- |
| GET | `/contacts` | Get all contacts | None | List of all contacts |
| POST | `/contacts` | Create new contact | Contact data | Success message \+ ID |
| PUT | `/contacts/{id}` | Update existing contact | Updated fields | Success message |
| DELETE | `/contacts/{id}` | Delete contact | None | Success message |

**Example Request/Response:**

**\# Create a contact**  
POST /contacts

{  
    "first\_name": "John",  
    "last\_name": "Doe",  
    "phone\_number": "050-1234567"  
}

**\# Response**  
{  
    "message": "Contact created successfully",  
    "id": 4  
}

### 3\. Database Operations

You must implement the following database operations **without using an ORM**:

- Create a new contact (INSERT)  
- Retrieve all contacts (SELECT)  
- Update a contact (UPDATE)  
- Delete a contact (DELETE)

SQL queries should be written manually using raw SQL statements (formatted to include relevant parameters).

## Technical Requirements

### Basic Project Structure

week10\_contacts/  
├── .gitignore  
├── README.md	\# What this project does and how to run the code  
├── compose.yaml  
├── app/  
│   ├── Dockerfile  
│   ├── main.py          \# FastAPI application  
│   ├── data\_interactor.py      \# Handles mysql connection and CRUD operations  
│   └── requirements.txt  
└── sql/  
    └── init.sql         \# Database Schema initialization script

**Note**: This is only the minimal requirement, and further separation of concerns is encouraged.

### 

### Class Design Requirements

You must implement the following:

1. **Contact Class**  
   - Properties: id, first\_name, last\_name, phone\_number  
   - Method to convert contact to dictionary format

   

2. **CRUD Functions**  
   - `create_contact(first_name, last_name, phone_number)` → returns new contact ID (queried from DB)  
   - `get_all_contacts()` → returns list of Contact objects  
   - `update_contact(id, ...)` → returns success boolean  
   - `delete_contact(id)` → returns success boolean

   

3. **Database Module**  
   - Function to create and return database connection  
   - Needs to handle connection errors

   

4. **FastAPI Application**  
   - Define request/response models using Pydantic  
   - Implement 4 API endpoints (routes)  
   - Handle errors with appropriate HTTP status codes  
   - **Note:** You do NOT need to use async functions \- regular functions are fine

### Docker Requirements

1. **Application Dockerfile** (`app/Dockerfile`)  
   - Base image: Python 3.11-slim  
   - Install dependencies from requirements.txt  
   - Copy application code  
   - Expose port 8000  
   - Run uvicorn server

   

2. **Docker Compose File** (`compose.yaml`)  
   - Define two services: `db` and `api`  
   - MySQL service with environment variables for database initialization  
   - API service built from local Dockerfile  
   - Named volume for MySQL data persistence  
   - Mount `init.sql` to initialize database  
   - API depends on database service  
   - Expose API on ports 8000:80

   

3. **Database Initialization** (`sql/init.sql`)  
   - Create database  
   - Create contacts table with proper schema  
   - Insert 4 sample contacts for testing (example given below)

   

4. **Configuration requirements**  
- Environment variables for MySQL container: `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`

## Git & Documentation Requirements

### Git Requirements

1. **Repository Setup**  
   - Create a GitHub repository named `week10_compose`  
   - Include `.gitignore` excluding:  
     - `__pycache__/`  
     - `*.pyc`  
     - `.env`  
     - Any IDE-specific files

   

2. **Commit Standards**  
   - Clear commit messages describing what changed  
   - Commit often  
- **Bonus (+5 pts):** Create a branch for each major component you develop and merge it into the main branch when it’s finished  
    
3. **Suggested Commit Flow**  
   - "Initial project structure"  
   - "Add database schema and init script"  
   - "Implement Contact class"  
   - "Add CRUD database methods"  
   - "Implement GET and POST endpoints"  
   - "Implement PUT and DELETE endpoints"  
   - "Add Docker configuration"  
   - "Add documentation and testing instructions"

### Documentation Requirements

Your `README.md` must include:

1. **Project Description** \- Brief overview of what the project does \+ API endpoints  
2. **Setup Instructions** \- Step-by-step guide to run the project:  
     
   \# Example \- from project root directory:  
     
   docker compose up \-d  
     
3. **Testing Instructions** \- Example curl commands to test each endpoint

## Testing Requirements

You must test and verify:

1. **Container Startup**  
     
   - Both containers start without errors  
   - API is accessible on port 8000  
   - Database initializes with sample data

   

   

   

2. **CRUD Operations**  
     
   - GET /contacts returns sample data  
   - POST /contacts creates new contact successfully  
   - PUT /contacts/{id} updates contact  
   - DELETE /contacts/{id} removes contact

   

3. **Data Persistence**  
     
   - Stop containers: `docker compose down`  
   - Restart containers: `docker compose up -d`  
   - Data should still exist (test with GET /contacts)

   

4. **Error Handling**  
     
   - Updating non-existent contact returns appropriate error  
   - Deleting non-existent contact returns appropriate error

## Submission Guidelines

### What to Submit

Submit your GitHub repository URL via MOODLE.

### Submission Checklist

Before submitting, verify:

- [ ] Repository is accessible (public and submitted to moodle)  
- [ ] README.md contains relevant information  
- [ ] .gitignore file exists and is properly configured  
- [ ] All required files are present in correct structure  
- [ ] Application starts successfully with `docker compose up -d`  
- [ ] All 4 API endpoints work correctly (tested using CURL/postman/browser)  
- [ ] Data persists after stopping and restarting containers

### Testing Your Submission

**Test your project before turning it in:**

**\# 1\. Clone your repository**

git clone \<your-repo-url\> test\_submission

cd test\_submission

**\# 2\. Start the application**

docker compose up \-d

**\# 3\. Wait 30 seconds for database initialization**

sleep 30

**\# 4\. Test GET all contacts**

curl http://localhost:8000/contacts

**\# 5\. Test CREATE contact**

curl \-X POST http://localhost:8000/contacts \\

  \-H "Content-Type: application/json" \\

  \-d '{"first\_name":"Test","last\_name":"User","phone\_number":"050-9999999"}'

**\# 6\. Test UPDATE contact (use actual ID from previous response)**

curl \-X PUT http://localhost:8000/contacts/4 \\

  \-H "Content-Type: application/json" \\

  \-d '{"phone\_number":"052-8888888"}'

**\# 7\. Test DELETE contact**

curl \-X DELETE http://localhost:8000/contacts/4

**\# 8\. Test data persistence**

docker compose down

docker compose up \-d

sleep 30

curl http://localhost:8000/contacts  \# Should show original contacts

**\# 9\. Clean up**

docker compose down \-v

**If any test fails, fix the issue before submitting.**

## Recommended Timeline

### Day 1 (6 hours)

**Hours 1-2: Setup and Database**

- Create GitHub repository  
- Set up project structure  
- Write database schema and init.sql  
- Create database.py module

**Hours 3-4: Contact Class and CRUD**

- Implement Contact class  
- Implement create\_contact and get\_all\_contacts functions  
- Test database operations manually

**Hours 5-6: First API Endpoints**

- Create requirements.txt  
- Implement FastAPI application structure  
- Implement GET /contacts endpoint  
- Implement POST /contacts endpoint  
- Test with curl

### Day 2 (6 hours)

**Hours 1-2: Complete API**

- Implement PUT /contacts/{id} endpoint  
- Implement DELETE /contacts/{id} endpoint  
- Test all endpoints

**Hours 3-4: Dockerization**

- Create Dockerfile  
- Test building Docker image  
- Create compose.yaml  
- Test running with Docker Compose

**Hours 5-6: Testing and Documentation**

- Test all functionality  
- Test data persistence  
- Complete README.md  
- Final testing and cleanup

## Resources

### Recommended Material

- FastAPI Basics: [https://fastapi.tiangolo.com/tutorial/first-steps/](https://fastapi.tiangolo.com/tutorial/first-steps/)  
- MySQL Connector Python: [https://dev.mysql.com/doc/connector-python/en/](https://dev.mysql.com/doc/connector-python/en/)  
- Docker Compose: [https://docs.docker.com/compose/](https://docs.docker.com/compose/)

### Helpful References

- SQL Query Syntax: [https://www.w3schools.com/sql/](https://www.w3schools.com/sql/)  
- Pydantic Models: [https://docs.pydantic.dev/latest/](https://docs.pydantic.dev/latest/)  
- Docker Volumes: [https://docs.docker.com/storage/volumes/](https://docs.docker.com/storage/volumes/)

### Example Dataset

Sample data for testing (include in your init.sql):

INSERT INTO contacts (first\_name, last\_name, phone\_number) VALUES

    ('John', 'Doe', '050-1234567'),  
    ('Jane', 'Smith', '052-9876543'),  
    ('Bob', 'Johnson', '054-5555555'),  
    ('Jack', 'Robinson', '050-6115555');

## Frequently Asked Questions

**Q: Can I use an ORM like SQLAlchemy?**  
A: No. This project requires manual SQL queries to demonstrate understanding of database operations.

**Q: Do I need to use async/await in FastAPI?**  
A: No. Regular synchronous functions are acceptable for this project.

**Q: What if my database takes time to initialize?**  
A: Wait 30 seconds after `docker compose up` before testing. This is normal behavior.

**Q: Can I add extra endpoints?**  
A: Yes, But ensure the 4 required endpoints work first.

**Q: How do I test if my volume is working?**  
A: Create a contact, run `docker compose down` (without \-v), then `docker compose up -d`. The contact should still exist.

**Q: Should I include a .env file in Git?**  
A: No\! Add .env to .gitignore. You can include a .env.example with dummy values.

## Important Notes

### Code Quality Expectations

- Use meaningful variable and function names  
- Add comments explaining complex logic where needed  
- Keep functions focused and single-purpose  
- Handle errors gracefully

### Common Mistakes to Avoid

❌ Using `localhost` as database host in Docker (use service name `db`)  
❌ Forgetting to commit regularly  
❌ Not testing before submission  
❌ Skipping error handling  
❌ Not waiting for database initialization

✅ Test each component before moving forward  
✅ **Commit after each working component**  
✅ Use service names for container communication  
✅ Include sample data in init.sql

Good luck\! Remember: start early, commit often, test frequently.