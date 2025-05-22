# Real-Time-Chat-Application

1. **High level architecture for the Real time Chat Application**
                    ┌──────────────┐
                    │   Frontend   │
                    │ (HTML/JS)    │
                    └─────┬────────┘
                          │
              ┌───────────▼────────────┐
              │ WebSocket Connection   │
              │ ws://domain/ws/chat/   │
              └───────────┬────────────┘
                          │
                   ┌──────▼─────┐
                   │  Daphne    │  ← ASGI Server
                   └──────┬─────┘
                          │
        ┌─────────────────▼────────────────┐
        │ Django Channels (Consumers)      │
        │ - AuthMiddlewareStack            │
        │ - ChatConsumer                   │
        │ - Receives/Saves/Broadcasts Msg  │
        └────────────┬─────────────────────┘
                     │
        ┌────────────▼────────────┐
        │    Redis (Channel Layer)│
        │  - Pub/Sub Message Bus  │
        └────────────┬────────────┘
                     │
     ┌───────────────▼────────────────┐
     │ Other WebSocket Connections    │
     │ (Same Room / Group)            │
     └───────────────────────────────┘

          ┌──────────────────────────┐
          │     PostgreSQL / DB      │
          │  - Message table         │
          └──────────────────────────┘


2.Components:

| Component                         | Description                                       |
| --------------------------------- | ------------------------------------------------- |
| **Frontend (JS + HTML)**          | User interface for login, chat UI, message search |
| **Django REST APIs**              | Login/logout and message search functionality     |
| **WebSocket (Daphne + Channels)** | Real-time messaging over WebSockets               |
| **Redis**                         | Message broadcasting (channel layer) and cache    |
| **PostgreSQL (or SQLite)**        | Stores messages and users                         |
| **Daphne (ASGI server)**          | Handles HTTP & WebSocket connections              |

3. APIs involved:
   
**1. Login API**
Purpose:
Authenticate a user and create a session or return a token.

Request Type:
POST

Request Parameters:
username (string): The user's username
password (string): The user's password

Response:
Success: Login confirmation with user details
Failure: Error message indicating invalid credentials

Flow:
1.Frontend sends login credentials to the server.
2.Server verifies credentials.
3.On success, creates a session/token and returns a success response.
4.On failure, returns an error response.

**2. Logout API**
Purpose:
Log out the currently authenticated user by clearing the session or invalidating the token.

Request Type:
POST

Request Parameters:
None

Response:
Success: Message confirming successful logout

Flow:
Frontend triggers a logout request.
Server removes session data or invalidates token.
Returns confirmation response

**3. SignUp API**
Purpose:
Register a new user by creating a new account.

Request Type:
POST

Request Parameters:

email(string): email id
username (string): Desired username
password (string): Desired password
confirm password (string): Desired password

Response:
Success: Message confirming account creation
Failure: Error message if the username already exists

Flow:
Frontend sends desired username and password.
Server checks if the username is unique.
If valid, creates and stores the user account.
Returns success or error response accordingly.

 **4. Chat Room Message Search API**
 Purpose:Retrieve chat messages exchanged between the logged-in user and another user (identified by room_name), with optional search functionality for filtering messages by keyword.
Request Type:
GET

URL Pattern:
/chat/<room_name>/?search=<keyword>

Path Parameters:
room_name (string): Username of the user you want to chat with.
Query Parameters:
search (string, optional): Keyword or phrase to filter messages within the chat.

Authentication:
User must be logged in (session or token-based authentication).

Response:
HTML page (chat.html) rendered with:
List of chat messages between the two users, optionally filtered by search keyword.
List of all other users except the logged-in user.
List of users with their last exchanged message, sorted by recency.
Current room_name and the search keyword to populate the UI.

Flow:
The client sends a GET request to /chat/<room_name>/ optionally including search in the query string.
The server verifies the user is authenticated.
It retrieves messages exchanged between the current user and room_name.
If search is provided, it filters messages containing the search keyword.
Messages are ordered chronologically.
It fetches all users except the current user and finds their latest message with the logged-in user.
Passes all data to the template for rendering the chat room UI.

