Features

Authentication

overview:
A user can register to the system and login in with his credentials so that he can procced to the application and view his feed , profile , etc



Profile

overview:
A logged in user can view his profile to view his account details and be able to update it



Posts

overview:
a logged in user can add a post he can also edit or delete a post he added, also he can view posts of other people.




Comment

overview:
A logged in user can write a comment on a post, he could also edit or delete a comment that he added to a post, he also can view other people comments






Models:

Entity: Users
Purpose: login and store user info, inherite from django abstract user model
Important Fields: email , username, password, date_joined, profile-picture
Relations: a user can add posts, comments, likes
Deletion: cascade deleting post , like or comment , also find the image of the user and and images associated with the user posts and delete them from the storage if they exist

Entity: Posts
Purpose: the feed of what a user can see on the application of his posts or other people posts
Important Fields: user, text or image, comments, likes(who made them)
Relations: a post should have a user that created it, likes and comments
deletion: delete the associated likes and comments, also delete the image associated with the post if it exists in the storage


Entity: Comments
Purpose: what other people thought about the post, MVP comments are only direct children of posts.
Nested comment replies are postponed.
Important Fields: user, text or image
Relations: user , post
deletion: delete only the comment and any image associated with the comment if it exists in the storage

Entity: Likes
Purpose: users can like or remove a like of there own on any post
Important Fields: user, post
Relations: User, Post
deletion: delete only the like


