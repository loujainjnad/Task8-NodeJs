# Task Management System API Design

A complete backend system for a collaborative task management application that helps users organize their daily tasks, projects, and collaborate with team members.

##Table of Contents

- [Project Structure](#project-structure)
- [Required Libraries](#required-libraries)
- [Database Models](#database-models)
- [API Endpoints](#api-endpoints)
- [Middleware](#middleware)
- [Authentication](#authentication)
- [Error Handling](#error-handling)

---

## Project Structure

```
Task8-loujainjnad/
│
├── src/
│   ├── config/
│   │   ├── database.js          # MongoDB connection configuration
│   │   └── jwt.js               # JWT secret and configuration
│   │
│   ├── controllers/
│   │   ├── authController.js    # Authentication logic (register, login)
│   │   ├── taskController.js    # Task CRUD operations
│   │   ├── projectController.js # Project management operations
│   │   ├── userController.js    # User profile and statistics
│   │   └── notificationController.js # Notification handling
│   │
│   ├── models/
│   │   ├── User.js              # User schema
│   │   ├── Task.js              # Task schema
│   │   ├── Project.js           # Project schema
│   │   ├── ProjectInvite.js     # Project invitation schema
│   │   └── Notification.js      # Notification schema
│   │
│   ├── middleware/
│   │   ├── auth.js              # JWT authentication middleware
│   │   ├── errorHandler.js      # Global error handling middleware
│   │   ├── validate.js          # Request validation middleware
│   │   └── asyncHandler.js      # Async error wrapper
│   │
│   ├── routes/
│   │   ├── authRoutes.js        # Authentication routes
│   │   ├── taskRoutes.js        # Task routes
│   │   ├── projectRoutes.js     # Project routes
│   │   ├── inviteRoutes.js      # Project invitation routes
│   │   ├── userRoutes.js        # User routes
│   │   └── notificationRoutes.js # Notification routes
│   │
│   ├── utils/
│   │   ├── validators.js        # Input validation functions
│   │   ├── helpers.js           # Helper functions
│   │   ├── constants.js         # Application constants
│   │   └── scheduledJobs.js     # Cron jobs for notifications
│   │
│   ├── app.js                   # Express app configuration
│   └── server.js                # Server entry point
│
├── .env                         # Environment variables
├── .gitignore                   # Git ignore file
├── package.json                 # Dependencies and scripts
└── README.md                    # Project documentation
```

---

## Required Libraries

### Core Dependencies

#### **express** (^4.18.2)
- **Purpose**: Web framework for Node.js to build RESTful APIs
- **Why needed**: Provides routing, middleware support, and HTTP server functionality

#### **mongoose** (^7.5.0)
- **Purpose**: MongoDB object modeling tool for Node.js
- **Why needed**: Simplifies database operations, provides schema validation, and handles relationships between collections

#### **jsonwebtoken** (^9.0.2)
- **Purpose**: Implementation of JSON Web Tokens for authentication
- **Why needed**: Securely authenticate users and protect API endpoints without server-side sessions

#### **bcryptjs** (^2.4.3)
- **Purpose**: Library for hashing passwords
- **Why needed**: Securely store user passwords by hashing them before saving to database (never store plain text passwords)

#### **dotenv** (^16.3.1)
- **Purpose**: Loads environment variables from .env file
- **Why needed**: Securely manage sensitive configuration like database URLs, JWT secrets, and API keys

#### **express-validator** (^7.0.1)
- **Purpose**: Set of express.js middlewares for input validation and sanitization
- **Why needed**: Validate and sanitize user input to prevent invalid data and security vulnerabilities

#### **cors** (^2.8.5)
- **Purpose**: Enable Cross-Origin Resource Sharing
- **Why needed**: Allow frontend applications from different origins to access the API

#### **helmet** (^7.0.0)
- **Purpose**: Security middleware for Express
- **Why needed**: Set various HTTP headers to protect the app from common security vulnerabilities

#### **morgan** (^1.10.0)
- **Purpose**: HTTP request logger middleware
- **Why needed**: Log HTTP requests for debugging and monitoring purposes

#### **node-cron** (^3.0.2)
- **Purpose**: Task scheduler for Node.js
- **Why needed**: Schedule and run background jobs for task reminders and deadline notifications

### Development Dependencies

#### **nodemon** (^3.0.1)
- **Purpose**: Automatically restart Node.js application on file changes
- **Why needed**: Improve development experience by eliminating manual server restarts

---

## Database Models

### User Model

```javascript
{
  name: {
    type: String,
    required: [true, "Name is required"],
    trim: true,
    minlength: [2, "Name must be at least 2 characters"]
  },
  email: {
    type: String,
    required: [true, "Email is required"],
    unique: true,
    lowercase: true,
    trim: true,
    match: [/^\S+@\S+\.\S+$/, "Please provide a valid email"]
  },
  password: {
    type: String,
    required: [true, "Password is required"],
    minlength: [6, "Password must be at least 6 characters"],
    select: false // Don't return password in queries by default
  },
  avatar: {
    type: String,
    default: null
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
}
```

**Relationships:**
- One-to-Many with Tasks (user can have many tasks)
- Many-to-Many with Projects (user can be member of many projects)

---

### Task Model

```javascript
{
  title: {
    type: String,
    required: [true, "Task title is required"],
    trim: true,
    maxlength: [200, "Title cannot exceed 200 characters"]
  },
  description: {
    type: String,
    trim: true,
    maxlength: [1000, "Description cannot exceed 1000 characters"]
  },
  status: {
    type: String,
    enum: ["todo", "in progress", "done"],
    default: "todo"
  },
  priority: {
    type: String,
    enum: ["low", "medium", "high"],
    default: "medium"
  },
  dueDate: {
    type: Date,
    default: null
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: "User",
    default: null
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: "User",
    required: true
  },
  project: {
    type: mongoose.Schema.Types.ObjectId,
    ref: "Project",
    default: null
  },
  reminder: {
    type: Date,
    default: null
  },
  completedAt: {
    type: Date,
    default: null
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: {
    type: Date,
    default: Date.now
  }
}
```

**Relationships:**
- Many-to-One with User (assignedTo, createdBy)
- Many-to-One with Project (optional)

---

### Project Model

```javascript
{
  name: {
    type: String,
    required: [true, "Project name is required"],
    trim: true,
    maxlength: [100, "Project name cannot exceed 100 characters"]
  },
  description: {
    type: String,
    trim: true,
    maxlength: [500, "Description cannot exceed 500 characters"]
  },
  owner: {
    type: mongoose.Schema.Types.ObjectId,
    ref: "User",
    required: true
  },
  members: [{
    type: mongoose.Schema.Types.ObjectId,
    ref: "User"
  }],
  status: {
    type: String,
    enum: ["active", "archived", "completed"],
    default: "active"
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: {
    type: Date,
    default: Date.now
  }
}
```

**Relationships:**
- One-to-Many with User (owner)
- Many-to-Many with User (members array)
- One-to-Many with Tasks
- One-to-Many with ProjectInvites

---

### ProjectInvite Model

```javascript
{
  project: {
    type: mongoose.Schema.Types.ObjectId,
    ref: "Project",
    required: true
  },
  email: {
    type: String,
    required: [true, "Email is required"],
    lowercase: true,
    trim: true,
    match: [/^\S+@\S+\.\S+$/, "Please provide a valid email"]
  },
  invitedBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: "User",
    required: true
  },
  token: {
    type: String,
    required: true,
    unique: true
  },
  status: {
    type: String,
    enum: ["pending", "accepted", "rejected", "expired"],
    default: "pending"
  },
  expiresAt: {
    type: Date,
    required: true,
    default: () => new Date(Date.now() + 7 * 24 * 60 * 60 * 1000) // 7 days
  },
  acceptedAt: {
    type: Date,
    default: null
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
}
```

**Relationships:**
- Many-to-One with Project
- Many-to-One with User (invitedBy)
- One-to-One with User (when accepted, links to user account)

**Indexes:**
- `token` (unique)
- `email` and `project` (compound index for preventing duplicate invites)
- `expiresAt` (for cleanup of expired invites)

---

### Notification Model

```javascript
{
  user: {
    type: mongoose.Schema.Types.ObjectId,
    ref: "User",
    required: true
  },
  type: {
    type: String,
    enum: ["task_assigned", "task_due", "task_reminder", "project_invite", "task_completed"],
    required: true
  },
  title: {
    type: String,
    required: true
  },
  message: {
    type: String,
    required: true
  },
  relatedTask: {
    type: mongoose.Schema.Types.ObjectId,
    ref: "Task",
    default: null
  },
  relatedProject: {
    type: mongoose.Schema.Types.ObjectId,
    ref: "Project",
    default: null
  },
  read: {
    type: Boolean,
    default: false
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
}
```

**Relationships:**
- Many-to-One with User
- Many-to-One with Task (optional)
- Many-to-One with Project (optional)

---

## API Endpoints

### Authentication Endpoints

## POST /api/auth/register

- Method: POST
- Description: Create a new user account
- Authentication: Not required
- Request Body: 
  - name (string, required, minimum 2 characters)
  - email (string, required, must be unique, valid email format)
  - password (string, required, minimum 6 characters)
- Response Success (201): 
  ```json
  {
    "success": true,
    "message": "User registered successfully",
    "data": {
      "user": {
        "_id": "...",
        "name": "...",
        "email": "...",
        "createdAt": "..."
      },
      "token": "jwt_token_here"
    }
  }
  ```
- Response Errors:
  - 409: `{ "success": false, "error": "Email already exists" }`
  - 400: `{ "success": false, "error": "Password must be at least 6 characters" }`
  - 400: `{ "success": false, "error": "All fields are required" }`
  - 400: `{ "success": false, "error": "Please provide a valid email" }`
  - 500: `{ "success": false, "error": "Server error" }`
- Potential User Mistakes:
  - Submitting without filling all required fields
  - Using an email that already exists (returns 409 Conflict)
  - Entering a weak password (less than 6 characters)
  - Providing invalid email format

---

## POST /api/auth/login

- Method: POST
- Description: Authenticate user and receive JWT token
- Authentication: Not required
- Request Body: 
  - email (string, required)
  - password (string, required)
- Response Success (200): 
  ```json
  {
    "success": true,
    "message": "Login successful",
    "data": {
      "user": {
        "_id": "...",
        "name": "...",
        "email": "..."
      },
      "token": "jwt_token_here"
    }
  }
  ```
- Response Errors:
  - 400: `{ "success": false, "error": "Email and password are required" }`
  - 401: `{ "success": false, "error": "Invalid credentials" }`
  - 500: `{ "success": false, "error": "Server error" }`
- Potential User Mistakes:
  - Forgetting to provide email or password
  - Using incorrect email
  - Using incorrect password
  - Submitting empty fields

---

## GET /api/auth/me

- Method: GET
- Description: Get current authenticated user profile
- Authentication: Required (JWT token in Authorization header)
- Request Headers:
  - Authorization: Bearer {token}
- Response Success (200): 
  ```json
  {
    "success": true,
    "data": {
      "_id": "...",
      "name": "...",
      "email": "...",
      "createdAt": "..."
    }
  }
  ```
- Response Errors:
  - 401: `{ "success": false, "error": "Authentication required" }`
  - 401: `{ "success": false, "error": "Invalid or expired token" }`
  - 404: `{ "success": false, "error": "User not found" }`
- Potential User Mistakes:
  - Forgetting to include authentication token
  - Using expired token
  - Using invalid token format

---

### Task Endpoints

## GET /api/tasks

- Method: GET
- Description: Get all tasks for the logged-in user (tasks created by user or assigned to user)
- Authentication: Required (JWT token in Authorization header)
- Query Parameters:
  - status (string, optional: 'todo', 'in progress', 'done')
  - project (string, optional: project ID)
  - priority (string, optional: 'low', 'medium', 'high')
  - search (string, optional: search in task title and description)
  - assignedTo (string, optional: 'me' to filter tasks assigned to current user)
  - sortBy (string, optional: 'createdAt', 'dueDate', 'priority', default: 'createdAt')
  - sortOrder (string, optional: 'asc', 'desc', default: 'desc')
- Request Headers:
  - Authorization: Bearer {token}
- Response Success (200):
  ```json
  {
    "success": true,
    "count": 10,
    "data": [
      {
        "_id": "...",
        "title": "...",
        "description": "...",
        "status": "todo",
        "priority": "high",
        "dueDate": "...",
        "assignedTo": { "name": "...", "email": "..." },
        "createdBy": { "name": "...", "email": "..." },
        "project": { "name": "...", "_id": "..." },
        "reminder": "...",
        "createdAt": "...",
        "updatedAt": "..."
      }
    ]
  }
  ```
- Response Errors:
  - 401: `{ "success": false, "error": "Authentication required" }`
  - 500: `{ "success": false, "error": "Server error" }`
- Potential User Mistakes:
  - Forgetting to include authentication token
  - Using invalid status filter value
  - Using invalid project ID
  - Searching with empty query

---

## GET /api/tasks/:id

- Method: GET
- Description: Get a single task by ID (user must be creator or assigned to the task)
- Authentication: Required (JWT token in Authorization header)
- URL Parameters:
  - id (string, required: task ID)
- Request Headers:
  - Authorization: Bearer {token}
- Response Success (200):
  ```json
  {
    "success": true,
    "data": {
      "_id": "...",
      "title": "...",
      "description": "...",
      "status": "in progress",
      "priority": "high",
      "dueDate": "...",
      "assignedTo": { "name": "...", "email": "..." },
      "createdBy": { "name": "...", "email": "..." },
      "project": { "name": "...", "_id": "..." },
      "reminder": "...",
      "completedAt": null,
      "createdAt": "...",
      "updatedAt": "..."
    }
  }
  ```
- Response Errors:
  - 401: `{ "success": false, "error": "Authentication required" }`
  - 403: `{ "success": false, "error": "Not authorized to access this task" }`
  - 404: `{ "success": false, "error": "Task not found" }`
  - 400: `{ "success": false, "error": "Invalid task ID" }`
- Potential User Mistakes:
  - Using invalid task ID format
  - Trying to access task that doesn't belong to user
  - Forgetting authentication token

---

## POST /api/tasks

- Method: POST
- Description: Create a new task
- Authentication: Required (JWT token in Authorization header)
- Request Body: 
  - title (string, required, max 200 characters)
  - description (string, optional, max 1000 characters)
  - status (string, optional: 'todo', 'in progress', 'done', default: 'todo')
  - priority (string, optional: 'low', 'medium', 'high', default: 'medium')
  - dueDate (date, optional, ISO format)
  - assignedTo (string, optional: user ID)
  - project (string, optional: project ID)
  - reminder (date, optional, ISO format)
- Request Headers:
  - Authorization: Bearer {token}
- Response Success (201):
  ```json
  {
    "success": true,
    "message": "Task created successfully",
    "data": {
      "_id": "...",
      "title": "...",
      "description": "...",
      "status": "todo",
      "priority": "medium",
      "dueDate": null,
      "assignedTo": null,
      "createdBy": "...",
      "project": null,
      "reminder": null,
      "createdAt": "...",
      "updatedAt": "..."
    }
  }
  ```
- **Notification Creation**: 
  - If `assignedTo` is provided, a notification of type `task_assigned` is automatically created for the assigned user
  - Notification includes task details and assignment information
- Response Errors:
  - 400: `{ "success": false, "error": "Task title is required" }`
  - 400: `{ "success": false, "error": "Invalid status value" }`
  - 400: `{ "success": false, "error": "Invalid priority value" }`
  - 400: `{ "success": false, "error": "Invalid user ID for assignment" }`
  - 400: `{ "success": false, "error": "Invalid project ID" }`
  - 401: `{ "success": false, "error": "Authentication required" }`
  - 500: `{ "success": false, "error": "Server error" }`
- Potential User Mistakes:
  - Creating task without title
  - Using invalid status or priority values
  - Assigning task to non-existent user
  - Using invalid date format for dueDate or reminder
  - Assigning to project user is not a member of

---

## PUT /api/tasks/:id

- Method: PUT
- Description: Update an existing task (user must be creator or assigned to the task)
- Authentication: Required (JWT token in Authorization header)
- URL Parameters:
  - id (string, required: task ID)
- Request Body (all fields optional): 
  - title (string, max 200 characters)
  - description (string, max 1000 characters)
  - status (string: 'todo', 'in progress', 'done')
  - priority (string: 'low', 'medium', 'high')
  - dueDate (date, ISO format)
  - assignedTo (string: user ID)
  - project (string: project ID)
  - reminder (date, ISO format)
- Request Headers:
  - Authorization: Bearer {token}
- Response Success (200):
  ```json
  {
    "success": true,
    "message": "Task updated successfully",
    "data": {
      "_id": "...",
      "title": "...",
      "status": "done",
      "completedAt": "...",
      "updatedAt": "..."
    }
  }
  ```
- **Notification Creation**: 
  - If `assignedTo` is changed, a notification of type `task_assigned` is created for the newly assigned user
  - If status is changed to `done`, a notification of type `task_completed` is created for the task creator
  - If `dueDate` is set or updated, the system schedules a deadline notification (handled by cron job)
- Response Errors:
  - 400: `{ "success": false, "error": "Invalid status value" }`
  - 400: `{ "success": false, "error": "Invalid task ID" }`
  - 401: `{ "success": false, "error": "Authentication required" }`
  - 403: `{ "success": false, "error": "Not authorized to update this task" }`
  - 404: `{ "success": false, "error": "Task not found" }`
  - 500: `{ "success": false, "error": "Server error" }`
- Potential User Mistakes:
  - Trying to update task without permission
  - Using invalid status or priority values
  - Providing invalid date formats
  - Updating with empty body (no changes)

---

## DELETE /api/tasks/:id

- Method: DELETE
- Description: Delete a task (only creator can delete)
- Authentication: Required (JWT token in Authorization header)
- URL Parameters:
  - id (string, required: task ID)
- Request Headers:
  - Authorization: Bearer {token}
- Response Success (200):
  ```json
  {
    "success": true,
    "message": "Task deleted successfully"
  }
  ```
- Response Errors:
  - 400: `{ "success": false, "error": "Invalid task ID" }`
  - 401: `{ "success": false, "error": "Authentication required" }`
  - 403: `{ "success": false, "error": "Not authorized to delete this task" }`
  - 404: `{ "success": false, "error": "Task not found" }`
  - 500: `{ "success": false, "error": "Server error" }`
- Potential User Mistakes:
  - Trying to delete task created by someone else
  - Using invalid task ID
  - Forgetting authentication token

---

### Project Endpoints

## GET /api/projects

- Method: GET
- Description: Get all projects where user is owner or member
- Authentication: Required (JWT token in Authorization header)
- Query Parameters:
  - status (string, optional: 'active', 'archived', 'completed')
  - search (string, optional: search in project name)
- Request Headers:
  - Authorization: Bearer {token}
- Response Success (200):
  ```json
  {
    "success": true,
    "count": 5,
    "data": [
      {
        "_id": "...",
        "name": "...",
        "description": "...",
        "owner": { "name": "...", "email": "..." },
        "members": [{ "name": "...", "email": "..." }],
        "status": "active",
        "createdAt": "...",
        "updatedAt": "..."
      }
    ]
  }
  ```
- Response Errors:
  - 401: `{ "success": false, "error": "Authentication required" }`
  - 500: `{ "success": false, "error": "Server error" }`
- Potential User Mistakes:
  - Forgetting authentication token
  - Using invalid status filter

---

## GET /api/projects/:id

- Method: GET
- Description: Get a single project by ID (user must be owner or member)
- Authentication: Required (JWT token in Authorization header)
- URL Parameters:
  - id (string, required: project ID)
- Request Headers:
  - Authorization: Bearer {token}
- Response Success (200):
  ```json
  {
    "success": true,
    "data": {
      "_id": "...",
      "name": "...",
      "description": "...",
      "owner": { "name": "...", "email": "..." },
      "members": [{ "name": "...", "email": "..." }],
      "status": "active",
      "createdAt": "...",
      "updatedAt": "..."
    }
  }
  ```
- Response Errors:
  - 400: `{ "success": false, "error": "Invalid project ID" }`
  - 401: `{ "success": false, "error": "Authentication required" }`
  - 403: `{ "success": false, "error": "Not authorized to access this project" }`
  - 404: `{ "success": false, "error": "Project not found" }`
- Potential User Mistakes:
  - Using invalid project ID
  - Trying to access project user is not part of

---

## POST /api/projects

- Method: POST
- Description: Create a new project
- Authentication: Required (JWT token in Authorization header)
- Request Body: 
  - name (string, required, max 100 characters)
  - description (string, optional, max 500 characters)
  - status (string, optional: 'active', 'archived', 'completed', default: 'active')
- Request Headers:
  - Authorization: Bearer {token}
- Response Success (201):
  ```json
  {
    "success": true,
    "message": "Project created successfully",
    "data": {
      "_id": "...",
      "name": "...",
      "description": "...",
      "owner": "...",
      "members": [],
      "status": "active",
      "createdAt": "...",
      "updatedAt": "..."
    }
  }
  ```
- Response Errors:
  - 400: `{ "success": false, "error": "Project name is required" }`
  - 400: `{ "success": false, "error": "Invalid status value" }`
  - 401: `{ "success": false, "error": "Authentication required" }`
  - 500: `{ "success": false, "error": "Server error" }`
- Potential User Mistakes:
  - Creating project without name
  - Using invalid status value
  - Exceeding character limits

---

## PUT /api/projects/:id

- Method: PUT
- Description: Update a project (only owner can update)
- Authentication: Required (JWT token in Authorization header)
- URL Parameters:
  - id (string, required: project ID)
- Request Body (all fields optional): 
  - name (string, max 100 characters)
  - description (string, max 500 characters)
  - status (string: 'active', 'archived', 'completed')
- Request Headers:
  - Authorization: Bearer {token}
- Response Success (200):
  ```json
  {
    "success": true,
    "message": "Project updated successfully",
    "data": {
      "_id": "...",
      "name": "...",
      "status": "active",
      "updatedAt": "..."
    }
  }
  ```
- Response Errors:
  - 400: `{ "success": false, "error": "Invalid project ID" }`
  - 400: `{ "success": false, "error": "Invalid status value" }`
  - 401: `{ "success": false, "error": "Authentication required" }`
  - 403: `{ "success": false, "error": "Only project owner can update project" }`
  - 404: `{ "success": false, "error": "Project not found" }`
- Potential User Mistakes:
  - Trying to update project user doesn't own
  - Using invalid status value

---

## DELETE /api/projects/:id

- Method: DELETE
- Description: Delete a project (only owner can delete)
- Authentication: Required (JWT token in Authorization header)
- URL Parameters:
  - id (string, required: project ID)
- Request Headers:
  - Authorization: Bearer {token}
- Response Success (200):
  ```json
  {
    "success": true,
    "message": "Project deleted successfully"
  }
  ```
- Response Errors:
  - 400: `{ "success": false, "error": "Invalid project ID" }`
  - 401: `{ "success": false, "error": "Authentication required" }`
  - 403: `{ "success": false, "error": "Only project owner can delete project" }`
  - 404: `{ "success": false, "error": "Project not found" }`
- Potential User Mistakes:
  - Trying to delete project user doesn't own
  - Using invalid project ID

---

## POST /api/projects/:id/invite

- Method: POST
- Description: Invite a team member to a project by email (only owner can invite). Creates a ProjectInvite record with a unique token that can be used to accept the invitation. Works for both registered and unregistered users.
- Authentication: Required (JWT token in Authorization header)
- URL Parameters:
  - id (string, required: project ID)
- Request Body: 
  - email (string, required, valid email format)
- Request Headers:
  - Authorization: Bearer {token}
- Response Success (201):
  ```json
  {
    "success": true,
    "message": "Invitation sent successfully",
    "data": {
      "project": { "_id": "...", "name": "..." },
      "invite": {
        "_id": "...",
        "email": "...",
        "token": "unique_invite_token",
        "status": "pending",
        "expiresAt": "...",
        "inviteLink": "http://localhost:5000/api/projects/invites/accept?token=unique_invite_token"
      }
    }
  }
  ```
- Response Errors:
  - 400: `{ "success": false, "error": "Email is required" }`
  - 400: `{ "success": false, "error": "Invalid email format" }`
  - 400: `{ "success": false, "error": "Invalid project ID" }`
  - 401: `{ "success": false, "error": "Authentication required" }`
  - 403: `{ "success": false, "error": "Only project owner can invite members" }`
  - 404: `{ "success": false, "error": "Project not found" }`
  - 409: `{ "success": false, "error": "User is already a member of this project" }`
  - 409: `{ "success": false, "error": "Invitation already sent to this email" }`
- **Notification Creation**: 
  - If the invited email belongs to a registered user, a notification of type `project_invite` is automatically created
  - If the email doesn't belong to a registered user, an invitation email should be sent (future enhancement)
- Potential User Mistakes:
  - Inviting user that is already a member
  - Using invalid email format
  - Trying to invite as non-owner
  - Sending duplicate invitation to the same email

---

## GET /api/projects/invites/accept

- Method: GET
- Description: Accept a project invitation using the invitation token. If user is not logged in, they will be redirected to registration/login. If user is logged in, they are added to the project immediately.
- Authentication: Optional (if user is logged in, they are added directly; if not, token is validated and user can register/login)
- Query Parameters:
  - token (string, required: invitation token from ProjectInvite)
- Response Success (200) - User logged in:
  ```json
  {
    "success": true,
    "message": "Invitation accepted successfully",
    "data": {
      "project": {
        "_id": "...",
        "name": "...",
        "description": "..."
      },
      "user": {
        "_id": "...",
        "name": "...",
        "email": "..."
      }
    }
  }
  ```
- Response Success (200) - User not logged in:
  ```json
  {
    "success": true,
    "message": "Invitation token is valid. Please register or login to accept.",
    "data": {
      "invite": {
        "email": "...",
        "project": {
          "_id": "...",
          "name": "..."
        }
      },
      "requiresAuth": true
    }
  }
  ```
- Response Errors:
  - 400: `{ "success": false, "error": "Invitation token is required" }`
  - 404: `{ "success": false, "error": "Invitation not found" }`
  - 400: `{ "success": false, "error": "Invitation has expired" }`
  - 400: `{ "success": false, "error": "Invitation has already been accepted" }`
  - 400: `{ "success": false, "error": "Invitation has been rejected" }`
  - 409: `{ "success": false, "error": "User is already a member of this project" }`
- Potential User Mistakes:
  - Using expired invitation token
  - Using already accepted/rejected token
  - Using invalid token format

---

## POST /api/projects/invites/accept

- Method: POST
- Description: Accept a project invitation (for authenticated users). Alternative endpoint that accepts token in request body.
- Authentication: Required (JWT token in Authorization header)
- Request Body: 
  - token (string, required: invitation token)
- Request Headers:
  - Authorization: Bearer {token}
- Response Success (200):
  ```json
  {
    "success": true,
    "message": "Invitation accepted successfully",
    "data": {
      "project": {
        "_id": "...",
        "name": "...",
        "description": "..."
      }
    }
  }
  ```
- Response Errors:
  - 400: `{ "success": false, "error": "Invitation token is required" }`
  - 401: `{ "success": false, "error": "Authentication required" }`
  - 404: `{ "success": false, "error": "Invitation not found" }`
  - 400: `{ "success": false, "error": "Invitation has expired" }`
  - 400: `{ "success": false, "error": "Invitation has already been accepted" }`
  - 400: `{ "success": false, "error": "Email in invitation does not match your account" }`
  - 409: `{ "success": false, "error": "User is already a member of this project" }`
- Potential User Mistakes:
  - Using invitation token for different email than logged-in user
  - Using expired or already processed token

---

## POST /api/projects/invites/reject

- Method: POST
- Description: Reject a project invitation
- Authentication: Required (JWT token in Authorization header)
- Request Body: 
  - token (string, required: invitation token)
- Request Headers:
  - Authorization: Bearer {token}
- Response Success (200):
  ```json
  {
    "success": true,
    "message": "Invitation rejected successfully"
  }
  ```
- Response Errors:
  - 400: `{ "success": false, "error": "Invitation token is required" }`
  - 401: `{ "success": false, "error": "Authentication required" }`
  - 404: `{ "success": false, "error": "Invitation not found" }`
  - 400: `{ "success": false, "error": "Invitation has already been processed" }`
- Potential User Mistakes:
  - Using already processed token
  - Using invalid token

---

## DELETE /api/projects/:id/members/:memberId

- Method: DELETE
- Description: Remove a member from a project (only owner can remove members)
- Authentication: Required (JWT token in Authorization header)
- URL Parameters:
  - id (string, required: project ID)
  - memberId (string, required: member user ID)
- Request Headers:
  - Authorization: Bearer {token}
- Response Success (200):
  ```json
  {
    "success": true,
    "message": "Member removed from project successfully"
  }
  ```
- Response Errors:
  - 400: `{ "success": false, "error": "Invalid project ID" }`
  - 400: `{ "success": false, "error": "Invalid member ID" }`
  - 401: `{ "success": false, "error": "Authentication required" }`
  - 403: `{ "success": false, "error": "Only project owner can remove members" }`
  - 404: `{ "success": false, "error": "Project not found" }`
  - 404: `{ "success": false, "error": "Member not found in project" }`
- Potential User Mistakes:
  - Trying to remove owner from project
  - Using invalid member ID
  - Trying to remove as non-owner

---

### User Endpoints

## GET /api/users/me/tasks

- Method: GET
- Description: Get all tasks assigned to the current user
- Authentication: Required (JWT token in Authorization header)
- Query Parameters:
  - status (string, optional: 'todo', 'in progress', 'done')
  - project (string, optional: project ID)
- Request Headers:
  - Authorization: Bearer {token}
- Response Success (200):
  ```json
  {
    "success": true,
    "count": 8,
    "data": [
      {
        "_id": "...",
        "title": "...",
        "status": "todo",
        "dueDate": "...",
        "project": { "name": "...", "_id": "..." }
      }
    ]
  }
  ```
- Response Errors:
  - 401: `{ "success": false, "error": "Authentication required" }`
  - 500: `{ "success": false, "error": "Server error" }`
- Potential User Mistakes:
  - Forgetting authentication token

---

## GET /api/users/me/statistics

- Method: GET
- Description: Get task completion statistics for the current user
- Authentication: Required (JWT token in Authorization header)
- Request Headers:
  - Authorization: Bearer {token}
- Response Success (200):
  ```json
  {
    "success": true,
    "data": {
      "totalTasks": 25,
      "completedTasks": 10,
      "inProgressTasks": 5,
      "todoTasks": 10,
      "completionRate": 40,
      "tasksByPriority": {
        "high": 8,
        "medium": 12,
        "low": 5
      },
      "overdueTasks": 3
    }
  }
  ```
- Response Errors:
  - 401: `{ "success": false, "error": "Authentication required" }`
  - 500: `{ "success": false, "error": "Server error" }`
- Potential User Mistakes:
  - Forgetting authentication token

---

### Notification Endpoints

## GET /api/notifications

- Method: GET
- Description: Get all notifications for the current user
- Authentication: Required (JWT token in Authorization header)
- Query Parameters:
  - read (boolean, optional: filter by read status)
  - type (string, optional: filter by notification type)
- Request Headers:
  - Authorization: Bearer {token}
- Response Success (200):
  ```json
  {
    "success": true,
    "count": 15,
    "data": [
      {
        "_id": "...",
        "type": "task_assigned",
        "title": "...",
        "message": "...",
        "relatedTask": { "_id": "...", "title": "..." },
        "read": false,
        "createdAt": "..."
      }
    ]
  }
  ```
- Response Errors:
  - 401: `{ "success": false, "error": "Authentication required" }`
  - 500: `{ "success": false, "error": "Server error" }`
- Potential User Mistakes:
  - Forgetting authentication token

---

## PUT /api/notifications/:id/read

- Method: PUT
- Description: Mark a notification as read
- Authentication: Required (JWT token in Authorization header)
- URL Parameters:
  - id (string, required: notification ID)
- Request Headers:
  - Authorization: Bearer {token}
- Response Success (200):
  ```json
  {
    "success": true,
    "message": "Notification marked as read",
    "data": {
      "_id": "...",
      "read": true
    }
  }
  ```
- Response Errors:
  - 400: `{ "success": false, "error": "Invalid notification ID" }`
  - 401: `{ "success": false, "error": "Authentication required" }`
  - 403: `{ "success": false, "error": "Not authorized to access this notification" }`
  - 404: `{ "success": false, "error": "Notification not found" }`
- Potential User Mistakes:
  - Using invalid notification ID
  - Trying to mark notification that doesn't belong to user

---

## PUT /api/notifications/read-all

- Method: PUT
- Description: Mark all notifications as read for the current user
- Authentication: Required (JWT token in Authorization header)
- Request Headers:
  - Authorization: Bearer {token}
- Response Success (200):
  ```json
  {
    "success": true,
    "message": "All notifications marked as read",
    "data": {
      "updatedCount": 10
    }
  }
  ```
- Response Errors:
  - 401: `{ "success": false, "error": "Authentication required" }`
  - 500: `{ "success": false, "error": "Server error" }`
- Potential User Mistakes:
  - Forgetting authentication token

---

## Middleware

### Authentication Middleware (`auth.js`)

**Purpose**: Verify JWT token and attach user information to request object

**Functionality**:
- Extracts JWT token from Authorization header (Bearer token format)
- Verifies token signature and expiration
- Retrieves user from database using token payload
- Attaches user object to `req.user`
- Returns 401 error if token is missing, invalid, or expired

**Usage**: Applied to protected routes that require authentication

---

### Error Handler Middleware (`errorHandler.js`)

**Purpose**: Centralized error handling for all routes

**Functionality**:
- Catches all errors thrown in controllers
- Formats error responses consistently
- Handles different error types:
  - Validation errors (400)
  - Authentication errors (401)
  - Authorization errors (403)
  - Not found errors (404)
  - Server errors (500)
- Logs errors for debugging
- Returns user-friendly error messages

**Usage**: Applied as the last middleware in Express app

---

### Validation Middleware (`validate.js`)

**Purpose**: Validate and sanitize request data using express-validator

**Functionality**:
- Validates request body, query parameters, and URL parameters
- Checks required fields
- Validates data types and formats (email, date, etc.)
- Enforces length constraints
- Sanitizes input to prevent injection attacks
- Returns validation errors if data is invalid

**Usage**: Applied to routes that accept user input

---

### Async Handler Middleware (`asyncHandler.js`)

**Purpose**: Wrap async route handlers to automatically catch errors

**Functionality**:
- Wraps async functions in try-catch block
- Automatically passes errors to error handler middleware
- Eliminates need for manual try-catch in every controller

**Usage**: Wraps all async controller functions

---

### Authorization Middleware

**Purpose**: Check if user has permission to perform specific actions

**Functionality**:
- Task ownership check: Verify user created the task
- Task assignment check: Verify user is assigned to or created the task
- Project ownership check: Verify user owns the project
- Project membership check: Verify user is owner or member of project

**Usage**: Applied to routes that require specific permissions

---

## Authentication

### JWT Implementation Strategy

#### Token Generation
- When user registers or logs in, generate JWT token
- Token payload contains: `{ userId: user._id, email: user.email }`
- Token expires after 7 days (configurable)
- Secret key stored in environment variables

#### Token Storage
- Token sent to client in response body after successful authentication
- Client stores token (localStorage, sessionStorage, or httpOnly cookie)
- Token must be included in Authorization header for protected routes: `Authorization: Bearer {token}`

#### Token Verification Flow
1. Client sends request with token in Authorization header
2. Authentication middleware extracts token
3. Verify token signature using secret key
4. Check token expiration
5. Extract userId from token payload
6. Fetch user from database
7. Attach user to `req.user` object
8. Continue to route handler

#### Protected Routes
- All routes except `/api/auth/register` and `/api/auth/login` require authentication
- Authentication middleware checks for valid token
- Returns 401 if token is missing or invalid

#### Token Refresh Strategy
- Current implementation: User must login again after token expiration
- Future enhancement: Implement refresh token mechanism

#### Security Considerations
- Passwords are hashed using bcrypt (10 salt rounds)
- JWT secret stored in environment variables, never in code
- HTTPS should be used in production
- Token expiration prevents indefinite access
- Passwords are never returned in API responses

---

## Error Handling

### Error Response Format

All errors follow a consistent format:
```json
{
  "success": false,
  "error": "Error message here"
}
```

### Error Scenarios and Handling

#### 1. Validation Errors (400 Bad Request)
**Scenarios**:
- Missing required fields
- Invalid data types
- Invalid enum values (status, priority, etc.)
- Invalid email format
- Password too short
- Exceeding character limits
- Invalid date format

**Handling**:
- Use express-validator to validate input
- Return 400 status with descriptive error message
- List all validation errors if multiple fields are invalid

**Example**:
```json
{
  "success": false,
  "error": "Task title is required"
}
```

---

#### 2. Authentication Errors (401 Unauthorized)
**Scenarios**:
- Missing JWT token in request
- Invalid or expired JWT token
- Token signature verification fails
- User account not found (token valid but user deleted)

**Handling**:
- Authentication middleware checks token before route handler
- Return 401 status with "Authentication required" or "Invalid or expired token"
- Client should redirect to login page

**Example**:
```json
{
  "success": false,
  "error": "Authentication required"
}
```

---

#### 3. Authorization Errors (403 Forbidden)
**Scenarios**:
- User tries to access task they didn't create or aren't assigned to
- Non-owner tries to update/delete project
- Non-owner tries to invite/remove project members
- User tries to access notification that doesn't belong to them

**Handling**:
- Check user permissions in route handler or authorization middleware
- Return 403 status with descriptive message
- Log unauthorized access attempts

**Example**:
```json
{
  "success": false,
  "error": "Not authorized to update this task"
}
```

---

#### 4. Not Found Errors (404 Not Found)
**Scenarios**:
- Task ID doesn't exist in database
- Project ID doesn't exist
- User ID doesn't exist
- Notification ID doesn't exist
- Invalid MongoDB ObjectId format

**Handling**:
- Query database for resource
- If not found, return 404 status
- Validate ObjectId format before querying

**Example**:
```json
{
  "success": false,
  "error": "Task not found"
}
```

---

#### 5. Conflict Errors (409 Conflict)
**Scenarios**:
- Email already exists during registration
- User already member of project
- Duplicate invitation sent to same email
- User already accepted/rejected invitation
- Duplicate resource creation

**Handling**:
- Check for existing resources before creation
- Return 409 status for all conflict scenarios

**Example**:
```json
{
  "success": false,
  "error": "Email already exists"
}
```

---

#### 6. Server Errors (500 Internal Server Error)
**Scenarios**:
- Database connection failures
- Unexpected exceptions in code
- External service failures
- Unhandled promise rejections

**Handling**:
- Global error handler catches all unhandled errors
- Log detailed error information for debugging
- Return generic error message to client (don't expose internal details)
- Use try-catch blocks in async operations

**Example**:
```json
{
  "success": false,
  "error": "Server error"
}
```

---



