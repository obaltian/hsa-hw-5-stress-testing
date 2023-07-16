# Simple stress-testing practice of a web server, connected to a database.

Tech stack:

- **flask** web server
- **mysql** database
- **siege** for stress testing

This is a very simple web app, which only implements `/users` endpoint with following methods:

- GET - read **all** users
- POST - insert new user
- PUT - update existing user data
- DELETE - delete existing user

To run project, use:

```bash
docker-compose up --build
```

## Stress-testing summary

### Reading from database

So, the stress-testing was done with **siege** tool, which reads URLs to test from url files. All endpoints are tested except of DELETE, because it would require dynamic selection of user ID to delete.

So, let's start with testing only GET method, and see how it goes:

```bash
siege -c 5 -t 20S -f siege_urls_get.txt

[...]

Lifting the server siege...
Transactions:                    218 hits
Availability:                 100.00 %
Elapsed time:                  20.50 secs
Data transferred:              13.02 MB
Response time:                  0.47 secs
Transaction rate:              10.63 trans/sec
Throughput:                     0.64 MB/sec
Concurrency:                    4.97
Successful transactions:         218
Failed transactions:               0
Longest transaction:            0.69
Shortest transaction:           0.23
```

The application server runs as a single-process script, without any web server workers and paralelising stuff, so I don't expect performance to be very good.

The GET endpoint reads all users, and there are a bunch of users already inserted in my DB before tests, so it takes some time to read all of them.

Let's try to increase the number of concurrent users:

```bash
siege -c 20 -t 20S -f siege_urls_get.txt

[...]

Lifting the server siege...
Transactions:                    202 hits
Availability:                 100.00 %
Elapsed time:                  20.81 secs
Data transferred:              12.06 MB
Response time:                  1.96 secs
Transaction rate:               9.71 trans/sec
Throughput:                     0.58 MB/sec
Concurrency:                   19.03
Successful transactions:         202
Failed transactions:               0
Longest transaction:            2.88
Shortest transaction:           0.98
```

Wow, we increased API users count by 4 times, but the performance slightly decreased! Seems like we found a peak of our application server performance.

## Only writing (insert/update) to database

Now, let's try to test only POST and PUT methods, which insert and update users in database.

```bash
siege -c 5 -t 20S -f siege_urls_post_put.txt --content-type "application/json" --internet

[...]

Lifting the server siege...
Transactions:                   4099 hits
Availability:                 100.00 %
Elapsed time:                  20.55 secs
Data transferred:               0.11 MB
Response time:                  0.02 secs
Transaction rate:             199.46 trans/sec
Throughput:                     0.01 MB/sec
Concurrency:                    4.98
Successful transactions:        4099
Failed transactions:               0
Longest transaction:            0.17
Shortest transaction:           0.00
```

Wow, that's a lot of transactions per second! But, I am not sure if we can compare read and write directly, since write changes just a single user, not a couple of thousands.

Let's try to increase the number of concurrent users:

```bash
siege -c 20 -t 20S -f siege_urls_post_put.txt --content-type "application/json" --internet

[...]

Lifting the server siege...
Transactions:                   4108 hits
Availability:                 100.00 %
Elapsed time:                  20.75 secs
Data transferred:               0.11 MB
Response time:                  0.10 secs
Transaction rate:             197.98 trans/sec
Throughput:                     0.01 MB/sec
Concurrency:                   19.94
Successful transactions:        4108
Failed transactions:               0
Longest transaction:            0.25
Shortest transaction:           0.05
```

Again, performance did not increase. I think, because of the fact, that we have a single-process server, which can only handle one request at a time, and there are no any asynchronous operations we can switch context at (DB calls are sync), siege always gets the peak performance of this app.

## Reading and writing to database

Finally, let's run the combined test:

```bash
siege -c 5 -t 20S -f siege_urls_get_post_put.txt --content-type "application/json" --internet

[...]

Lifting the server siege...
Transactions:                    614 hits
Availability:                 100.00 %
Elapsed time:                  20.66 secs
Data transferred:              12.08 MB
Response time:                  0.16 secs
Transaction rate:              29.72 trans/sec
Throughput:                     0.58 MB/sec
Concurrency:                    4.87
Successful transactions:         614
Failed transactions:               0
Longest transaction:            1.11
Shortest transaction:           0.00
```

We see that performance worsened, because of `GET` all users endpoint.
