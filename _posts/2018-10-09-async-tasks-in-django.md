---
title: "Management Commands: Async Tasks without Celery in Django"
layout: post
date: 2018-10-09 10:00
tag:
- django
category: blog
author: taranjeet
---

Management commands in Django are custom python scripts, with the added benefit of having access to django environment. Django Docs about [management command](https://docs.djangoproject.com/en/2.1/howto/custom-management-commands/) tells in detail about how to write a management commands.

Management commands can be used to do any task in the background or run cron jobs. For example, we can have a management command which fetches and downloads content about a particular book and later on emails it to the user.

In the following section, we will explain how management command can be used to form asynchronous tasks. It will later on draw comparsion when these tasks are performed as celery tasks.

Lets start with an example library app, where we have Book and Author model. There is a view which downloads the list of all the books stored in the database as a csv file.

-- include code
    - book model
    - book view
    - pic of how the download view looks like

Lets add support to download csv of Books. We can start with using HttpResponse, and block the whole request-response cycle till the csv formation process is not complete. Ideally the flow of steps would look like

* User requests to download the csv
* Django view receives the request, process the parameters and forms a condition clause for the queryset.
* Next, the queryset is requested and now iterated on to write in a csv file.
* Once the whole iteration is complete on the queryset, and csv file writing is complete, response is sent with appropriate content type.
* The browser starts downloading the csv file.

The code to do so would look something like

-- include code
    - view
    - pic of how the csv is downloaded in the browser

This method works fine, but fails or times out, if the number of records in the database is huge. One solution for this can be to increase the timeout value at webserver level, but that approach doesn't work in all the situation. It will be linearly dependent on the number of records in the database and will have to manually increase if the record increase. This brings an additional headache.

Another way to solve this problem can be [StreamingHttpResponse](https://docs.djangoproject.com/en/2.1/ref/request-response/#django.http.StreamingHttpResponse). It is used to stream response from django to the browser. Django has a detailed documentation along with code of how StreamingHttpResponse can be used in [Streaming large Csv Files](https://docs.djangoproject.com/en/2.1/howto/outputting-csv/#streaming-csv-files). In our case, the code will look like

-- include code
    - views
    - pic of how the csv download generation is continuing in the browser.

This solution works fine, and is a good optimization over response sent via HttpResponse. However, the same problem can be solved using management command.

For this first we will write a management command which fetches the record from the database and creates a csv file.

-- include code
    - about management command

We can run the management command via shell and see that the csv file is being generated.

-- include code about running management command
    - show csv file generated as well.

Now we can plug this management command in the view. This means that whenever any request is received by the view to create and download a csv, the view uses subprocess module to spawn a process for management command that we wrote above. This way the view starts its processing in the background and the view returns response immediately to the user. This means that the view notifies the user that his request has been received and will be handed over the result soon. Note that now the response is sent, it means to deliver the generate csv file as response to the user, alternate means like sending the file via email has to be used. If the file size is to much, the file can be compressed to meet the maximum file attachement size for various email hosts.

The below code shows how the view can be modified to add support for spawning a new process for management command. Notice the use of virtual environment as well.

This way we can use a management command to do asynchronous tasks in django. This method has its own pros and cons when compared to doing asynchronous tasks with celery. Some of the points are

Pro

* No need of using any queing solution. For celery you need a task broker like RabbitMQ or Redis to maintain a queue and handover tasks to the worker. However, this is not the case with management commands.

Cons

* There may be some overhead in terms of time, since a new process is spawned to perform tasks.



