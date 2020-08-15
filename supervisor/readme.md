# Asynchronous Tasks With Django and Celery
## (remote daemonizing with supervisor)
Table of Contents

-   [Overview](https://realpython.com/asynchronous-tasks-with-django-and-celery/#overview)
-   [What is Celery?](https://realpython.com/asynchronous-tasks-with-django-and-celery/#what-is-celery)
-   [Setup](https://realpython.com/asynchronous-tasks-with-django-and-celery/#setup)
    -   [Step 1: Add celery.py](https://realpython.com/asynchronous-tasks-with-django-and-celery/#step-1-add-celerypy)
    -   [Step 2: Import your new Celery app](https://realpython.com/asynchronous-tasks-with-django-and-celery/#step-2-import-your-new-celery-app)
    -   [Step 3: Install Redis as a Celery “Broker”](https://realpython.com/asynchronous-tasks-with-django-and-celery/#step-3-install-redis-as-a-celery-broker)
-   [Celery Tasks](https://realpython.com/asynchronous-tasks-with-django-and-celery/#celery-tasks)
    -   [Add the Task](https://realpython.com/asynchronous-tasks-with-django-and-celery/#add-the-task)
-   [Periodic Tasks](https://realpython.com/asynchronous-tasks-with-django-and-celery/#periodic-tasks)
    -   [Add the Task](https://realpython.com/asynchronous-tasks-with-django-and-celery/#add-the-task_1)
-   [Running Locally](https://realpython.com/asynchronous-tasks-with-django-and-celery/#running-locally)
-   [Running Remotely](https://realpython.com/asynchronous-tasks-with-django-and-celery/#running-remotely)

## What is Celery?
“[Celery](http://www.celeryproject.org/)  is an asynchronous task queue/job queue based on distributed message passing. It is focused on real-time operation, but supports scheduling as well.” For this post, we will focus on the scheduling feature to periodically run a job/task.

**Why is this useful?**
-   Think of all the times you have had to run a certain task in the future. Perhaps  [you needed to access an API](https://realpython.com/api-integration-in-python/)  every hour. Or maybe you needed to  [send a batch of emails](https://realpython.com/python-send-email/)  at the end of the day. Large or small, Celery makes scheduling such periodic tasks easy.
-   You never want end users to have to wait unnecessarily for pages to load or actions to complete. If a long process is part of your application’s workflow, you can use Celery to execute that process in the background, as resources become available, so that your application can continue to respond to client requests. This keeps the task out of the application’s context.

## Setup

Before diving into Celery, grab the starter project from the Github  [repo](https://github.com/realpython/demo/releases/tag/v1). Make sure to activate a virtualenv, install the requirements, and  [run the migrations](https://realpython.com/django-migrations-a-primer/). Then fire up the server and navigate to  [http://localhost:8000/](http://localhost:8000/)  in your browser. You should see the familiar “Congratulations on your first Django-powered page” text. When done, kill the server.

Next, let’s install Celery  [using pip](https://realpython.com/what-is-pip/):
```
$ pip install celery==3.1.18
$ pip freeze > requirements.txt
```
Now we can integrate Celery into our Django Project in just three easy steps.
### Step 1: Add celery.py
Inside the “demo” directory, create a new file called  _celery.py_:
```
from __future__ import absolute_import
import os
from celery import Celery
from django.conf import settings

# set the default Django settings module for the 'celery' program.
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'demo.settings')
app = Celery('demo')

# Using a string here means the worker will not have to
# pickle the object when using Windows.
app.config_from_object('django.conf:settings')
app.autodiscover_tasks(lambda: settings.INSTALLED_APPS)


@app.task(bind=True)
def debug_task(self):
    print('Request: {0!r}'.format(self.request))
```
Take note of the comments in the code.
### Step 2: Import your new Celery app

To ensure that the Celery app is loaded when Django starts, add the following code into the  ___init__.py_  file that sits next to your  _settings.py_  file:
```
from __future__ import absolute_import

# This will make sure the app is always imported when
# Django starts so that shared_task will use this app.
from .celery import app as celery_app
```
Having done that, your  [project layout](https://realpython.com/python-application-layouts/)  should now look like:
```
demo
├── demo
│   ├── asgi.py
│   ├── celery.py
│   ├── __init__.py
│   ├── settings.py
│   ├── tasks.py
│   ├── urls.py
│   └── wsgi.py
├── feedback
│   ├── admin.py
│   ├── emails.py
│   ├── forms.py
│   ├── __init__.py
│   ├── models.py
│   ├── tasks.py
│   ├── tests.py
│   └── views.py
├── log
│   ├── demo_beat.log
│   └── demo_worker.log
├── manage.py
├── photos
│   ├── admin.py
│   ├── __init__.py
│   ├── models.py
│   ├── settings.py
│   ├── tasks.py
│   ├── tests.py
│   ├── utils.py
│   └── views.py
├── requirements.txt
├── supervisor
│   ├── demo_celerybeat.conf
│   ├── demo_celery.conf
│   └── readme.md
└── templates
    ├── base.html
    ├── feedback
    │   ├── contact.html
    │   └── email
    │       ├── feedback_email_body.txt
    │       └── feedback_email_subject.txt
    └── photos
        └── photo_list.html
```
### Step 3: Install Redis as a Celery “Broker”

Celery uses “[brokers](http://celery.readthedocs.org/en/latest/getting-started/brokers/)” to pass messages between a Django Project and the Celery  [workers](http://celery.readthedocs.org/en/latest/userguide/workers.html). In this tutorial, we will use  [Redis](https://realpython.com/caching-in-django-with-redis/#what-is-redis)  as the message broker.

First, install  [Redis](http://redis.io/)  from the official download  [page](http://redis.io/download)  or via brew (`brew install redis`) and then turn to your terminal, in a new terminal window, fire up the server:
```
$ redis-server
```
You can test that Redis is working properly by typing this into your terminal:
```
$ redis-cli ping
```
Redis should reply with  `PONG`  - try it!
Once Redis is up, add the following code to your settings.py file:

```# CELERY STUFF
BROKER_URL = 'redis://localhost:6379'
CELERY_RESULT_BACKEND = 'redis://localhost:6379'
CELERY_ACCEPT_CONTENT = ['application/json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TIMEZONE = 'Africa/Nairobi'
```
You also need to add Redis as a dependency in the Django Project:
```
$ pip install redis==2.10.3
$ pip freeze > requirements.txt
```
That’s it! You should now be able to use Celery with Django. For more information on setting up Celery with Django, please check out the official Celery  [documentation](http://docs.celeryproject.org/en/latest/django/first-steps-with-django.html#using-celery-with-django).

Before moving on, let’s run a few sanity checks to ensure all is well…
Test that the Celery worker is ready to receive tasks:
```
$ celery -A demo worker -l info
...
[2015-07-07 14:07:07,398: INFO/MainProcess] Connected to redis://localhost:6379//
[2015-07-07 14:07:07,410: INFO/MainProcess] mingle: searching for neighbors
[2015-07-07 14:07:08,419: INFO/MainProcess] mingle: all alone
```
Kill the process with CTRL-C. Now, test that the Celery task scheduler is ready for action:
```
$ celery -A demo beat -l info
...
[2015-07-07 14:08:23,054: INFO/MainProcess] beat: Starting...
```
Boom!
Again, kill the process when done.
### Add the Task

Basically, after the user submits the feedback form, we want to immediately let him continue on his merry way while we process the feedback, send an email, etc., all in the background.

To accomplish this, first add a file called  _tasks.py_  to the “feedback” directory:
```
from celery.decorators import task
from celery.utils.log import get_task_logger
from feedback.emails import send_feedback_email
logger = get_task_logger(__name__)

@task(name="send_feedback_email_task")
def send_feedback_email_task(email, message):
    """sends an email when feedback form is filled successfully"""
    logger.info("Sent feedback email")
    return send_feedback_email(email, message)
```
Then update  _forms.py_  like so:
```
from django import forms
from feedback.tasks import send_feedback_email_task

class FeedbackForm(forms.Form):
    email = forms.EmailField(label="Email Address")
    message = forms.CharField(label="Message", widget=forms.Textarea(attrs={'rows': 5}))
    honeypot = forms.CharField(widget=forms.HiddenInput(), required=False)

    def send_email(self):
        # try to trick spammers by checking whether the honeypot field is
        # filled in; not super complicated/effective but it works
        if self.cleaned_data['honeypot']:
            return False
        send_feedback_email_task.delay(self.cleaned_data['email'], self.cleaned_data['message'])
```
In essence, the  `send_feedback_email_task.delay(email, message)`  function processes and sends the feedback email in the background as the user continues to use the site.
## Periodic Tasks
Often, you’ll need to schedule a task to run at a specific time every so often - i.e.,  [a web scraper](https://realpython.com/python-web-scraping-practical-introduction/)  may need to run daily, for example. Such tasks, called  [periodic tasks](http://celery.readthedocs.org/en/latest/userguide/periodic-tasks.html), are easy to set up with Celery.

Celery uses “celery beat” to schedule periodic tasks. Celery beat runs tasks at regular intervals, which are then executed by celery workers.

For example, the following task is scheduled to run every fifteen minutes:
### Add the Task
Add a  _tasks.py_  to the  `photos`  app:
```
from celery.task.schedules import crontab
from celery.decorators import periodic_task
from celery.utils.log import get_task_logger
from photos.utils import save_latest_flickr_image
logger = get_task_logger(__name__)

@periodic_task(run_every=(crontab(minute='*/15')), name="task_save_latest_flickr_image", ignore_result=True)
def task_save_latest_flickr_image():
    """ Saves latest image from Flickr """
    save_latest_flickr_image()
    logger.info("Saved image from Flickr")
```
Here, we run the `save_latest_flickr_image()` function every fifteen minutes by wrapping the function call in a `task`. The `@periodic_task` decorator abstracts out the code to run the Celery task, leaving the _tasks.py_ file clean and easy to read!
## Running Locally

Ready to run this thing?

With your Django App and Redis running, open two new terminal windows/tabs. In each new window, navigate to your project directory, activate your virtualenv, and then run the following commands (one in each window):
```
$ celery -A demo worker -l info
$ celery -A demo beat -l info
``` 
When you visit the site on  [http://127.0.0.1:8000/](http://127.0.0.1:8000/)  you should now see one image. Our app gets one image from Flickr every 15 minutes:

Take a look at `photos/tasks.py` to see the code. Clicking on the “Feedback” button allows you to… send some feedback:
## Running Remotely with Supervisor

Installation is simple. Grab version  [five](https://github.com/realpython/demo/releases/tag/v5)  from the repo (if you don’t already have it). Then SSH into your remote server and run:

`$ sudo apt-get install supervisor` 

We then need to tell Supervisor about our Celery workers by adding configuration files to the “/etc/supervisor/conf.d/” directory on the remote server. In our case, we need two such configuration files - one for the Celery worker and one for the Celery scheduler.

Locally, create a folder called “supervisor” in the project root. Then add the following files…

**Celery Worker:  _demo_celery.conf_**
**Celery Scheduler/beat:  _demo_celerybeat.conf_**

Basically, these supervisor configuration files tell supervisord how to run and manage our ‘programs’ (as they are called by supervisord).

In the examples above, we have created two supervisord programs named “democelery” and “democelerybeat”.

Now just copy these files to the remote server in the “/etc/supervisor/conf.d/” directory.

We also need to create the log files that are mentioned in the above scripts on the remote server:
```
$ touch /var/log/celery/demo_worker.log
$ touch /var/log/celery/demo_beat.log
```
Finally, run the following commands to make Supervisor aware of the programs - e.g.,  `democelery`  and  `democelerybeat`:
```
$ sudo supervisorctl reread
$ sudo supervisorctl update
```
Run the following commands to stop, start, and/or check the status of the `democelery`program:
```
$ sudo supervisorctl stop democelery
$ sudo supervisorctl start democelery
$ sudo supervisorctl status democelery
```
## Final Tips

1.  _Do not pass Django model objects to Celery tasks._  To avoid cases where the model object has already changed before it is passed to a Celery task, pass the object’s primary key to Celery. You would then, of course, have to use the primary key to get the object from the database before working on it.
2.  _The default Celery scheduler creates some files to store its schedule locally._  These files would be “celerybeat-schedule.db” and “celerybeat.pid”. If you are using a version control system like Git (which you should!), it is a good idea to ignore this files and not add them to your repository since they are for running processes locally.