erDiagram
    Users ||--o{ Posts : creates
    Users ||--o{ Comments : creates
    Posts ||--o{ Comments : has
    Posts ||--o{ LIKE : has
    Users ||--o{ LIKE : creates
    Users {
        string email
        string username
        string password
        url profile-picture
        datetime date_joined
    }
    Posts {
        forginkey Users
        string text
        url image
    }
    Comments {
        string text
        url image
        forginkey Users
        foginkey post
    }
    LIKE {
         forginkey Post
         forginkey User
         created_at
    }


Database design

Users
-------
id
email unique
username unique
password (hashed) 
profile_picture
date_of_birth
phone_number
date_joined
...

Posts
-------
id
auther_id
created_at
text
image
...

Comments
----------
id
auther_id
created_at
text
image
post
...

Like

----------
id
user_id
post_id
created_at

UNIQUE(user_id, post_id)
...

API Design

Conventions
-----------
- Request and response bodies use JSON unless an endpoint accepts an image upload.
- Image uploads use multipart/form-data with an optional `image` file field. Responses return the stored image URL in the `image` field.
- IDs in paths and response bodies are database IDs. Timestamps use ISO 8601 UTC strings.
- Authenticated requests use the Django `sessionid` cookie. The cookie is Secure, HttpOnly, and SameSite=Lax in production.
- Requests that change state (POST, PATCH, DELETE) include `X-CSRFToken` with the CSRF token issued by Django. The Browser forwards the session and CSRF cookies/headers to Django.
- Successful JSON responses use `Content-Type: application/json`. A `204 No Content` response has no body.
- Error responses use `{ "error": { "code": "...", "message": "...", "details": {} } }`. `details` may be omitted when there are no field-specific errors.

Authentication
---------------

`POST /api/auth/register` (public)
- Headers: `Content-Type: application/json`, `Accept: application/json`
- Request body: `{ "fullname": "Jane Doe", "email": "jane@example.com", "username": "jane", "password": "...", "date_of_birth": "1998-04-12", "phone_number": "+15551234567" }`. `date_of_birth` and `phone_number` are optional.
- Response: `201 Created`; body `{ "id": 42, "fullname": "Jane Doe", "email": "jane@example.com", "username": "jane", "profile_picture": null, "date_of_birth": "1998-04-12", "phone_number": "+15551234567", "created_at": "2026-10-05T12:00:00Z" }`.
- The password is accepted in the registration request. Django hashes it before storage; plaintext passwords are never stored or returned in API responses. Duplicate email/username returns `409 Conflict`.


`POST /api/auth/login` (public)
- Headers: `Content-Type: application/json`, `Accept: application/json`
- Request body: `{ "username_or_email": "jane", "password": "..." }`.
- Response: `200 OK`; headers include `Set-Cookie: sessionid=...; HttpOnly; Secure; SameSite=Lax` and Django's CSRF cookie. Body `{ "user": { "id": 42, "fullname": "Jane Doe", "email": "jane@example.com", "username": "jane", "profile_picture": null } }`.
- Invalid credentials return `401 Unauthorized` with the standard error body.

`POST /api/auth/logout` (authenticated)
- Headers: `Accept: application/json`, `Cookie: sessionid=...`, `X-CSRFToken: ...`
- Request body: none.
- Response: `204 No Content`; expires the session cookie.

Feed and posts
--------------

`GET /api/feed` (authenticated; includes posts by all users, ordered by `created_at DESC, id DESC`)
- Headers: `Accept: application/json`, `Cookie: sessionid=...`
- Query: optional `limit` (default 20, maximum 100) and `offset` (default 0).
- Request body: none.
- Response: `200 OK`; body `{ "results": [<post>], "limit": 20, "offset": 0, "total": 125 }`. A post is `{ "id": 7, "author": { "id": 42, "username": "jane", "fullname": "Jane Doe", "profile_picture": null }, "created_at": "2026-10-05T12:00:00Z", "text": "Hello", "image": null, "like_count": 3, "liked_by_me": false, "comment_count": 2 }`.

`POST /api/posts` (authenticated)
- Headers: `Accept: application/json`, `Cookie: sessionid=...`, `X-CSRFToken: ...`; `Content-Type: application/json` for text-only posts, or `multipart/form-data` when uploading an image.
- Request body: `{ "text": "Hello", "image": null }` as JSON, or multipart fields `text` and optional `image` file. At least one of non-empty `text` or `image` is required.
- Response: `201 Created`; body is a post representation as above, with `like_count: 0`, `liked_by_me: false`, and `comment_count: 0`.

`GET /api/posts/{id}` (authenticated)   
- Headers: `Accept: application/json`, `Cookie: sessionid=...`
- Request body: none.
- Response: `200 OK`; body is a post representation. Unknown IDs return `404 Not Found`.

`PATCH /api/posts/{id}` (authenticated; author only)
- Headers: `Accept: application/json`, `Cookie: sessionid=...`, `X-CSRFToken: ...`; `Content-Type: application/json` or `multipart/form-data` when uploading/replacing an image.
- Request body: any subset of `{ "text": "Updated text", "image": null }`; multipart may provide `text` and/or a replacement `image` file. Omit `image` to leave it unchanged; send `null`/empty value to remove it. At least one field is required.
- Response: `200 OK`; body is the updated post representation. Non-authors receive `403 Forbidden`.

`DELETE /api/posts/{id}` (authenticated; author only)
- Headers: `Accept: application/json`, `Cookie: sessionid=...`, `X-CSRFToken: ...`
- Request body: none.
- Response: `204 No Content`. Associated comments and likes are deleted by cascade. Non-authors receive `403 Forbidden`.

Likes and comments
------------------

`POST /api/posts/{id}/like` (authenticated)
- Headers: `Accept: application/json`, `Cookie: sessionid=...`, `X-CSRFToken: ...`
- Request body: none.
- Response: `201 Created`; body `{ "post_id": 7, "user_id": 42, "created_at": "2026-10-05T12:00:00Z" }`. The `(user_id, post_id)` unique constraint ensures one like per user and post; if already liked, return `200 OK` with the existing like representation.

`DELETE /api/posts/{id}/like` (authenticated; removes the current user's like)
- Headers: `Accept: application/json`, `Cookie: sessionid=...`, `X-CSRFToken: ...`
- Request body: none.
- Response: `204 No Content`. If the current user has not liked the post, return `404 Not Found`.

`POST /api/posts/{id}/comments` (authenticated)
- Headers: `Accept: application/json`, `Cookie: sessionid=...`, `X-CSRFToken: ...`; `Content-Type: application/json` or `multipart/form-data` when uploading an image.
- Request body: `{ "text": "Nice post", "image": null }` as JSON, or multipart fields `text` and optional `image` file. At least one of non-empty `text` or `image` is required.
- Response: `201 Created`; body `{ "id": 15, "post_id": 7, "author": { "id": 42, "username": "jane", "fullname": "Jane Doe", "profile_picture": null }, "created_at": "2026-10-05T12:05:00Z", "text": "Nice post", "image": null }`.


`GET /api/posts/{id}/comments` (authenticated, returns all comments for a post ordered by created_at ascending)
- Headers: `Accept: application/json`, `Cookie: sessionid=...`
- Query: optional `limit` (default 20, maximum 100) and `offset` (default 0).
- Request body: none.
- Response: `200 OK`; body `{ "results": [<comment>], "limit": 20, "offset": 0, "total": 5 }`. A comment is `{ "id": 15, "post_id": 7, "author": { "id": 42, "username": "jane", "fullname": "Jane Doe", "profile_picture": null }, "created_at": "2026-10-05T12:05:00Z", "text": "Nice post", "image": null }`.

`GET /api/comments/{id}` (authenticated)
- Headers: `Accept: application/json`, `Cookie: sessionid=...`
- Request body: none.
- Response: `200 OK`; body is a comment representation as above. Unknown IDs return `404 Not Found`.

`PATCH /api/comments/{id}` (authenticated; author only)
- Headers: `Accept: application/json`, `Cookie: sessionid=...`, `X-CSRFToken: ...`; `Content-Type: application/json` or `multipart/form-data` when uploading/replacing an image.
- Request body: any subset of `{ "text": "Updated comment", "image": null }`; image semantics match post PATCH. At least one field is required.
- Response: `200 OK`; body is the updated comment representation. Non-authors receive `403 Forbidden`.

`DELETE /api/comments/{id}` (authenticated; author only)
- Headers: `Accept: application/json`, `Cookie: sessionid=...`, `X-CSRFToken: ...`
- Request body: none.
- Response: `204 No Content`. delete only the comment. Non-authors receive `403 Forbidden`.

Profiles and accounts
---------------------

`GET /api/profile/{id}` (authenticated)
- Headers: `Accept: application/json`, `Cookie: sessionid=...`
- Request body: none.
- Response: `200 OK`; body `{ "id": 42, "fullname": "Jane Doe", "username": "jane", "profile_picture": null, "created_at": "2026-10-05T12:00:00Z" }`. Private account details such as email and phone number are returned only when the requester is the profile owner. Unknown IDs return `404 Not Found`.

`PATCH /api/profile/{id}` (authenticated; profile owner only)
- Headers: `Accept: application/json`, `Cookie: sessionid=...`, `X-CSRFToken: ...`; `Content-Type: application/json` or `multipart/form-data` when uploading a profile picture.
- Request body: any subset of `{ "fullname": "Jane Smith", "username": "janesmith", "email": "jane@example.com", "date_of_birth": "1998-04-12", "phone_number": "+15551234567", "profile_picture": null }`; multipart accepts the scalar fields and an optional `profile_picture` file. Password cannot be changed here.
- Response: `200 OK`; body is the updated profile representation, including private fields for its owner. Duplicate email/username returns `409 Conflict`; non-owners receive `403 Forbidden`.

`DELETE /api/user/{id}` (authenticated; account owner or admin)
- Headers: `Accept: application/json`, `Cookie: sessionid=...`, `X-CSRFToken: ...`
- Request body: none.
- Response: `204 No Content`; related posts, comments, and likes are deleted by cascade. The deleted user's session is invalidated. Other users receive `403 Forbidden`.

Common errors
-------------
- `400 Bad Request`: malformed request or field validation failure.
- `401 Unauthorized`: missing or invalid session, or invalid login credentials.
- `403 Forbidden`: authenticated user lacks permission, or CSRF validation failed.
- `404 Not Found`: requested resource does not exist (or is not visible to the requester).
- `409 Conflict`: unique email/username constraint violation.
- `415 Unsupported Media Type`: unsupported request content type.
- `500 Internal Server Error`: unexpected server error; response does not expose stack traces or secrets.


Authentication architecture

register ----> login (username or email, password) 
use Django session + secure HTTP-only cookie
browser sends the request with session cookie
django confirmes if the session is valid and belongs to which user
logging out expires the session


LOGIN
Browser → Django → create session → sends session back with cookies → browser stores session


Authenticated API request
Browser → Django (verify cookie)



Frontend architecture

assets
app/
    apis/
    components/
    features/
    hooks/
    lib/
    services/
    types/
    pages/



Initial infrastructure
    
flowchart TD
    B[Browser]
    N[Next.js - UI and pages]
    D[Django REST Framework]
    P[(PostgreSQL)]

    B -->|Page requests| N
    B -->|API requests + session cookie| D
    D --> P


Phase 2 features

relationships
messaging
notifications
freinds