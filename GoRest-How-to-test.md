# How to test the GoRest API

Base URL: `https://gorest.co.in/public/v2`

Import file: `GoRest.postman_collection.json`

The default budget is **90 requests per minute for one access token**. This collection makes **27 calls**, and only the write steps send your token. Run it **once**. Do not set iterations above 1. Do not press Run again in the same minute if a step already reported that fewer than 10 calls remain.

## Before you run

1. Open https://gorest.co.in and sign in.
2. Open https://gorest.co.in/my-account/access-tokens and create a token. Leave its limit at 90.
3. In Postman, click Import and choose `GoRest.postman_collection.json`.
4. Click the collection **GoRest API**. Open the Variables tab. Paste the token into `token`. Do not put the word Bearer in that box.
5. Click the collection, then Run. Leave every folder ticked. Iterations = 1. Delay = 0.
6. A later request uses the user id, post id, comment id, and todo id saved by earlier requests. Do not press Send on GR-15 by itself.

Green means that step passed. Open a red step to see which check failed.

`X-RateLimit-Remaining` is how many calls that token can still make this minute. The collection fails a step when that number drops below 10, so you stop before the API returns 429.

## Test data

| Field | Value |
| --- | --- |
| New user name | Learner Case |
| New user email | learner.&lt;time&gt;@example.com. The collection fills in the time so the email is new on every run. |
| Gender | female |
| Status at create | active |
| Status after GR-13 | inactive |
| Post title | API practice note |
| Post body | Created by the GoRest collection. |
| Comment name | Priya Sharma |
| Comment email | priya.sharma@email.com |
| Comment body | Clear and useful. |
| Todo title | Finish the API notes |
| Todo due date | 2027-12-01T00:00:00.000Z |
| Todo status at create | pending |
| Todo status after GR-20 | completed |
| Unknown user id | 999999999 |

## 1 Read and pagination

These six calls do not send the token.

| Case | Input | Expected |
| --- | --- | --- |
| GR-01 | `GET /users?page=1&per_page=2` | 200. Two users. `X-Pagination-Page` is 1. `X-Pagination-Limit` is 2. Total is greater than 2. |
| GR-02 | `GET /users?page=2&per_page=2` | 200. Two users. Page header is 2. The first id is not the first id from page 1. |
| GR-03 | `GET /users?gender=female&status=active&per_page=5` | 200. Every row has gender female and status active. |
| GR-04 | `GET /users/999999999` | 404. Message is `Resource not found`. |
| GR-05 | `GET /users?per_page=101` | 200. Limit header stays 10. The API ignores 101. The documented maximum page size is 100. |
| GR-06 | `GET /users.xml?per_page=1` with Accept `application/xml` | 200. Content type contains xml. The body contains `<objects`. |

## 2 Users

| Case | Input | Expected |
| --- | --- | --- |
| GR-07 | POST `/users` with name No Token, email notoken@example.com, gender male, status active. No token. | 401. Message is `Authentication failed`. |
| GR-08 | POST `/users` with the token and the new-user row above. | 201. Name, email, gender, and status match the input. The id is saved. |
| GR-09 | GET `/users/{{userId}}` with the token. | 200. The email matches GR-08. |
| GR-10 | POST `/users` with name Missing Email, gender male, status active. No email. | 422. The error list includes field `email`. |
| GR-11 | POST `/users` with gender `other` and a new email. | 422. The error list includes field `gender`. Allowed values are male and female. |
| GR-12 | POST `/users` again with the email from GR-08. | 422. The error list includes field `email`. |
| GR-13 | PATCH `/users/{{userId}}` with `{ "status": "inactive" }`. | 200. Status is inactive. |

## 3 Posts, comments, and todos

| Case | Input | Expected |
| --- | --- | --- |
| GR-14 | GET `/users/{{userId}}/posts` | 200. The list is empty. |
| GR-15 | POST `/users/{{userId}}/posts` with the post title and body above. | 201. `user_id` is the saved user. The post id is saved. |
| GR-16 | GET `/posts/{{postId}}` | 200. Title is API practice note. |
| GR-17 | POST `/posts/{{postId}}/comments` with the comment row above. | 201. `post_id` matches. The comment id is saved. |
| GR-18 | GET `/posts/{{postId}}/comments` | 200. One comment, and its id is the saved comment id. |
| GR-19 | POST `/users/{{userId}}/todos` with the todo row above. | 201. Status is pending. The todo id is saved. |
| GR-20 | PATCH `/todos/{{todoId}}` with `{ "status": "completed" }`. | 200. Status is completed. |

## 4 Cleanup

| Case | Input | Expected |
| --- | --- | --- |
| GR-21 | DELETE `/comments/{{commentId}}` | 204. The body is empty. |
| GR-22 | DELETE `/posts/{{postId}}` | 204. The body is empty. |
| GR-23 | DELETE `/todos/{{todoId}}` | 204. The body is empty. |
| GR-24 | DELETE `/users/{{userId}}` | 204. The body is empty. |
| GR-25 | GET `/users/{{userId}}` | 404. Message is `Resource not found`. |

## 5 Limit and delay

| Case | Input | Expected |
| --- | --- | --- |
| GR-26 | `GET /users?force_status=429&per_page=1` | 429. `simulated` is true. Header `X-Simulated-Status` is 429. This simulated response does not spend the real 90-call budget. |
| GR-27 | `GET /users?delay=200&per_page=1` | 200. Header `X-Simulated-Delay-Ms` is 200. The call waits about 200 milliseconds. Do not use a delay of 5000 here. |

## If you see 429 for real

The body is `{ "message": "Too many requests" }` and `X-RateLimit-Remaining` is 0. Wait for the number of seconds in `X-RateLimit-Reset`, then run the collection once more. Do not click Run again immediately.
