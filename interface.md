Interface 
=================

### Sign in

request: GET /api/login

| param | type | description |
| ----- | ---- | ----------- |
| username | string | user name|
| password | string | password encode in md5 |

response: 

| param | type | description |
| ----- | ---- | ----------- |
| retcode | int | retcode, 0 for succ |
| err_str | string | error description |