**Database Schema :**

+-------------+                +------------------------------+
|   User      |                |          Message             |
+-------------+                +------------------------------+
| id (PK)     |<------------+  | id (PK)                      |
| username    |             |  | sender_id (FK → User.id)    |
| password    |             +--| receiver_id (FK → User.id)  |
+-------------+                | text                         |
                               | timestamp                    |
                               +------------------------------+
**Relationships**
User → Message (as sender)
One user can send many messages
sender_id in Message → id in User
User → Message (as receiver)
One user can receive many messages
receiver_id in Message → id in User


**Security:**

**1. User Authentication**
Use Django's built-in User model for secure password storage (hashed using PBKDF2).
Protect views using @login_required or Django REST Framework permissions like IsAuthenticated.

**2. WebSocket Authentication**
Use AuthMiddlewareStack in Django Channels to attach the logged-in user to the WebSocket scope.
Validate the user identity in each WebSocket message handler to prevent spoofing.

**3. Authorization (Access Control)**
Ensure users can only send/receive messages that involve them as sender or receiver.
Block unauthorized access to rooms or messages using checks in views and consumers.

**4. Input Validation & Sanitization**
Validate all incoming user inputs (like message text) on both client and server sides.
Use Django’s auto-escaping or libraries like bleach to sanitize user-generated content and prevent XSS.

**5. Rate Limiting**
Apply rate limits on API endpoints (e.g., login, message send) using Django REST Framework or Redis.
Throttle excessive WebSocket messages to prevent spamming or denial-of-service.

**6. CSRF Protection**
Enable CSRF protection for all POST/PUT requests using Django’s middleware.
Use CSRF tokens in HTML forms and AJAX requests for authenticated users.

**7. Secure Transmission**
Use HTTPS in production to encrypt all HTTP and WebSocket traffic.
Redirect all requests to HTTPS using SECURE_SSL_REDIRECT = True in settings.

**8. Secure Redis and Channels Setup**
Restrict Redis access to localhost or use authentication with a strong Redis password.
Avoid exposing your ASGI and Channels servers to public networks without firewalls.

**9. Database Security**
Rely on Django ORM to prevent SQL injection (no raw queries unless necessary).
Use least privilege principle: your database user should only have permissions it needs.

**10. Monitoring and Logging**
Log critical events like login failures, suspicious WebSocket traffic, and message spikes.
Integrate monitoring tools like Sentry or set up alerts for error patterns or abuse detection.



**Cost Optimization Techniques**

1. Use Scalable Hosting (e.g., VPS or PaaS)
Deploy on cloud services like DigitalOcean, Linode, or AWS Lightsail instead of large, expensive platforms.
Choose auto-scalable instances to adjust resources during traffic spikes.

2. Efficient WebSocket Handling
Use Daphne with Uvicorn or ASGI workers efficiently behind a load balancer (e.g., Nginx).
Reuse connections and limit idle timeouts to avoid resource wastage.

3. Optimize Redis Usage
Use Redis only for channels layer and pub/sub; avoid unnecessary data caching.
Use a managed Redis instance with limited memory and monitoring to avoid overprovisioning.

4. Database Optimization
Optimize queries with indexes on sender, receiver, and timestamp fields.
Use read replicas or caching for high-read scenarios instead of scaling DB vertically.

5. Use Caching Wisely
Cache user lists, last messages, and static chat history using Redis or in-memory caching.
Avoid caching rapidly changing data like live chat messages.

6. Optimize Frontend Load
Use pagination or lazy loading for messages instead of loading full chat history.
Compress and minify static assets (JS, CSS) and use CDN for delivery.

7. Use Docker & CI/CD
Deploy using Docker containers to isolate services and optimize resource usage.
Automate builds and use CI/CD pipelines (like GitHub Actions) to reduce deployment costs and errors..

8. Selective Logging
Avoid verbose logging in production (e.g., logging every message).
Log only critical events to save storage and reduce logging service costs.

9. Optimize Message Storage
Archive or delete old messages not accessed in a long time.
Use cold storage for historical chat logs if long-term retention is needed.


